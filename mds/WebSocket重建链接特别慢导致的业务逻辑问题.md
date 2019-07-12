---
title: WebSocket重建链接特别慢导致的业务逻辑问题
date: 2019-01-21 11:43:11
tags:
---

### 基本情况是：
做的期货产品，新增了期权交易的模块

期权Activity作为独立的Module，进入的时候需要
    &nbsp;&nbsp;&nbsp;&nbsp;一：请求Http获取到标的物信息（BTC or ETH）
    &nbsp;&nbsp;&nbsp;&nbsp;二：建立自己的WebSocket链接，该链接不同于期货模块的WebSocket。
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;期货的WebSocket在Application初始化的时候就建立链接，期权的必须在第一次进入期权Activity（onCreate）的时候初始化链接，然后退出的时候（onDestroy）的时候断开链接，然后下次进入再重新初始化。
    &nbsp;&nbsp;&nbsp;&nbsp;三：如果在期权Activity打开新的界面或者关屏，然后返回或者开屏（触发onResume），需要重新请求标的物Http，也需要重新订阅WebSocket，但是不需要重连。
    
<!--more-->

**问题表现：**
在测试环境，每次第一次进入期权Activity（登录情况下），都可以拿到账户信息（依赖WebSocket链接之后发送订阅账户的Channel），退出期权Activity再重进，都可以正常的再次订阅，继续拿到账户信息
但是在预发环境和正式环境，登录情况下第一次进入期权Activity，虽然很慢，但是一般也可以拿到账户信息，但是这个时候退出再进入就拿不到账户信息了，其他的订阅（历史点数据、新增点数据和收益率等的）都正常，然后点击充值跳转到其他界面，再返回，又可以拿到账户信息。

非常的诡异

### 分析：

问题可能在与订阅需要依赖: 1.标的物Http请求已拿到数据 2.WebSocket链接已建立，
但是在写代码的时候，在标的物数据回来之后，只是做了delay延迟几百毫秒就直接发送账户信息的订阅了，可能那个时候webSocket链接并未建立。
但是为什么其他订阅正常呢？然后跳转到其他界面再回来重新订阅就可以了。而且第一次进入界面一般也能订阅账户信息也可以拿到。

怀疑是http标的物返回比webSocket建立链接快，导致这个逻辑异常，但是刚才的三个问题点没有明确清晰化原因。（测试环境一是比较快，二是http请求和webSocket建立链接的时间都差不多，慢就都慢，快就都快，所以没有这个问题出现）

贴出当时写的代码逻辑：
    
    1. ```java
    onCreate() {
          //链接webSocket
          initWebSocket()
          //注册网络变化Receiver
          initNetReceiver()
    }
    ```
    
    2. ```java
    onResume() {
          //请求Http标的物
         reqAsset() {
              //回调中调用订阅webSocket
              setAssetList();
             subsCribeAccountAndInvestHis();
          }
    }
    ```
    
###    代码片段：
    
setAssetList中调用setCurrentAsset
```java
handler.postDelayed(new Runnable() {
    @Override
    public void run() {
        if (assetList != null && assetList.size() > 0 && isSocketConnect) {
            Log.e("reqType-subAsset", "addIndexAndAsset");
            addOptionSpotIndex();
            setModel(assetList.get(current));
        }
    }
}, 150);
```

subsCribeAccountAndInvestHis中调用
```java
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                  if (isSocketConnect) {
                    getAccountInfo();
                    if (showNewHis == false)
                        subscribeHis();
                  }
            }
        }, 500);
```
    
连续两个handler，订阅其他包括历史点数据的在前，延迟150，订阅账户信息的在后，再延迟500，结果却如上文所说，进入一次返回再重新进入，总是历史点数据的可以正常返回，账户的不可以。

因为isSocketConnect依赖与建立webSocket是否成功，所以判断，订阅账户信息的时候链接并未建立，但是订阅历史点数据的时候，链接已经建立。
但是我明明是延迟150就订阅历史点，延迟500才订阅的账户，为什么先订阅的成功了，后订阅的反而没成功呢？
又因为是正式环境，无法正常调试，所以只能一遍遍提交代码，腆着脸去跟主管说打包看看，提了五六次都没解决问题。。

后来看打印的日志，是历史点数据订阅有发，但是账户订阅没发，改了不管isSocketConnect状态都发，发现账户订阅的日志在前面，而且发了也是没有订阅成功的。
看来，handler并没有按预期的顺序去发送。。
但是做了简单的实验，是不存在这种情况的，当前时间+postDelay的时间少的先执行，另一个当前时间+postDelay时间多的后执行，这个没有问题。

唯一的解释是setAssetList走了两次，因为确实出现过打印了两次历史点数据订阅的情况，那样的话其实相当于有三个postDelay，先是历史点订阅的delay，然后是账户信息的delay，然后又是历史点的delay，但是只有最后一次的历史点的订阅才生效了。

最后还是用很low的标志位去解决问题，第一次来，在socket链接之后再调用reqAsset，就不存在订阅的问题
在onResume中判断是不是第一次，不是第一次就再调用reqAsset，防止第一次进来调两次reqAsset，也防止打开新页面再返回不凋reqAsset。

### 结论：
折腾四五个小时，耽误一群小伙伴+主管的时间，真的是无语了
再找下，历史点订阅为什么可以走两次。
目前只有两个地方可能导致走两次
setCurrentAsset
1. webSocket断线重连
2. assetLayout的onClickListener

结论是确实走了webSocket重连，然后其实都历史点订阅了两次，第二次都时候因为前面有两次delay，所以链接已经建立，最终才订阅生效了。

**走webSocket的原因是：**
实现了OptionNetReceiver的接收器，为了在断网然后重新联网的时候能够重新建立webSocket链接，但是在new OptionNetReceiver的时候，是放在onCreate中的，其实每次进来，除了正常的建立webSocket之外，OptionNetReceiver会进一次onReceive事件，导致每次都必然重连一次，重连的逻辑里，只是调用了历史点数据等的订阅，没有调用账户信息的订阅，所以才会出现上述诡异问题。

完结。还是基本功要扎实啊，下盘要稳，内功深厚才可以。不要跟拼多多一样。
结论1:<font color=red>handler.postDelay没问题，delay的时间少的先执行，执行时间就是postDelay调用时间+delay时间，不受任何其他影响。</font>
结论2:<font color=red>Receiver在注册的时候会走一次符合注册事件的onReceiver事件，而不仅仅是注册之后有变化才通知。注意！！！</font>