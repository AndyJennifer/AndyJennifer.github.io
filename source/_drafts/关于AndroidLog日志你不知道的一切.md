---
title: 关于AndroidLog日志你不知道的一切
tags:
  - null
categories:
  - null
---

### 前言

* 简单介绍日志的输出快捷键
* 介绍常用的日志框架，logger，timer，Hugo
* Timer日志输出加+log日志上传

Logger地址：https://github.com/orhanobut/logger
Hugo地址： https://github.com/JakeWharton/hugo
Timer地址：https://github.com/JakeWharton/timber
日志上传:     https://github.com/tony19/logback-android
Timber博客参考文章：https://blog.csdn.net/Tomasyb/article/details/76034231
Log快捷键及需求参考文章：https://blog.csdn.net/wangshihui512/article/details/51042704
Hugo 参考文章：https://www.jianshu.com/p/88abba27a2f3
Logback: https://github.com/tony19/logback-android
美图日志框架：https://juejin.im/entry/5865f3ddac502e006129b86d  https://tech.meituan.com/2018/10/11/logan-open-source.html

需要思考的问题：

* Hugo为什么要定义gradle插件
* Hugo中使用的技术是否了解，注解，AOP，AspectJ，Gradle插件。
需要完成的功能：
* 打印漂亮的日志（程序员调试查看用）
* 日志信息上传功能(用于分析线上bug)
* 用于埋点(统计用户行为）
* 日志业务逻辑分析（用于分析线上的bug)

### 最后

站在巨人的肩膀上，才能看的更远~
