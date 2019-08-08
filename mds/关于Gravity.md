---
title: 关于Gravity
date: 2018-08-08 13:55:27
tags: Android
---

一直对layout_gravity和gravity的使用迷迷糊糊的，有时候看文档，感觉明白了意思，但是使用的时候又总是忘记，非常的蛋疼，这次下决心要弄清楚一下这两个layout标签的真正区别了。

### 官方说明
layout_gravity
Gravity specifies how a component should be placed in its group of cells.

gravity
Specifies how to align the text by the view's x- and/or y-axis when the text is smaller than the view.

总而言之，layout_gravity是在本view在父窗口中的布局描述，gravity指的是本窗口中的元素布局描述
另外，layout_gravity和gravity常见使用在LinearLayout中，RelativeLayout一般不会使用，因为实际已经跟自身的特点属性冲突了，使用中需要注意，一会儿会从源码角度分析下。

<!--more-->
### 源码分析 准备
分析gravity之前，先确认下xml的layout布局文件中的各种属性attrs，比如layout_width等是如何映射入View的，layout是如何变成View的
调用的基本堆栈如下：
1. oncreate调用setContentView
2. setContentView调用inflate
3. inflate调用rInflate(parser, root, inflaterContext, attrs, false);
注意第三点中的rInfalte是一个递归调用，其中会调用到onCreateView或createView，会调用newInstance

经过实验，View（比如TextView、LinearLayout等这里列举一个纯View一个ViewGroup）的gravity一般对应自身的mGravity属性，但是layout_gravity一般对应父容器的***Layout.LayoutParams里面的gravity属性
比如ViewGroup.LayoutParams、LinearLayout.LayoutParams等,应该跟onMeasure中的MeasureSpec的知识关联。

有两个已知的结论

一：

    当自身的layout_gravity和父容器对自己默认的gravity冲突时，优先选择自身的layout_gravity； 
二： 

    当android:orientation=”horizontal”时 
    自身的:layout_gravity属性有效取值为: 
    right,，left，center_horizontal有效； 
    子组件的layout_gravity有效取值为： 
    top，bottom，center_vertical 

    当android:orientation=”vertical”时 
    自身的layout_gravity属性有效取值为： 
    top，buttom，center_vertical 
    子组件的layout_gravity有效取值为： 
    right,，left，center_horizontal

### 源码分析 目标
gravity最终影响的是父子元素的相对布局，猜测是在onLayout的时候，会根据gravity属性去做不同的布局，所以关键点清晰：直接分析LinearLayout的onLayout函数。

```java
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (mOrientation == VERTICAL) {
            layoutVertical(l, t, r, b);
        } else {
            layoutHorizontal(l, t, r, b);
        }
    }
```

然后分析下layoutVertical(l, t, r, b);--ps:好好学习下官方是怎么自定义View的......
```java
/**
     * Position the children during a layout pass if the orientation of this
     * LinearLayout is set to {@link #VERTICAL}.
     *
     * @see #getOrientation()
     * @see #setOrientation(int)
     * @see #onLayout(boolean, int, int, int, int)
     * @param left
     * @param top
     * @param right
     * @param bottom
     */
    void layoutVertical(int left, int top, int right, int bottom) {
        ......
        
        final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
        final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

        //确定基点
        switch (majorGravity) {
           case Gravity.BOTTOM:
               // mTotalLength contains the padding already
               childTop = mPaddingTop + bottom - top - mTotalLength;
               break;

               // mTotalLength contains the padding already
           case Gravity.CENTER_VERTICAL:
               childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
               break;

           case Gravity.TOP:
           default:
               childTop = mPaddingTop;
               break;
        }

        //分别对每一个子View进行布局
        for (int i = 0; i < count; i++) {
            final View child = getVirtualChildAt(i);
            if (child == null) {
                childTop += measureNullChild(i);
            } else if (child.getVisibility() != GONE) {
                final int childWidth = child.getMeasuredWidth();
                final int childHeight = child.getMeasuredHeight();

                final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
                //涉及到layout_garvity的地方了，如果自己的gravity与子View
                //的layout_gravity冲突，以子View的layout_gravity为准。
                //否则设置gravity为自己的gravity--关键点
                int gravity = lp.gravity;
                if (gravity < 0) {
                    gravity = minorGravity;
                }
                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                                + lp.leftMargin - lp.rightMargin;
                        break;

                    case Gravity.RIGHT:
                        childLeft = childRight - childWidth - lp.rightMargin;
                        break;

                    case Gravity.LEFT:
                    default:
                        childLeft = paddingLeft + lp.leftMargin;
                        break;
                }

                if (hasDividerBeforeChildAt(i)) {
                    childTop += mDividerHeight;
                }

                childTop += lp.topMargin;
                setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                        childWidth, childHeight);
                childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

                i += getChildrenSkipCount(child, i);
            }
        }
    }
```

### 勘误else

    void layoutVertical(int left, int top, int right, int bottom) {
        .........
        //本人感觉对应的是 android:layout_gravity属性，即自己在父容器的位置
        final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
        //本人感觉对应的是android:gravity，即子组件的默认位置
        final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
        .........
    }
    版权声明：本文为CSDN博主「jike0901xuye」的原创文章，遵循CC 4.0 by-sa版权协议，转载请附上原文出处链接及本声明。
    原文链接：https://blog.csdn.net/jike0901xuye/article/details/47658529

感觉这里的这两个注释应该不对。
mGravity应该就是对应自己的gravity属性，不会跟layout_gravity这样扯上关系的。
上文说过layout_gravity是对应在父容器的LayoutParams中的gravity的
在LinearyLayout中在void measureHorizontal(int widthMeasureSpec, int heightMeasureSpec)也能看到lp.gravity的使用身影：
```java
            if (baselineAligned) {
                final int childBaseline = child.getBaseline();
                if (childBaseline != -1) {
                    // Translates the child's vertical gravity into an index
                    // in the range 0..VERTICAL_GRAVITY_COUNT
                    final int gravity = (lp.gravity < 0 ? mGravity : lp.gravity)
                            & Gravity.VERTICAL_GRAVITY_MASK;
                    final int index = ((gravity >> Gravity.AXIS_Y_SHIFT)
                            & ~Gravity.AXIS_SPECIFIED) >> 1;

                    maxAscent[index] = Math.max(maxAscent[index], childBaseline);
                    maxDescent[index] = Math.max(maxDescent[index], childHeight - childBaseline);
                }
            }
```

### 延伸
关于Gravity，在使用中有很多 mGravity & Gravity.VERTICAL_GRAVITY_MASK的写法，有必要单独研究下Gravity这个单独的类，看下这里的&各种标记的设计需求与原理