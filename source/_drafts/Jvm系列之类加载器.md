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

