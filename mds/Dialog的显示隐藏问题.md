---
title: Dialog的显示隐藏问题
date: 2018-07-24 22:18:10
tags: Android
---

之前在iOS上出现过，当时解决了，但是没有做记录，细节给忽略了，现在在android环境下又出现，然后找了半天。

### 场景如下：
封装一个网络请求库，中间加入一个装饰者的progressWrapper，结果在连续好几个请求，第一个请求使用progress，后面几个没使用；或者是一个使用，其他几个没使用progress，顺序不管的情况，都会出现进度指示器一直显示的问题，但是调试了几次发现count是正常的，也正常调用了dialog.dismiss方法，但是就是毫无反应。。。

<!--more-->

```java
@Override
protected void onInitData(Bundle bundle) {
    super.onInitData(bundle);
    digitalSymbol = bundle.getString(BundleKeys.DIGITAL_SYMBOL);
    fiatSymbol = bundle.getString(BundleKeys.FIAT_SYMBOL);
    if(digitalSymbol == null || fiatSymbol == null) {
        digitalSymbol = "";
        fiatSymbol = "";
    }

    requestRedDots();
    requestNotice();
    //也是一个http请求
    requestOtcKycBean(false);
    requestCoinPair();
}
```

实际上stackoverflow有人一阵见血了。。查了半天，忽略了多个请求会设置不同的Context的问题，其他的请求虽然没有使用dialog，但是会改变
上下文context，
```java
private void setContext() {
    if(context == null)
        return;

    builder = new MaterialDialog.Builder(context);
    builder.content("正在处理请稍候...");
    builder.progress(true, 0);
    builder.cancelListener(this);
}
```

所以不要在setContext中build progress，把build放在show的同时来做绑定，解决问题。（踩过的坑还要踩，实在不可救药）

### <font color=red>stackover的回答如下：</font>
	9down voteaccepted

	It isn't dismissing because the AlertDialog you're calling AlertDialog.dismiss on isn't the same one that's shown.

	In other words, you're calling alert.show() and using dialog.dismiss(). To fix it call dialog.show().

	shareimprove this answer
	edited Mar 7 '14 at 19:33


	answered Mar 7 '14 at 19:18

	adneal
	26.2k997133

	Made an edit, you're calling AlertDialog.dismiss on the wrong AlertDialog. – adneal
<br>
### 新的问题：
改了之后，一般情况不会出现不消失的问题，但是还是在密集网络请求的时候，偶尔会出现dialog不消失，只是频率没有之前高。到底是什么问题，需要仔细排查一下原因。

需要在count处加同步，否则，还是会出现多个context，然后取消的时候没有取消多个，还是上述问题，通过修改show的时候build的方法并没有完全解决那个问题。应该是这样。(结论不对，这个新的问题好像是count-1之后还是大于0，导致无法调用dialog.dismiss)。

确实是count>0 而且，没有能回调 dialog dismiss的情况，但是并不是count-1之后还是大于0，而是根本没有回调，不走success也不走failure。代码如下：（挖的好深的坑）

```java
@Override
public void doGet(String url, Map<String, String> params, BtbCallBack callback) {
    if (isNetUnavailable(callback)) return;

    url = OkHttp3Utils.appendParams(url, params);
    //创建OkHttpClient请求对象
    OkHttpClient okHttpClient = getOkHttpClient();
    //创建Request
    Request.Builder builder = new Request.Builder();
    insertHeader(builder);
    Request request = builder.url(url).tag(tag).build();
    //得到Call对象
    Call call = okHttpClient.newCall(request);
    //执行异步请求
    call.enqueue(callback);
}
```

isNetUnavilable方法如下：
```java
private boolean isNetUnavailable(BtbCallBack callback) {
    boolean netIsEnable = NetWorkUtils.isNetWorkAvailable();
    if (!netIsEnable) {
        if (callback != null) {
            Log.e("progress unav", "catch unavalible");
            callback.NetworkUnavailable();
        }
        return true;
    }

    if(tag == null) {
        Log.e("progress unav", "catch tag null");
        return true;
    }
    return false;
}
```


<font color=red>有可能在show之后，很小的几率到执行doGet的时候检测到网络不可用，直接返回，那么进度条就无法消失了。
几率很小的原因是正常网络闪断的可能性很小，所以很难重现出来。
还有一种可能性很小的是tag为null了，也会在isNetUnavailable处返回，导致进度条不关闭。</font>

还有一种可能性很小的是cancel request的问题，取消请求的问题。目前分析也有可能导致进度条不关闭。
还有一种可能是OKHttp根本没有返回，没有返回成功或失败，超时设置并未生效。

还有一种可能是count未能正常计数，未能正常减1。
<font color=red>好像是这种场景引起的，前两个showloading，先请求的后回来响应，后请求的先回来响应，导致count计数错乱。然后showloading会一直显示，无法消除。  应该是线程的同步问题，同时修改count的值，导致异常
但是因为线程只有两个，一般不会出现这个问题。</font>

<font color=red>修改count的类型</font>
private AtomicInteger count;
可以解决线程不安全问题。AtomicInteger可以保证原子性。
<font color=red>注意：修改为volatile，并不能解决问题。volatile只能保证可见性，就是说实时读到i的最新值，但不能保证原子性</font>

### <font color=red>另外的衍生问题：
<font color=red>上述问题可以解决线程安全问题，但是有新的需求是1s延迟之后才显示动画，代码这样处理：</font>

```java
public ProgressCallback(BtbCallBack callback) {
    this.callback = callback;
    //两个showloading 同时请求，会以后一个是否可cancel为准
    progressCancel = canCancel;
    showProgress.setCanCancel(progressCancel);
    handler.postDelayed(showProgress, 1000);
    canCancel = true;
    showLoading = false;
}

private class ProgressRunnalbe implements Runnable {
    private boolean canCancel = true;

    public void setCanCancel(boolean canCancel) {
        this.canCancel = canCancel;
    }

    @Override
    public void run() {
        if (count.getAndIncrement() == 0) {
            builder.cancelable(canCancel);
            progressDialog = builder.build();
            Log.e("progress show", progressDialog.toString());
            progressDialog.show();
        }
    }
}
```

这个时候又出现其他问题了，又是因为延迟执行导致的线程安全问题，这种情况下，有可能数据回来的时候，已经隐藏了progress（<font color=red>执行隐藏，但是并没有dialog，所以什么都不做</font>），
但是真正的progress还没有开始show。。。。结果还是关不掉。看来必须要在这里加锁了。。。。
