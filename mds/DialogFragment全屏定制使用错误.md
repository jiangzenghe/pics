---
title: DialogFragment全屏定制使用错误
date: 2019-02-02 17:55:10
tags: Android
---

###场景描述
根据看书的小蜗牛的示例，写了一个全屏Dialog的DialogFragment实现，结果在运行的时候发现，如果重写onCreteView使用自己的layout，
就会闪退，报错android.util.AndroidRuntimeException: requestFeature() must be called before adding content。
改为默认的DialogFragment的super.onCreateView(inflater, container, savedInstanceState);就不会崩溃，但是达不到全屏
的目的。

<!--more-->

看书的小蜗牛在文章中指出的几个关键点都有在onActivityCreated中写了：

```java
public void onActivityCreated(Bundle savedInstanceState) {
    <!--关键点1-->
        getDialog().getWindow().requestFeature(Window.FEATURE_NO_TITLE);
        //onActivityCreated中getView为非null，然后调用dialog.setContentView()
        super.onActivityCreated(savedInstanceState);
    <!--关键点2-->
        getDialog().getWindow().setBackgroundDrawable(new ColorDrawable(0x00000000));
        getDialog().getWindow().setLayout(WindowManager.LayoutParams.MATCH_PARENT, WindowManager.LayoutParams.MATCH_PARENT);
    }
```

onCreateView的写法如下：

```java
@Nullable
@Override
public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, Bundle savedInstanceState) {
// return super.onCreateView(inflater, container, savedInstanceState);
return inflater.inflate(R.layout.fragment_full_dialog, container, false);
}
```

报错android.util.AndroidRuntimeException: requestFeature() must be called before adding content
字面意思是不能在setContentView之后调用requestFeature，但是尼玛我明明是在setContentView之前调用的requestFeature啊

###寻找
反复实验，onCreateView默认的方法返回null，如果重写了则会让DialogFragment的mView不为空。
而在DialogFragment的onActivityCreated中，如果mView不为空，会调用setContentView，代码如下

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);

    if (!mShowsDialog) {
        return;
    }

    View view = getView();
    if (view != null) {
        if (view.getParent() != null) {
            throw new IllegalStateException(
                    "DialogFragment can not be attached to a container view");
        }
        mDialog.setContentView(view);
    }
    final Activity activity = getActivity();
    if (activity != null) {
        mDialog.setOwnerActivity(activity);
    }
    mDialog.setCancelable(mCancelable);
    mDialog.setOnCancelListener(this);
    mDialog.setOnDismissListener(this);
    if (savedInstanceState != null) {
        Bundle dialogState = savedInstanceState.getBundle(SAVED_DIALOG_STATE_TAG);
        if (dialogState != null) {
            mDialog.onRestoreInstanceState(dialogState);
        }
    }
}
```
但是看字面意思，总归是在setContentView之前调用了requestFeature，为什么会这样呢？

从报错堆栈的反面出发，来找下问题

```java
android.util.AndroidRuntimeException: requestFeature() must be called before adding content
at com.android.internal.policy.PhoneWindow.requestFeature(PhoneWindow.java:317)
at com.android.internal.app.AlertController.installContent(AlertController.java:231)
at android.app.AlertDialog.onCreate(AlertDialog.java:423)
at android.app.Dialog.dispatchOnCreate(Dialog.java:397)
at android.app.Dialog.show(Dialog.java:298)
at android.support.v4.app.DialogFragment.onStart(DialogFragment.java:416)
```

requestFeature() must be called before adding content是PhoneWindow的requestFeature被调用的时候，
如果mContentParentExplicitlySet==true，就会throw的Error，mContentParentExplicitlySet会在
setContentView的时候被修改为true，说白了就如同小蜗牛说的

比如需不需要Title，而布局样式在选定后就不能再改变了（大小可以），有些属性是选择布局文件的参考，如果是在setContentView之后再设定，就是失去了意义，另外Android也不允许在选定布局后，设置一些影响布局选择的属性，会抛出异常

那这里的问题可能并不是requestFeature的调用时机问题，而是由于什么原因，改变了mContentParentExplicitlySet之后，又调用了
requestFeature，仔细排查，发现generateLayout也会调用requestFeature，而generateLayout会被installDecor从而间接的被setContentView调用。那么有理由怀疑是Dialog导致它的PhoneWindow的setContentView被调用了两次。

此时突然想起，引入AlertDialog的时候是multiple choies的，就是说有android.app.AlertDialog,同时还有
android.support.v7.app.AlertDialog。两者的setContentView函数是不同的，把AlertDiglog都改为support版本的，这个
错误就消失了。。

仔细回归代码，在重写onCreateView的情况下，会在onActivityCreated调用mDialog.setContentView；然后在Fragment的
onStart方法会调用mDialog.show(),而在show中依次会调用onCreate，调用AlertController的installContent，仔细看
不同包的AlertDialog对应的AlertController也是不同的，分别对应
com.android.internal.app.AlertController和android.support.v7.app.AlertController,
两者的installContent实现是不同的，前者调用了mWindow.setContentView，分析到这里，真相大白。

###结论
跟support-v？有关
AlertDialog的正常版与support版本的代码实现并不相同

根本原因是调用了两次setContentView
另外严格注意兼容碎片包导致的问题，严格注意应该使用的版本库，一不小心就弄错了，一般来说最好使用兼容包。

###节外生枝

```java
java.lang.IllegalStateException: You need to use a Theme.AppCompat theme (or descendant) with this activity.
at android.support.v7.app.AppCompatDelegateImplV9.createSubDecor(AppCompatDelegateImplV9.java:354)
at android.support.v7.app.AppCompatDelegateImplV9.ensureSubDecor(AppCompatDelegateImplV9.java:323)
at android.support.v7.app.AppCompatDelegateImplV9.setContentView(AppCompatDelegateImplV9.java:275)
at android.support.v7.app.AppCompatDialog.setContentView(AppCompatDialog.java:88)
at android.support.v4.app.DialogFragment.onActivityCreated(DialogFragment.java:411)
at com.tiger.app.widget.FullScreenDialogFragment.onActivityCreated(FullScreenDialogFragment.java:30)
at android.support.v4.app.Fragment.performActivityCreated(Fragment.java:2355)
at android.support.v4.app.FragmentManagerImpl.moveToState(FragmentManager.java:1451)
at android.support.v4.app.FragmentManagerImpl.moveFragmentToExpectedState(FragmentManager.java:1759)
at android.support.v4.app.FragmentManagerImpl.moveToState(FragmentManager.java:1827)
at android.support.v4.app.BackStackRecord.executeOps(BackStackRecord.java:797)
```

以上报错是修改为support的Activity包之后，新的报错，跟主题有关的，根据对应提示添加一下就可以了

###拓展
在回顾下关于兼容包-support v4和v7等的知识，做下总结。