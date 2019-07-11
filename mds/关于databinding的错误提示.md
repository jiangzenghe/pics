---
title: 关于databinding的错误提示
date: 2018-07-05 21:30:59
tags: Android
---

今天打包的时候提示：

错误: 程序包com.btb.databinding不存在

google搜索：有这样的回答

###  
/********别人的问题
AAAViewModel这个文件在aaa包下，如果你移动了AAAViewModel这个文件到bbb包下，问题就来了，其他引用到这个文件的java类里都会自动将aaa修改到bbb，但是databinding这个地方不会修改，还是保持着com.aaa.AAAViewModel，这个时候它已经标红了，但除非进入这个xml中看，不然你根本发现不了这个问题。所以你只要将它改为

<!--more-->

```
<data>
    <variable
        name="viewModel"
        type="com.bbb.AAAViewModel" />
</data>
```

就万事大吉了，然而当我发现的时候，几个小时已经过去。。。



所以移动文件的时候一定小心，特别是使用了databinding的xml中，因为目前Android Studio还不能帮你自动把引用文件的包名改为最新的。

别人的问题***********/
###  

但是这个不是我的问题

另外自己也碰到过这个问题，原因是xml文件写错的，根本不对，没有按照databinding的约定来写。但是我这里也是符合标准的。

后来就只能怀疑代码问题了，但是我在正常的时候跟现在异常的时候的代码基本一样，就是调整了一点点。
但是后来发现确实就是 因为这一点点，导致了 这种离奇的databinding的报错，但是改过的代码却没有任何的报错，非常的误导人。

### <font color=red>具体原因是这样：</font>
我在Pub的static方法中调用了MeFragment的static方法
obtainCountryID，但是改动过程中把MeFragment的这个方法移到别的类中，把原有的MeFragment的类中的方法删掉了，（设计上其实不推荐这样）

但是问题是as没给报代码错误，反而一直在报databinding的错误。
非常误导别人的。

吃饭之前还在嘲笑xt提的这个问题，结果回来这个问题就直接打脸，找了一个小时。。
xt的问题如下：仔细对下吧，其实细节的提示差不多。其实都是编译失败，只不过as没有给红色的错误提示，没有给错误代码的标注展示。

![as 错误提示](https://app.yinxiang.com/shard/s59/res/e631cc3c-016c-4b30-b241-3960e702561f/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_15307938708763.png)
