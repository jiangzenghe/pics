---
title: TransactionTooLargeException
date: 2018-07-08 21:42:49
tags: Android
---

发布的应用报了一堆这个错误，但是我一脸懵b，我压根没有传太多东西在binder里面啊。。
网上搜了N多，都是说不要传太多在binder里面啊诸如此类。。。

	When you get this exception in your application, please analyze your code.

	1. Are you exchanging lot of data between your services and application?
	2. Using intents to share huge data, (for example, the user selects huge number of files from gallery share press share, the URIs of the selected files will be transferred using intents)
	3. receiving bitmap files from service
	4. waiting for android to respond back with huge data (for example,getInstalledApplications() when the user installed lot of applications)
	5. using applyBatch() with lot of operations pending

<!--more-->

最后统计bug数据，发现报错的都是7.0,7.1或者8.0的机器，而且报错跟其他的分析不太一样，别的分析起码会说在什么时候确实传送了大数据，但是
我的报错是这样：

```
1 java.lang.RuntimeException:android.os.TransactionTooLargeException: data parcel size 533928 bytes
2 android.app.ActivityThread$StopInfo.run(ActivityThread.java:3968)
3 ......
4 Caused by:
5 android.os.TransactionTooLargeException:data parcel size 533928 bytes
6 android.os.BinderProxy.transactNative(Native Method)
7 android.os.BinderProxy.transact(Binder.java:767)
8 android.app.IActivityManager$Stub$Proxy.activityStopped(IActivityManager.java:4657)
9 android.app.ActivityThread$StopInfo.run(ActivityThread.java:3952)
10 android.os.Handler.handleCallback(Handler.java:790)
11 android.os.Handler.dispatchMessage(Handler.java:99)
12 android.os.Looper.loop(Looper.java:164)
13 android.app.ActivityThread.main(ActivityThread.java:6600)
14 java.lang.reflect.Method.invoke(Native Method)
15 com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:518)
16 com.android.internal.os.ZygoteInit.main(ZygoteInit.java:821)
```

关键词<font color=red>activityStopped</font>其他的是没有这个关键词的

最后在stackoverflow搜到这样一篇：
报错一模一样

I updated Nexus 5X to Android N, and now when I install the app (debug or release) on it I am getting TransactionTooLargeException on every screen transition that has Bundle in extras. The app is working on all other devices. The old app that is on PlayStore and has mostly same code is working on Nexus 5X. Is anyone having the same issue?

```
java.lang.RuntimeException: android.os.TransactionTooLargeException: data parcel size 592196 bytes
   at android.app.ActivityThread$StopInfo.run(ActivityThread.java:3752)
   at android.os.Handler.handleCallback(Handler.java:751)
   at android.os.Handler.dispatchMessage(Handler.java:95)
   at android.os.Looper.loop(Looper.java:154)
   at android.app.ActivityThread.main(ActivityThread.java:6077)
   at java.lang.reflect.Method.invoke(Native Method)
   at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:865)
   at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:755)
Caused by: android.os.TransactionTooLargeException: data parcel size 592196 bytes
   at android.os.BinderProxy.transactNative(Native Method)
   at android.os.BinderProxy.transact(Binder.java:615)
   at android.app.ActivityManagerProxy.activityStopped(ActivityManagerNative.java:3606)
   at android.app.ActivityThread$StopInfo.run(ActivityThread.java:3744)
   at android.os.Handler.handleCallback(Handler.java:751) 
   at android.os.Handler.dispatchMessage(Handler.java:95) 
   at android.os.Looper.loop(Looper.java:154) 
   at android.app.ActivityThread.main(ActivityThread.java:6077) 
   at java.lang.reflect.Method.invoke(Native Method) 
   at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:865) 
   at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:755) 
```

然后靠谱的回答是：

> At the end, my problem was with things that were being saved onSaveInstance, and not with things that were being sent to next activity. I removed all saves where I can't control a size of objects (network responses), and now it's working.

Update:

> To preserve big chunks of data, Google is suggesting to do it with Fragment that retains instance. Idea is to create empty Fragment without a view with all necessary fields, that would otherwise be saved in Bundle. Add setRetainInstance(true); to Fragment's onCreate method. And then save data in Fragment on Activity's onDestroy and load them onCreate. Here is an example of Activity:

```
public class MyActivity extends Activity {

    private DataFragment dataFragment;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);

        // find the retained fragment on activity restarts
        FragmentManager fm = getFragmentManager();
        dataFragment = (DataFragment) fm.findFragmentByTag(“data”);

        // create the fragment and data the first time
        if (dataFragment == null) {
            // add the fragment
            dataFragment = new DataFragment();
            fm.beginTransaction().add(dataFragment, “data”).commit();
            // load the data from the web
            dataFragment.setData(loadMyData());
        }

        // the data is available in dataFragment.getData()
        ...
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        // store the data in the fragment
        dataFragment.setData(collectMyLoadedData());
    }
}
```

And example of Fragment:

```java
public class DataFragment extends Fragment {

    // data object we want to retain
    private MyDataObject data;

    // this method is only called once for this fragment
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // retain this fragment
        setRetainInstance(true);
    }

    public void setData(MyDataObject data) {
        this.data = data;
    }

    public MyDataObject getData() {
        return data;
    }
}
```
More about it, you can read [here.](https://developer.android.com/guide/topics/resources/runtime-changes.html)


Done
最后应该是解决这个问题了，over，要细心发现区别，找到关键，不要听信不明白人的误导，也不要单纯相信报错，之前总结过很多次了。
