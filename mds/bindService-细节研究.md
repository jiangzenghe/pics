---
title: bindService 细节研究
date: 2018-03-16 20:21:40
tags: Android
---

据说在manifest.xml里配置

```
<service
    android:name=".testservice.MyService"
    android:process=":remote">
</service>
```

然后在主线程MainActivity中启动MyService----bindService（this，serviceConnection，BIND_AUTO_CREATE），
会使被启动的MyService跟启动的线程不在同一线程中，
但是实际测试中不管是在MyService的onCreate或onBind中打印当前线程，还是在ServiceConnnetion的回调中打印当前线程，都还是显示是主线程。

<!--more-->

<font color=red>注：更改下，这里应该是两个不同进程的主线程，所以打印的时候打印的都是main，其实一个是第一个进程的main，另外一个是另外一个进程的main。</font>

需要再分析bindService的具体细节（Binder机制），然后对比来看一下这个结论是怎么一回事。

另外，在
https://www.cnblogs.com/smyhvae/p/4070518.html这篇文章的分析中称

**”------------中间为原文内容 红色为我加的注释------------------**
###  

> 点击start_service，服务启动。既然已经将Service的android:process属性指定成:remote，
此时Service和Activity不在同一个线程内，那么即使在Service的onStartCommand()方法中执行耗时操作而不重新开启子线程，程序也不会阻塞。
但是，如果点击bind_service按钮绑定服务，程序会崩溃的。这是因为，目前MyService已经是一个远程Service了，Activity和Service运行在两个不同的进程当中，这时就不能再使用传统的建立关联的方式，程序也就崩溃了。
<font color=red>（这个地方我实际测试了一下，并不会像文中说的这样崩溃，而是正常的运行）</font>

> 现在我们总结一下：
第四段中使用的是传统的方式和Service建立关联，默认MainActivity和MyService在同一个线程内，如果将Service的android:process属性指定成:remote，此时MainActivity和MyService将在不同的线程内，<font color=blue>但是无法绑定服务。</font>
<font color=red>（可以绑定服务）</font>
本段中（第五段）使用的是IPC跨进程通信，MainActivity和MyService在不同的进程中，可以绑定远程服务。
<font color=red>（当然可以，跟Acvitity和Service在不在不同的进程中，在不在不同的线程中没有关系）</font>
###  
**” ------------中间为原文内容-----------------------------** <br>
注释：红色的注释地方我试过不会发生，但是有其他的知名博客也是跟文章观点一致，有可能是在不同的android版本下，发生了不同的情况？
<font color=red>注：文中红色部分是可以的，但是要基于aidl去操作；
另外文章的本观点也是没错的，他是基于没有使用aidl，而是只在MyService中使用一个继承了Binder实体的对象来进行bindservice，所以关联绑定的过程中自然就崩溃了。<br>***非常重要</font>
