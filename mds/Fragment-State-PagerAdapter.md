---
title: Fragment(State)PagerAdapter
date: 2018-06-08 12:59:23
tags: Android
---

### 问题描述
记得很久之前，曾经用ViewPager+Fragment+Adapter做过一个页面，在Activity中嵌入三个可滑动切换的Fragment，Activity自身可以刷新，按理说刷新之后嵌入的三个Fragment的数据也应该会刷新掉（onRefresh之后通过重新调用setAdapter刷新数据），但是好像出现问题，三个子Fragment的数据并没有刷新。这里找下原因与细节的差异。

补充：
并不只是三个Fragment的数据应该被刷新而没刷新，而是N个子Fragment的N需要被刷新，例如本来是3个子Fragment，但是新的数据过来，应该要变化为4个子Fragment，而且各Fragment的数据已经发生变化。

### 问题关键点
出现这种情况，应该是使用了FragmentPagerAdapter来做viewPager的设置，如果改为FragmentStatePagerAdapter，就不会出现这种情况了。
但是为什么FragmentStatePagerAdapter可以做到刷新数据，但是FragmentPagerAdapter不可以呢？

PS:<font color=red>关键的版本声明：</font>
本篇涉及的代码，是在android 6.0的大版本下的描述，在android 7.0即21版本下的版本查看，有些关于moveToState和detachFragment、removeFragment的源码并不一致，可能最后的分析仍然有效，官方代码两版本逻辑的可能没变。
但是仍然有必要申明在借鉴@爱看书的小蜗牛的源码分析之后，在21版本下发现代码实际有出入，导致非常难核实验证到问题的关键，最后在6.0版本才看到跟分析源码一致的代码。

<!--more-->

### 基础信息
两种FragmentPagerAdapter都继承自PagerAdapter，其实现最终会决定FragmentManager中管理Fragment的细节，所以在开始之前，有必要搞清楚FragmentManager对于Fragment的管理细节

重点关注moveToState函数，设计到Fragment的状态转化。

此处关注下@爱看书的小蜗牛这里关于FragmentManager的Trasaction栈的描述

    最后看一下Fragmentmanager中Transaction栈，FragmentManager的Transaction栈到底是做什么的呢？FragmentManager对于Fragment的操作是分批量进行的，在一个Transaction中有多个add、remove、attach操作，Android是有返回键的，为了支持点击返回键恢复上一个场景的操作，Android的Fragment管理引入Transaction栈，更方便回退，其实将一个Transaction的操作全部翻转：添加变删除、attach变detach，反之亦然。对于每个入栈的Transaction，都是需要出栈的，而且每个操作都有前后文，比如进入与退出的动画，当需要翻转这个操作，也就是点击返回键的时候，需要知道如何翻转，也就是需要记录当前场景，对于remove，如果没有入栈操作，说明不用记录上下文，可以直接清理掉。对于ViewPager在使用FragmentPagerAdapter/FragmentStatePagerAdapter的时候都不会addToBackStack，这也是为什么detach跟remove有时候表现一致或者不一致的原因。简单看一下出栈操作，其实就是将原来从操作翻转一遍，当然，并不是完全照搬，还跟当前的Fragment状体有关。
    
    作者：看书的小蜗牛
    链接：https://www.jianshu.com/p/6df5b232c902

注意这里理解这句话
"对于ViewPager在使用FragmentPagerAdapter/FragmentStatePagerAdapter的时候都不会addToBackStack，这也是为什么detach跟remove有时候表现一致或者不一致的原因"

FragmentManager 主维护三个List列表，mActive、mAdded和mBackStack。
mAdded列表是被当前添加到容器中去的，而mActive是全部曾经参与的Fragment集合，只要没有被remove，就会存在，可以认为mAdded的Fragment都是活着的，而mActive的Fragment却可能被处决并被置null，并且只有makeInactive函数会执行处决。

可以认为mActive的但是非mAdded状态的fragment，就是FragmentManager中被缓存的fragment。

### 源码分析
ViewPager的setAdapter会调用

```java
for (int i = 0; i < mItems.size(); i++) {
    final ItemInfo ii = mItems.get(i);
    <!--全部destroy-->
    mAdapter.destroyItem(this, ii.position, ii.object);
}
```
然后在populate中会调用instantiateItem

FragmentStatePagerAdapter
```java
    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment) object;

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        while (mSavedState.size() <= position) {
            mSavedState.add(null);
        }
        mSavedState.set(position, fragment.isAdded()
                ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
        mFragments.set(position, null);

        mCurTransaction.remove(fragment);
    }
    
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        // If we already have this item instantiated, there is nothing
        // to do.  This can happen when we are restoring the entire pager
        // from its saved state, where the fragment manager has already
        // taken care of restoring the fragments we previously had instantiated.
        if (mFragments.size() > position) {
            Fragment f = mFragments.get(position);
            if (f != null) {
                return f;
            }
        }

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        Fragment fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
        if (mSavedState.size() > position) {
            Fragment.SavedState fss = mSavedState.get(position);
            if (fss != null) {
                fragment.setInitialSavedState(fss);
            }
        }
        while (mFragments.size() <= position) {
            mFragments.add(null);
        }
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
        mFragments.set(position, fragment);
        mCurTransaction.add(container.getId(), fragment);

        return fragment;
    }
```

FragmentPagerAdapter
```java
    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        mCurTransaction.detach((Fragment)object);
    }
    
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // Do we already have this fragment?
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
            mCurTransaction.attach(fragment);
        } else {
            fragment = getItem(position);
            if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        if (fragment != mCurrentPrimaryItem) {
            fragment.setMenuVisibility(false);
            fragment.setUserVisibleHint(false);
        }

        return fragment;
    }
```

总而言之，State在destroy的时候是remove，而非State只是detach，这样在非State中并没有改变fragment的inActive状态，FragmentManager缓存中还是存有fragment的。
然后在setAdapter的时候调用instantiateItem，非State会去FragmentManager缓存中寻找，而State只会在本Adapter中寻找，否则就直接getItem，然后由于是新的Adapter，肯定找不到就会重建。不会去FragmentManager缓存中寻找。

### 结论
如果要想做到Fragment的ViewPager的setAdapter整体刷新，请使用FragmentStatePagerAdapter，
如果需要notifyDataSetChanged刷新，除了使用FragmentStatePagerAdapter并重写
```java
public int getItemPosition(@NonNull Object object) {
        return POSITION_NONE;
    }
```

### 延伸
为什么要用ViewPager.setAdapter刷新数据,而不用Fragment(State)PagerAdapter.notifyDataSetChanged?
Fragment(State)PagerAdapter.notifyDataSetChanged会调用到ViewPager.dataSetChanged.

是不是全部都用FragmentStatePagerAdapter就可以了？那什么时候直接适用FragmentPagerAdapter呢？

