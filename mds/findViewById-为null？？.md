---
title: findViewById 为null？？
date: 2018-08-15 22:32:02
tags: Android
---

再自定义View中嵌套自定义View，然后用findViewById寻找子元素-自定义View，返回null

搜索了以下，有个帖子比较全，讲的比较细，但是没有描述到我的情况，这个帖子如下：
https://blog.csdn.net/ITermeng/article/details/52034186
描述了各种情况。

<!--more-->

我的问题是跟DataBinding有关，其实也就是跟clean有关的，我在父自定义View的layout布局中使用了跟其他布局相同的自定义类DepthFixNumberView和id，结果狗日的编译工具认为是一个。。拉风道长有在自己的博客中涉及此处，其实这真的是编译工具的坑啊。。

### <font color=red>拉风的原文</font>

错误：findViewById返回Null，报nullpointer错误

网上搜了下，拾人牙慧，总结原因，一般为4种：



**1.在另一个view的元素应该用baseView.findViewById()来拿**
findViewById()是要指定view的，如果在该view下找不到，自然报null。平时注意养成写view.findViewById()的习惯就不容易错了。

 

**2.findViewById在setContentView(R.layout.main);之前.**
即在setContentView调用之前，调用了findViewById去找main布局中的界面元素lv_contactbook，那么所得到的lv一定是null。正确的做法是将上面代码中加粗的哪一行，挪至setContentView方法调用之后即可。



**3.clean一下工程，让ID重新生成**
这种情况是调用LayoutInflater.inflate将布局xml规定的内容转化为相应的对象。比如有rowview.xml布局文件(比如在自定义Adapter的时候，用作ListView中的一行的内容的布局)，假定在自定的Adapter的getView方法中有类似如下的代码：

```java
View rowview = (View)inflater.inflate(R.layout.rowview, parent, false);
TextView tv_contact_id =(TextView)rowview.findViewById(R.id.tv_contact_id);
TextView tv_contactname =(TextView)rowview.findViewById(R.id.tv_contactname);
```

有时候居然也会发现rowview非空，但tv_contact_id和tv_contactname都是null！仔细看代码，怎么也看不出错误来。到底是什么原因造成的呢？答案是Eclipse造成的，要解决这个问题，需要这个项目clean一次(Project菜单 -> Clean子菜单)，这样就OK了。

**4.对于自定义view，可能是构造方法不对**
```java
public MyView(Context context,AttributeSet attr) {
        super(context); //这里调用的构造方法不对。应该调用super(context,attr);
}
```

<font color=red>我的情况算是第3种情况吧。</font>
