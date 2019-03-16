---
title: OkHttp源码解析
tags:
- OkHttp
categories:
- 源码分析
---



### OkHttp中的Builder类
```
    final Dispatcher dispatcher;  //分发器
    final Proxy proxy;  //代理
    final List<Protocol> protocols; //协议
    final List<ConnectionSpec> connectionSpecs; //传输层版本和连接协议
    final List<Interceptor> interceptors; //拦截器
    final List<Interceptor> networkInterceptors; //网络拦截器
    final ProxySelector proxySelector; //代理选择
    final CookieJar cookieJar; //cookie
    final Cache cache; //缓存
    final InternalCache internalCache;  //内部缓存
    final SocketFactory socketFactory;  //socket 工厂
    final SSLSocketFactory sslSocketFactory; //安全套接层socket 工厂，用于HTTPS
    final CertificateChainCleaner certificateChainCleaner; // 验证确认响应证书 适用 HTTPS 请求连接的主机名。
    final HostnameVerifier hostnameVerifier;    //  主机名字确认
    final CertificatePinner certificatePinner;  //  证书链
    final Authenticator proxyAuthenticator;     //代理身份验证
    final Authenticator authenticator;      // 本地身份验证
    final ConnectionPool connectionPool;    //连接池,复用连接
    final Dns dns;  //域名
    final boolean followSslRedirects;  //安全套接层重定向
    final boolean followRedirects;  //本地重定向
    final boolean retryOnConnectionFailure; //重试连接失败
    final int connectTimeout;    //连接超时
    final int readTimeout; //read 超时
    final int writeTimeout; //write 超时
```

### 发起请求

#### 发起同步请求
```
String run(String url) throws IOException {
  Request request = new Request.Builder()
      .url(url)
      .build();

  Response response = client.newCall(request).execute();
  return response.body().string();
}
```

```
@Override public Response execute() throws IOException {
	//第一步，判断是否被执行
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      //第二步
      client.dispatcher().executed(this);
      //第三步
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      //第四步
      client.dispatcher().finished(this);
    }
  }
```
- 第一步 检查这个 call是否已经被执行了，每个 call 只能被执行一次，如果想要一个完全一样的 call，可以利用 all#clone 方法进行克隆。
- 第二步 利用 client.dispatcher().executed(this) 来进行实际执行，dispatcher 是刚才看到的 OkHttpClient.Builder 的成员之一，它的文档说自己是异步 HTTP请求的执行策略，现在看来，同步请求它也有掺和。
- 第三步 调用 getResponseWithInterceptorChain() 函数获取 HTTP 返回结果，从函数名可以看出，这一步还会进行一系列“拦截”操作。
- 第四步 最后还要通知 dispatcher 自己已经执行完毕。
dispatcher 这里我们不过度关注，在同步执行的流程中，涉及到 dispatcher 的内容只不过是告知它我们的执行状态，比如开始执行了（调用 executed），比如执行完毕了（调用 finished），在异步执行流程中它会有更多的参与。
真正发出网络请求，解析返回结果的，还是 getResponseWithInterceptorChain：

### Dispatcher

```
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

#### 发起异步请求

```
synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

### 拦截器责任链
```

  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```
- **interceptors**：在配置 OkHttpClient 时设置的拦截器；
- **RetryAndFollowUpInterceptor**：负责失败重试以及重定向；
- **BridgeInterceptor**：负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应
- **CacheInterceptor**：负责读取缓存直接返回、更新缓存；
- **ConnectInterceptor**：负责和服务器建立连接的 ；
- **networkInterceptors**： 配置 OkHttpClient网络拦截器；
- **CallServerInterceptor**： 负责向服务器发送请求数据、从服务器读取响应数据。
- 在return chain.proceed(originalRequest);中开启链式调用：

### RealInterceptorChain

```
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    //判断是否是同一端口号与主机地址
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // 已经被执行的请求，不能再被执行一次
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // 调用下一个拦截器
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```


```
 RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```
- 实例化下一个拦截器对应的RealIterceptorChain对象，这个对象会在传递给当前的拦截器
- 得到当前的拦截器：interceptors是存放拦截器的ArryList
- 调用当前拦截器的intercept()方法，并将下一个拦截器的RealIterceptorChain对象传递下去,除了在client中自己设置的interceptor,第一个调用的就是**retryAndFollowUpInterceptor**。


### RetryAndFollowUpInterceptor

```
/**
   * How many redirects and auth challenges should we attempt? Chrome follows 21 redirects; Firefox,
   * curl, and wget follow 20; Safari follows 16; and HTTP/1.0 recommends 5.
   */
  private static final int MAX_FOLLOW_UPS = 20;
```

### BridgeInterceptor
构建用户的请求与响应体

### CacheInterceptor
判断是否需要读取缓存数据
 
### ConnectInterceptor
构建连接通道，也就是Socket通道

### CallServerInterceptor
发起请求拦截器