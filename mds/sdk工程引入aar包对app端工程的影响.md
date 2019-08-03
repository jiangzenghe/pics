---
title: sdk工程引入aar包对app端工程的影响
date: 2019-03-02 17:54:45
tags: Android
---

### 场景描述：
我们提供了一个供三方使用的库包
某个版本的lib库添加了腾讯的防水墙的验证码三方库，发布之前都是没有问题的，发布之后（经过混淆），线上因为弹出条件毕竟苛刻，测试
并没有实际验证，某一天使用库的人反馈说出现崩溃，并发送了崩溃日志

<!--more-->

```java
Lcom/tencent/captchasdk/R$layout;
at com.tencent.captchasdk.TCaptchaDialog.onCreate(Unknown Source:7)
at android.app.Dialog.dispatchOnCreate(Dialog.java:610)
at android.app.Dialog.show(Dialog.java:454)
at com.fota.option.utils.ValidCodeDialogUtils.showDialog(ValidCodeDialogUtils.java:96)
at com.fota.option.OptionPresenter.a(OptionPresenter.java:379)
at com.fota.option.h.a(OptionSdkPresenter.java:54)
at com.fota.option.core.a$1.run(BasePresenter.java:53)
at android.os.Handler.handleCallback(Handler.java:907)
at android.os.Handler.dispatchMessage(Handler.java:105)
at android.os.Looper.loop(Looper.java:216)
at android.app.ActivityThread.main(ActivityThread.java:7593)
at java.lang.reflect.Method.invoke(Native Method)
at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:524)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:987)
```

important-有助于帮助查找问题并解决
重新查看了下我们的库引用验证码库的方式，是直接把腾讯提供的aar包copy到我们的库工程的libs里面，最后我们提供给其他人使用的时候，
又是打包了我们自己的aar库上传至我们的maven仓库。

### 问题过程
先看日志，开始怀疑是混淆问题，后来仔细看了下，google下发现应该是找不到R里面的某个文件，跟一般的NoClassdefFoundError并
不同。最开始的修改implement为api的形式，并不能解决问题，还是NoClassdefFoundError。
如果腾讯能提供lib库工程的方式添加引用，那么就没什么问题，但是他们提供的是arr包，我们又是提供给其他人arr包，导致打出的包里面缺失
必要的layout文件。

接下来一开始的思路是提供验证码aar给接入商开发者，这样应该可以找到layout等资源，但是如果如果接入者直接copy验证码aar到自己的libs
本地库，那么在build的过程中就会跟我们的库包报重复Error。。
既然报了重复Error，我想在引用我们的库包的时候exclude出验证码aar包，但是试了各种命令格式都行不通。可能exclude命令并不能正确的
剔除aar的包或者没有找到正确的命令写法。遂打算放弃这种方案了，就算能找到方法，还是需要接入者配合去添加库，真的很扯。

上两步的方案失败之后，还是选择从我们提供的库里去把res资源打包进去，开始直接把res的资源copy到option的库res文件中，但是依然
com.tencent.captchasdk.TCaptchaDialog还是依然找不到R.layout.tcaptcha_popup，我的理解是如果引用libs中的验证码aar，
在找R的时候会到com.tentcent.captchasdk.R中去找layout，当然找不到。

由于时间紧张，而且验证码aar中的代码毕竟少，所以选择直接把代码copy到option库的源码中去，res也都copy过去，问题解决。
其实正经的解决方案应该是重写TCaptchaDialog，然后找layout的时候更改链接到option的包名下去找R，因为option的res中已经
添加了layout等。

```java
public static int getResourseIdByName(String packageName, String className, String name) {
Class r = null;
int id = 0;
try {
r = Class.forName(packageName + ".R");

Class[] classes = r.getClasses();
Class desireClass = null;

for (int i = 0; i < classes.length; i++) {
if(classes[i].getName().split("\\$")[1].equals(className)) {
desireClass = classes[i];

break;
}
}

if(desireClass != null)
id = desireClass.getField(name).getInt(desireClass);
} catch (ClassNotFoundException e) {
e.printStackTrace();
} catch (IllegalArgumentException e) {
e.printStackTrace();
} catch (SecurityException e) {
e.printStackTrace();
} catch (IllegalAccessException e) {
e.printStackTrace();
} catch (NoSuchFieldException e) {
e.printStackTrace();
}

return id;

}
```

原来使用R.layout.***的地方用getResourseIdByName(context.getPackageName(), "layout", "***")的方式。

### 问题解决
最后还是通过把aar包中的所有代码都copy到现代码中去，保持混淆，把资源都copy到现工程的res中才解决这问题。
重视：
以后还是要在自己这边严格按照流程过完才可以。否则就太坑了。

### 延伸
关于验证的核心应该在assets的tcaptcha_webview.html文件里，原生代码量并不多所以我可以直接copy解决这个问题，
后续可以研究下腾讯防水墙的实现细节。
