---
title: java-面试准备
abbrlink: 43363
date: 2019-02-24 21:16:41
tags:
---

# Exception和Error的区别， 运行时异常和一般异常的区别

Exception和Error都继承了-Throwable类，在Java中只有Throwable类型的实例才可以被抛出或者捕获，是异常处理机制的基本组成类型。

Exception和Error体现了Java平台设计者对不同异常情况的分类。Exception是程序正常运行中，可以预料的意外情况，可能并且应该被捕获，进行相应处理。

Error是指正常情况下，不大可能出现的情况，绝大部分的Error都会导致程序（比如JVM自身）处于非正常的，不可恢复状态。比如OutOfMemoryError之类，都是Error的子类。

Exception分为可检查异常和不检查异常，可检查异常在源代码里必须显式的进行捕获处理，这是编译器检查的一部分。

Error属于Throwable，而不是Exception。