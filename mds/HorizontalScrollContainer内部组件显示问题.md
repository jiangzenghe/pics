---
title: HorizontalScrollContainer内部组件显示问题
date: 2018-04-28 17:55:54
tags: Android
---

自己实现了一个HorizontalScroll的view，嵌套在google的HorizontalScrollView内部，其实是一个水平方向的linearlayout，然后每次add一个组件。

开始每次add的组件只是一个TextView，没有什么问题，可以点击切换，水平滑动。后来需求升级，需要在TextView下面添加一个被选中的底部bar，所以每次add的组件就理所当然升级成为一个垂直方向的linearlayout，然后里面包含原有的TextView，和一个新加的用以展示底部bar的View。

完事以后底部bar的View死活展示不出来。经过半天多的调试，发现是使用动态代码生成的时候，垂直方向的父linearlayout添加TextView和BarView的时候，要严格注意被添加view的LayoutParams。

需要添加的时候这样才可以：

<!--more-->

**父LinearLayout**
LinearLayout.LayoutParams linearParentParams = new LinearLayout.LayoutParams(
        LayoutParams.<font color=red>WRAP_CONTENT</font>, LayoutParams.<font color=red>WRAP_CONTENT</font>);


**原有的TextView**
LinearLayout.LayoutParams params = new LinearLayout.LayoutParams(
                    LayoutParams.<font color=red>WRAP_CONTENT</font>, LinearLayout.LayoutParams.<font color=red>WRAP_CONTENT</font>);
    params.setMargins(24, 0, 24, 0);
linearLayout.addView(textView, params);

**新加的BarView**
（使用TextView来玩，用别的View即使属性设置成如下的参数，也显示不出来,注意<font color=red>WRAP_CONTENT</font>）

```java
TextView bottom = new TextView(this.getContext());
LinearLayout.LayoutParams bottomParams = new LinearLayout.LayoutParams(
                    LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    bottomParams.setMargins(24, 0, 24, 0);
    bottom.setLayoutParams(bottomParams);
    //用textview玩，必须要设text文本值了            
    bottom.setText("agef");
linearLayout.addView(bottom);
```

或者用ImageView来，弄一个bar的样式图片，就可以了。
反正用View是不行的，不管属性怎么设，都不会显示出来的。

### 问题的根节：
其实并不是使用哪种view不行 而是（<font color=red>使用View不要用WRAP_CONTENT，因为里面没内容；使用textView.getWidth()也没显示出来是因为TextView还没有被加进去，width为0</font>）