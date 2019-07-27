---
title: RxJava网络请求封装
date: 2019-02-23 22:14:46
tags: 
- Rxjava
- Retrofit
---

{% asset_img 漫长的婚约.jpg 漫长的婚约 %}

最近平时开发中，用到了RxJava进行网络请求的封装，其中遇到了一个问题与大家分享一下。在无网络的情况下，我的程序直接抛出了不能连接某某地址。就以登录请求为例，具体代码如下：

```java
 public void login(String userName, String password) {
        mModel.login(userName,passWord).subscribe(new Consumer<UserLoginInfo>() {
            @Override
            public void accept(UserLoginInfo loginInfo) throws Exception {
              //省略逻辑代码
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
               Toast.makeText(mContext, throwable.getMessage(), Toast.LENGTH_SHORT).show();
            }
        });
    }
```

上述代码中，我是直接将错误信息打印出来的。当网络情况下。程序会直接抛出**java.net.ConnectException: Failed to connect to /192.168.1.107:8080**，为了良好的用户体验，不应该将网络异常提示出来。用户看后也是**一脸懵逼**，所以必须将此异常进行处理。提示正常的信息。如**网络异常，请检查网络重试**。那下面我们就开始封装吧。

#### 处理异常信息

既然程序出现异常的时候会走。相应的Consumer< Throwable >接口，那么我们直接自定义一个类。实现该接口。并对错误进行分类处理。具体代码如下：

```java
public abstract class ConsumerError<T extends Throwable> implements Consumer<T> {
    @Override
    public void accept(T t) throws Exception {
        String errorMessage = "";
        int errorCode = 0;
        if (t instanceof SocketException) {//请求异常
            errorMessage = "网络异常，请检查网络重试";
        } else if (t instanceof UnknownHostException) {//网络异常
            errorMessage = "请求失败，请稍后重试...";
        } else if (t instanceof SocketTimeoutException) {//请求超时
            errorMessage = "请求超时";
        } else if (t instanceof ServerException) {//服务器返回异常
            errorMessage = t.getMessage();
            errorCode = ((ServerException) t).getErrorCode();
        }
        onError(errorCode, errorMessage);
    }

    public abstract void onError(int errorCode, String message);
}
```

上述代码我们可以发现。我们创建了一个抽象类。并且提供了一个**onError**抽象方法（参数1：错误码，参数2：错误信息）。当出现错误的时候，我们只需要创建匿名内部类，并**实现onError方法**就行了。细心的小伙伴可以看见这里出现了一个**ServerException**，这是什么鬼。那接下来，慢慢说。

#### 自定义服务器异常

在网络请求中，服务器会返回一些错误。当我们收到服务器返回的这些错误信息的时候。可能会对错误信息进行一些相关的处理。说到服务器异常。那就必须要提到服务器返回的数据格式与网络请求。具体实现如下：

#### 服务器返回数据格式

```java
public class Result<T> implements Serializable {
    public boolean result;
    public int code;
    public String message;
    public T data;
}
```

#### 具体的网络请求

```java
public Observable<UserLoginInfo> login(String userName, String password) {
 return Api.getDefault().login(userName,password).compose(RxHelper<UserLoginInfo>handleResult());
}
```

上述代码中，请求地址与参数都可以不用管。直接查看compose()函数中**RxHelper<`Boolean`>handleResult())**。这里有可能有些小伙伴不知道compose操作符。[传送门](https://www.jianshu.com/p/3d0bd54834b0)

##### RxHelper具体实现

```java
public class RxHelper {

      //处理请求结果 针对 有code message data 的json
      //对请求状态吗进行分析，如果成功获取result 中的data
      public static <T> ObservableTransformer<Result<T>, T> handleResult() {
        return upstream -> upstream.flatMap(result -> {
            if (result.code == HttpErrorCode.HTTP_NO_ERROR) {
                return createSuccessData(result.data);
            } else {
                return Observable.error(new ServerException(result.code, result.message));①
            }
        }).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread());
    }

    //处理没有data的Result
    public static ObservableTransformer<Result, Result> handleOnlyResult() {
        return upstream -> upstream.flatMap(result -> {
            if (result.code == HttpErrorCode.HTTP_NO_ERROR) {
                return createSuccessData(result);
            } else {
                return Observable.error(new ServerException(result.code, result.message));②
            }
        }).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread());
    }


    //创建成功返回的数据
    private static <T> Observable<T> createSuccessData(final T t) {
        return Observable.create(new ObservableOnSubscribe<T>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<T> emitter) throws Exception {
                try {
                    emitter.onNext(t);
                    emitter.onComplete();
                } catch (Exception ex) {
                    emitter.onError(ex);
                }
            }
        });
    }
}

```

RxHelper主要用于网络请求帮助类处理。对请求结果进行了判断，同时对请求与响应线程进行了处理，看①②处，发现如果当前数据返回状态码不是成功的话，程序就会直接抛出**带有当前的状态码与错误信息的Error**。程序抛出Error后，自然会走我们观察者中的Error方法（**也就是我们自定义的ConsumerError中的error方法**）。

这样到此。整个异常处理流程就完全清楚了。当然在服务器返回的错误中。你可以根据服务器返回的错误。定义相应的错误处理。

#### 错误信息封装

```java
public class HttpErrorCode {

    /**
     * 请求的服务不存在
     */
    public static final int ERROR_404 = 404;

    /**
     * 系统内部错误
     */
    public static final int ERROR_500 = 500;

    /**
     * 程序内部异常
     */
    public static final int ERROR_998 = 998;

    /**
     * 未知错误
     */
    public static final int ERROR_999 = 999;
    /**
     * 请求成功
     */
    public static final int HTTP_NO_ERROR = 200;

    /**
     * 自定义异常
     */
    public static final int USER_NOT_EXIT = 100000;
}

```

#### 封装后具体代码

```java
public void login(String userName, String password) {
        mModel.login(userName,passWord).subscribe(new Consumer<UserLoginInfo>() {
            @Override
            public void accept(UserLoginInfo loginInfo) throws Exception {
              //省略逻辑代码
            }
        },new ConsumerError<Throwable>() {
                    @Override
                    public void onError(int errorCode, String message) {
                      if(errorCode== HttpErrorCode.USER_NOT_EXIT){//根据具体错误，处理相应逻辑。

                      }
                    }
                });
    }
```

这样我们的请求封装就完成啦。我们只要在ConsumerError中的onError根据不同的错误码，来处理相应的业务逻辑了。是不是很方便？

最后，附上我写的一个基于Kotlin 仿开眼的项目[SimpleEyes](https://github.com/AndyJennifer/SimpleEyes)(ps: 其实在我之前，已经有很多小朋友开始仿这款应用了，但是我觉得要做就做好。所以我的项目和其他的人应该不同，不仅仅是简单的一个应用。但是，但是。但是。重要的话说三遍。还在开发阶段，不要打我)，欢迎大家follow和start
