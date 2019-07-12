---
title: AppTheme的继承与冲突问题
date: 2018-08-29 11:42:49
tags: Android
---

各种主题的切换会涉及到颜色值的抽取，本来的color需要转换为attr，这里记录下做主题切换中碰到的问题。

### 场景还原：
工程目录结构：
1. app工程 持有自己的styles 依赖commLib
2. commLib工程 持有自己的styles
3. klinelib工程 依赖commLib

<!--more-->

变化之前，都是好的，klinelib里的工程，颜色值正常，circle与rectangle颜色正常，没有透明度等，然后
需要做黑白主题切换，原来的主题是黑色的，现在加了一套白色的主题，于是把原有的color都替换成style里的item，用attr来寻找，然后发现klinelib里的底色变成透明的了，遮不住下面的背景。

调试了一下午，发现在取color值的时候，原来是getResource.getColor,现在需要通过这种方式获取：

> context.getTheme().resolveAttribute(id, typedValue, true);

id就是attr的对应id，从theme中获取。

开始怀疑context不同，结论在发现真相之后是相同的，（最开始没怀疑这个地方，总是纳闷为什么到了klinelib里面颜色值就变了，感觉是透明度变了，其实颜色根本没设上，一直都是透明度100%的无颜色值），最开始一直对比app工程里正常的值，发现有些地方设置用了setBackgroundResource，这里明显是不对的，因为attr对应其实就是一个color值，只要直接拿到值当color用就可以了。

最后终于怀疑到了正点上，打算调试颜色的值，看看是不是谁偷偷给改了透明度，结果调试一看，值都是0.。
然后就怀疑获取颜色的地方，原来在app工程里，有个Pub的类，在里面获取颜色值，然后把Pub复制，改名为CommonPub，挪到了CommLib中，也是同样的代码去获取值，但是问题在于，app和commLib都有自己的style，定义的styletheme名称也是一样的，但是commLib使用的是全部的attr的类，app里只有一部分，然后Pub和CommonPub都是app的上下文，app又没有从commLib继承主题Theme，结果导致取kline需要的那个颜色总是取不出来，（不过这里没有崩溃也很令人诧异），本来theme就不该这么玩的，中间也引起了几次崩溃的问题，当时应该早能联想到应该跟这个地方相关的，浪费了很多时间。

正确的做法应该是CommLib定义全部，其他的所有从CommLib继承，这样才能无问题的切换theme。