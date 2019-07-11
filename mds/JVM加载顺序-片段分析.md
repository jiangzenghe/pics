---
title: JVM加载顺序 片段分析
date: 2018-02-21 21:57:32
tags: java
---

看深入理解Java虚拟机，看到代码与虚拟机的分析章节。

```java
class Lava {
     private int speed = 5;

     void flow() {
     }
}

class Volcano {
     public static void main(Sting[] args) {
          Lava lava = new Lava();
          lava.flow();
     }
}
```

<!--more-->

### **对应的字节码助记符**
Lava的默认构造方法

    0:aload_0
    1:invokespecial #10 //Object init()
    4:aload_0
    5:iconst_5
    6:putfield #12 //speed
    9:return

Volcano
default construct

    0:aload_0
    1:invokespecial #8 //Object init()
    4:return

Volcano
static main()

    0:new #16 //Lava
    3:dup
    4: invokespecial #18 //Lava init()
    7:astore_1
    8:aload_1
    9: invokespecial #19 //Lava flow()
    12:return

### 运行分析：

* **执行Volcano的main方法**
    > 1.执行Lava lava = new Lava()
    java代码new Lava
    对应字节码new + dup + invokespecial init + astore
    > 2.lava.flow
    java代码lava.flow
    对应字节码aload + invokespecial flow

**总顺序**
装载Volcano
开始执行main 装载Lava
常量池替换 （解析）-- 符号引用转换为直接引用
终于 开始分配内存 （开始显示对应字节码的new指令）-- speed初始值设为默认值0
Lava对象压入栈中（其实是两次压入 对应dup指令）
调用Lava的构造方法 （构造方法中把speed的值初始化为5）
最后调用flow方法


1. 装载-----查找并装载类型的二进制数据
2. 连接-----执行验证、准备和解析（可选）
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;验证: 确保被导入的类型的正确性
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;准备: 为类变量分配内存，并将其初始化为默认值
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;解析: 把类型中的符号引用转换为直接引用
3. 初始化-----把类变量初始化为正确初始值。

### 零碎小总结索引：
LineNumberTable属性存在于Code属性中， 它建立了字节码偏移量到源代码行号之间的联系。