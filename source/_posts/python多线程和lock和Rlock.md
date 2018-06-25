---
title: python多线程和lock和Rlock
date: 2018-06-14 17:02:12
tags:
---

在threading模块中，定义两种类型的琐：threading.Lock和threading.RLock。

RLock允许在同一线程中被多次acquire。而Lock却不允许这种情况。注意：如果使用RLock，那么acquire和release必须成对出现，即调用了n次acquire，必须调用n次的release才能真正释放所占用的琐。

rlock只能在同一个线程acquire和release

lock可以在不同的线程acquire和release

rlock的作用: 在递归程序中，可能会需要重复的acuqire,release，这时候，就必须要使用rlock。