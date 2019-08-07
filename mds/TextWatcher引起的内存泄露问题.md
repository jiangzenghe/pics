---
title: TextWatcher引起的内存泄露问题
date: 2017-11-21 20:23:27
tags: Android
---

### 场景描述
一次偶然的机会，听到同事说在activity中editText.addtextChangedListener(new TextWatcher),会引起内存泄漏，然后个人非常诧异，不能理解为什么会引起内存泄漏。

然后在google上搜索关于TextWatcher的内存泄漏问题，很多浅尝辙止的博文，写了那么两句，

    TextWatcher会引起activity内存泄露。
    EditText设置了addTextChangedListener的界面，要在onDestroy里调用removeTextChangedListener释放掉。


这些博文的描述毫无分析，只有一个简单的结论，所以自己还是决定确定下细节，为什么会引起内存泄漏。

<!--more-->

### 分析准备
Android的内存泄漏，一般都是因为占用了在Activity的onDestroy之后，还是有根GCRoot的链指向了该Activity，导致该Activity无法被释放，从而引起OOM。

常见的泄漏点有：
1. static修饰的变量，持有activity引用
2. 匿名内部类（默认也是持有activity引用），又在其中做耗时操作等（异步的任务等）
3. Handler造成的内存泄漏
4. Bitmap大图像等
5. 数据库Cursor游标等
6. 其他的比如流未关闭、广播registerReceiver未unregisterReceiver、Adapter未使用缓存contentView等


### 分析过程
首先进行理论分析
在@Jinwong的一篇博文中分析的比较详细，写的也比较好，其中有一片段是关于addTextChangedListener的泄露描述


    如果集合类仅仅有添加元素，而没有相应的删除机制，会导致内存被占用。当将集合中元素置空，但是集合因为持有对元素的引用，导致内存回收不，而发生内存泄露。解决方法是，可先删除元素然后置空，或者直接将集合置空。

    Android  中常见的集合类内存泄露有ValueAnimator调用addUpdateListener，EditText调用addTextChangedListener()而未注销监听导致内存泄露


查看EditText相关的源码，addTextChangedListener未remove，顶多也就影响mListeners里面多了一个实体，虽然new TextWatcher也是匿名内部类引用了activity，但是Activity在onDestroy之后，如果没有在TextWatcher中做什么耗时的异步任务，不会出现mListeners无法被回收的情况，（mListeners为privte的ArrayList）所以无从下结论说不remove这个TextWatcher的listener就会引起内存泄漏。未找到泄漏点

在实际应用中，模拟可内存泄漏的代码，使用leakCanary和profiler同时进行实验。
思路如下：
新建一LeakActivity，又MainActivity start，在onCreate的时候分配大量内存，然后写一种可造成oom的示例，反复进入LeakActivity，可瞬间导致memory占用飙升至几百m，然后换成addTextChangedListener未remove，看是否会出现内存泄漏即可。

```java
public class LeakActivity extends FragmentActivity {
    private EditText leakText;
    private List<Integer> floata = new ArrayList<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);

        leakText = findViewById(R.id.leak);
	//制造内存泄漏，直接分配很多M内存
        for(int i=0;i<1000000;i++) {
            floata.add(i);
        }

        leak();
    }

    public void leak() {
        //匿名内部类会引用其外围实例LeakAty.this,所以会导致内存泄漏
        new Thread(new Runnable() {

            @Override
            public void run() {
                while (true) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```
上述程序会在反复进入几次之后彪到一二百的内存，然后由于与匿名内部类中的耗时异步任务无法回收，会在profiler中看到明显的内存飙升并且最后oom


```java
leakText.addTextChangedListener(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {

            }

            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {

            }

            @Override
            public void afterTextChanged(Editable s) {

            }
        });
```
在屏蔽掉leak的代码之后，在onCreate在只添加TextWatcher并且不删除，反复进，观察到其实是能一直回收的，会稳定在100m左右，每次超过100就会回收掉。

### 结论
经过自己测试，发现单纯的EditText.addTextChangeListener(new TextWatcher)并不会引起内存泄漏。

相信自己的判断，并经过系统正确的验证，而非相信人云亦云。
