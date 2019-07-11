---
title: GsonObjectCallBack奇怪的debug问题
date: 2018-05-16 18:33:49
tags: Android
---

说奇怪，也不奇怪，估计跟调试者的疏忽有关系。
先上代码

```java
public abstract class CustomGsonCallback2<T> extends GsonObjectCallback<T> {
    
    @Override
    public void onResponse(Call call, Response response) {
        String json = null;
        try {
            json = response.body().string();
        } catch (IOException e) {
            e.printStackTrace();
        }
        Gson gson = new Gson();
        ExecResult execResult = gson.fromJson(json, ExecResult.class);
        json = gson.toJson(execResult.data);
        Class clz = this.getClass();
        if (clz.getGenericSuperclass() instanceof ParameterizedType) {
            ParameterizedType type = (ParameterizedType) clz.getGenericSuperclass();
            Type[] types = type.getActualTypeArguments();
            Class<T> cls = (Class<T>) types[0];
            final T t = gson.fromJson(json, cls);
            handler.post(new Runnable() {
                @Override
                public void run() {
                    onSuccess(t);
                }
            });
        } else {
            handler.post(new Runnable() {
                @Override
                public void run() {
                    onSuccess(null);
                }
            });
        }

    }
}
```

<!--more-->

该类继承了OkHttp的Callback类。

现象：
1.
调试的时候，解析json出错了，原因很简单，没有按json格式来。
但是后来新建一个类（就是上面的类），打了断点在第7行<font color=red>json = response.body().string();</font>（标红部分），这次应该不会出错，但是竟然还是出错了。
细节表现为String json没有正常解析出来。单步就直接跳到OKHttp的RealCall的finally里面去了，下面的代码并没有执行，非常奇怪，表现好像就是try catch Error了。

2.
调试很多次，每次在标红部分去Evaluate一下response.body().string()的值，然后就发现会跳OkHttp的finally，如果单步调试进去，会发现okhttp把response关闭掉了，而且还会提示你response只会被读取一次，但是nnd我还没读呢啊，就给老子关闭了，错误如下：
<font color=red>Method throw 'java.lang.IllegalStateException' exception.</font>
Close的Exception，不会再继续往下走了。


### 最后的结论：
调试的时候使用Evaluate调试值会消耗流，系统认为你已经使用完流了，在继续执行，当然出问题啦。。。。可能就是这么简单把。还以为是大坑。
因为上述情况只在@fan同事那里出现，我这里没有重现。换了手机之后我和剑威两边都未出现，后来手机换回去，两边也再未出现，也不可能是系统什么的问题。那只有上面的解释是科学合理的。
