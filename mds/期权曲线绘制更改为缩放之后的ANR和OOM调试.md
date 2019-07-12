---
title: 期权曲线绘制更改为缩放之后的ANR和OOM调试
date: 2018-12-12 11:44:11
tags: Android
---

一个很恶心的问题，原来的期权曲线展示只做了四个维度，然后后来要实现缩放的效果，在添加缩放手势的同时，对原有的代码做了重构，代码发生了很大变化，但是修改完之后，发现总是会在10几分钟的时候发生OOM或者出现无反应的ANR，虽然原来的期权曲线性能也不太好，但是不至于在十几分钟就会发生OOM崩溃什么的，为此踏上了一条寻找OOM原因的不归之路。

<!--more-->

代码review没有发现什么特别的地方。万般无奈只好研究了下Android Profiler来分析CPU和Memory，真的是巨艰难。里面的细节还是蛮多的，搜了几篇文章感觉还是很不错的，结果最好发现其实都是官方文档的翻译。。。看来找资料第一选择还是要先去官档查看啊。

### 归纳总结下几个关键点。
Profiler的Memory分析图功能做的还是很细致完善的，就是占用内存太大，经常出现heap dump卡死的情况。
Java、Code、Native、Others等的细分和沿时间线绘制Memory占用就不赘述了，主要讲几点不容易get到的。

* Memory右侧三个按钮，官方解释，第一个GC模拟，第二个Java Heap dump，第三个录制 
    &nbsp;&nbsp;&nbsp;&nbsp;第一个就是强制联调机器GC，毕竟容易理解。
    &nbsp;&nbsp;&nbsp;&nbsp;第二个不容易理解，我的理解是执行Java Heap Dump，其实就是点击之后，profiler会监视内存,<font color=red>大概一小段时间之后，就会把之后的那个瞬间的snapshot给记录</font>，也就是会记录gc之后的内存中还存在的那些heap相关数据，可能是刚才的细分中的几个区交叉的数据集，java+Others等的数据。
    &nbsp;&nbsp;&nbsp;&nbsp;第三个录制，我理解是搜集这段时间allocate和dellocate的数据，上限65535（上限收集这么多），所以经常dump之后的数据allocate非常多，但是录制里面最大也就65535个。

* 录制或dump之后（dump会额外消耗很多调试机器内存，所以dump的时候监控的内存会突然多出来一截），下方会展示分析的数据，里面会有Shallow Size和Retained Size，Shallow是直接占用内存，Retained是引用涉及的内存。

* 官方文档中说Shallow和Retained Size是以Byte为单位的，但是我发现Retained Size会超出总的内存非常多（Java+Others+Code+Native等等等和），我一度认为官档是错的，可能是bit为单位的，因为实在超出太多了，比如Retained是1个多G，总监视内存才200m，但是后来想想，可能Retained内部有重复的，应该是看shallow里面的内存和，不超过总监是内存就对的，单位应该确实是以Byte为单位的。

### 然后讲问题分析的具体流程和细节。（讲一讲怎么发现最后的代码问题的）
思路是有两个分支，一个是之前的notScale分支，不会轻易出现OOM；另外一个是scale分支，容易OOM，思路即选取几个关键节点，比如初始进入、1m和4m（一般scale分支在4m左右就会卡死，或者出现OOM，OOM和卡死的原因大致一致是因为GC来不及回收，一直GC耗尽CPU，释放完不成导致Memory也告罄），在这几个时间点dump heap，然后查看捕获的数据，对比相同时间点scale和notScale的after dump的内存构成。
刚进入的时候，scale和notScale的内存占用差不多，1m节点scale和notScale的java内存已经有差距但不明显，3m节点两者但java内存差距已经挺大了要到10m左右的差距了。
1.not-scale-start图
![not-scale-start图](https://github.com/jiangzenghe/pics/blob/master/tech/20181212114411-1.png?raw=true)

2.scale-start图
![scale-start图](https://github.com/jiangzenghe/pics/blob/master/tech/20181212114411-2.png?raw=true)

1m和3m的图就不上传了，对比发现not-scale分支中，能一直稳定的gc，然后每次gc之后的内存占用在15分钟之内一般维持在100m左右，但是scale分支的native和java内存增长非常迅速，一般在4分左右已经分别增长20m和10m，总体增长进三四十m。
PS：保存或捕获heap dump非常蛋疼，一般第二次dump系统就会卡死，需要性能好一点的机器啊

**scale-1min图**
![scale-1min图](https://github.com/jiangzenghe/pics/blob/master/tech/20181212114411-3.png?raw=true)

可以看到not-scale的分支不管在几分钟，一般内存占用的排序是（按shallow size排序）byte[]、FinalizerReference、long等顺序可能有变化，但是占用的大头都是这几个。（没搞明白FinalizerReference是哪来的，可能跟heap dump有关），但是scale-1min里面，占用靠前的出来一个Float和Object[]，占用了3.6M，已经很大了。
观察到这个之后，再进一步来分析，这次只调试scale分支，细致对比scale刚进入、1m和4m等时间节点，观察是不是上述Float等的占用在不断增长。

这次使用官方提供的保存文件来分析，没有截图，文件类型是hprof，可以展示各占用的细节。
文件对比中，发现Float的字节占用从1m时候的3.6m变成4m时候的12m，3分钟的时间增长了整整10m，基本上可以确定是Float在一直吃掉内存，然而又没有释放，导致了之后的OOM和CPU耗尽。

那么最后的问题是代码的什么地方用到了Float，而且是什么原因导致了Float无法被释放呢？
这时候就要借用Profiler提供的Instance和Callback来寻找了，其实在代码里全局搜索Float也可以笨办法找到，不过有点low的。<br>看截图
![checkout-prof](https://github.com/jiangzenghe/pics/blob/master/tech/20181212114411-4.png?raw=true)

题外话：搞清楚Total Count、Heap Count、Sizeof、Shallow Size、Retained Size、Depth、Dominating Size等概念

可以看到Instance点击之后，下面的Reference Tree展示的调用堆栈，OptionChartRender里面有涉及，进到代码页面，搜索Float，找到罪魁祸首：
代码

```java
     private List<Float> temp = new ArrayList<>();
    @Override
    protected void generateFilledPath(ILineDataSet dataSet, int startIndex, int endIndex, Path outputPath) {

        final float fillMin = dataSet.getFillFormatter().getFillLinePosition(dataSet, mChart);
        final float phaseY = mAnimator.getPhaseY();
        final boolean isDrawSteppedEnabled = dataSet.getMode() == LineDataSet.Mode.STEPPED;

        final Path filled = outputPath;
        filled.reset();

        final Entry entry = dataSet.getEntryForIndex(startIndex);
        //jiang
        if(entry == null) {
            return;
        }

        filled.moveTo(entry.getX(), fillMin);
        filled.lineTo(entry.getX(), entry.getY() * phaseY);
        temp.add(entry.getX());

        // create a new path
        Entry currentEntry = null;
        Entry previousEntry = null;
        for (int x = startIndex + 1; x <= endIndex; x++) {

            currentEntry = dataSet.getEntryForIndex(x);
            if(currentEntry == null) {
                if (previousEntry != null) {//改currentEntry为previousEntry
                    filled.lineTo(previousEntry.getX(), fillMin);
                }
                filled.close();
                return;
            }
            //jiang 20181211
            if (currentEntry.getY() < 0) {
                break;
            }

            if (isDrawSteppedEnabled && previousEntry != null) {
                filled.lineTo(currentEntry.getX(), previousEntry.getY() * phaseY);
                temp.add(currentEntry.getX());
            }

            filled.lineTo(currentEntry.getX(), currentEntry.getY() * phaseY);
            temp.add(currentEntry.getX());

            previousEntry = currentEntry;
        }

        if (currentEntry != null) {
            filled.lineTo(currentEntry.getX(), fillMin);
            temp.add(currentEntry.getX());
        }

        filled.close();
    }
```

<font color=red>
private List<Float> temp = new ArrayList<>();<br>
temp.add(currentEntry.getX());
</font>

注意看标红地方，generateFilledPath方法是为了绘制遮罩层而重写的父类方法，当时不知道为什么添加了temp，类型为ArrayList<Float>，然后review代码发现这个东西没有任何用处，可能是当时为了debug加入的代码，事后没有清理。
generateFilledPath方法会一直调用，导致temp占用内存飙升，然后因为一直存在引用，也无法被回收，导致几分钟之后内存占用就被耗尽！

找了好几天，就是这么个蛋疼的结果。。其实类似低级失误或者粗心大意导致的此类问题已经非常多了，真的像自己commit代码的备注一样，要好好审视一下自己了。

最后还是要再确认下，为什么Float为占用那么多呢?
float是4字节，然后Float是12字节，不算很夸张，但是我的代码里添加了4次。。然后每0.5s就要调很多次。。
看来作为float的对象类型，不知道多少个的list数组（看内存分析文件，得有105万个Float。。。
more instances 1050928 of 1051028 remains，也就是list的长度到一百万了，不崩才怪），Float类型的已经可以影响到内存了，本来还觉得应该不至于，看来还是要保持精细管理的习惯，不要因为机器性能和硬件的提升，就写一些很low的代码作死。。
PS：1，000，000*12byte = 12m，嗯真的是作死啊

### 总结
1.不要再粗心大意！不要再粗心大意！不要再粗心大意！
2.学习Profiler的使用，对内存等的细节更加确认与巩固
