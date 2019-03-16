---
title: Jvm系列之类加载器
tags:
- Jvm
categories:
- Jvm
---

Jvm系列之了类加载器（六）

### 类加载器分类

#### 从java虚拟机的角度
- 启动类加载器（Bootstrap ClassLoader) 虚拟机的一部分
- 其他类加载器，由Java语言实现，独立于虚拟机的外部，并且全部继承抽象类java.lang.ClassLoader。

#### 从开发人员的角度来看
- 启动类加载器
1. 作用：负责将存放在<JAVA_HOME>\lib 目录中的，或者被-Xbootclasspath参数所指定的路径中的，且是能被虚拟机识别的（按照文件名进行识别）的类库加载到虚拟机内存中，
2. 特点：启动类加载器，无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委托给引导类加载器，那直接使用null代替即可。

- 扩展类加载器
作用：负责将存放在<JAVA_HOME>\lib\ext 目录中的或者java.ext.dirs系统变量中路径中所有的类库

- 应用程序加载器
作用：负责加载用户类路径（ClassPath)所指定的类库

### 双亲委派模型
工作过程：当一个类加载器收到了类加载的请求，它首先不会自己尝试去加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此。因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有父类加载器反馈自己无法完成这个加载请求时，子类加载器才会去尝试去加载。
好处：确定某个对象的唯一性，是需要类加载器确定的，相同类由不同的类加载器加载，是被判断为不同对象的。


### 双亲委派模型的破坏

#### 第一次破坏
双亲委派模型是JDK1.2之后才引入的。而类加载及抽象类ClassLoader在JDK1.0时代就有了。之前的操作是重写loadClass方法。为了向前兼容提供了findClass方法。

```
 protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);//重写findClass方法

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

```
#### 第二次破坏
双亲委派模型能很好的解决各个类加载器的基础类的统一问题。但是会出现基础类又要回调用户的代码。所以为了解决这个问题，添加了线程上下文类加载器。这个类加载器可以通过Java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时还未设置，它将从父线程中继承一个，如果在应用程序的全局范围内没有设置过的话，那这个类加载器就是应用程序类加载器。也就是说现在父类加载器可以请求子类加载器完成类的加载动作了。

```
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name.toCharArray();

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        //注意这里的判断，从父类获取
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
```

```
 public static ClassLoader getSystemClassLoader() {
        initSystemClassLoader();
        if (scl == null) {
            return null;
        }
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkClassLoaderPermission(scl, Reflection.getCallerClass());
        }
        return scl;
    }

    private static synchronized void initSystemClassLoader() {
        if (!sclSet) {
            if (scl != null)
                throw new IllegalStateException("recursive invocation");
            sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
            if (l != null) {
                Throwable oops = null;
                scl = l.getClassLoader();
                try {
                    scl = AccessController.doPrivileged(
                        new SystemClassLoaderAction(scl));
                } catch (PrivilegedActionException pae) {
                    oops = pae.getCause();
                    if (oops instanceof InvocationTargetException) {
                        oops = oops.getCause();
                    }
                }
                if (oops != null) {
                    if (oops instanceof Error) {
                        throw (Error) oops;
                    } else {
                        // wrap the exception
                        throw new Error(oops);
                    }
                }
            }
            sclSet = true;
        }
    }
```


```
   public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }

        Thread.currentThread().setContextClassLoader(this.loader);
        String var2 = System.getProperty("java.security.manager");
        if (var2 != null) {
            SecurityManager var3 = null;
            if (!"".equals(var2) && !"default".equals(var2)) {
                try {
                    var3 = (SecurityManager)this.loader.loadClass(var2).newInstance();
                } catch (IllegalAccessException var5) {
                } catch (InstantiationException var6) {
                } catch (ClassNotFoundException var7) {
                } catch (ClassCastException var8) {
                }
            } else {
                var3 = new SecurityManager();
            }

            if (var3 == null) {
                throw new InternalError("Could not create SecurityManager: " + var2);
            }

            System.setSecurityManager(var3);
        }

    }

```



#### 第三次破坏
osgi

#### 自定义类加载器
```
    protected ClassLoader() {
        this(checkCreateClassLoader(), getSystemClassLoader());
    }
```
### 参考
[IBM-深入探讨 Java 类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)