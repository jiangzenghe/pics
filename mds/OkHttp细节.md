---
title: OkHttp细节
date: 2018-04-04 16:38:44
tags: Android
---

OKHttp加入拦截器之后，返回response的Http1xStream&FixedLengthSource在read的时候有一个间接的buffer缓存。
在使用使用者自己的buffer 读数据的时候，其实是从间接buffer中来获取，为什么这两个buffer会出现数据不同的情况，仔细看下。

RealBufferedSource 的 内部类InputStream，使用者read的时候就是从该内部InputStream中读取数据流。

```java
@Override public InputStream inputStream() {
    return new InputStream() {
        @Override public int read() throws IOException {
            if (closed) throw new IOException("closed");
            if (buffer.size == 0) {
                long count = source.read(buffer, Segment.SIZE);
                if (count == -1) return -1;
            }
            return buffer.readByte() & 0xff;
        }
        
        @Override public int read(byte[] data, int offset, int byteCount) throws IOException {
            if (closed) throw new IOException("closed");
            checkOffsetAndCount(data.length, offset, byteCount);
        
            if (buffer.size == 0) {
                long count = source.read(buffer, Segment.SIZE);
                if (count == -1) return -1;
            }
            
            return buffer.read(data, offset, byteCount);
        }
        
        @Override public int available() throws IOException {
            if (closed) throw new IOException("closed");
            return (int) Math.min(buffer.size, Integer.MAX_VALUE);
        }
        
        @Override public void close() throws IOException {
            RealBufferedSource.this.close();
        }
    
        @Override public String toString() {
            return RealBufferedSource.this + ".inputStream()";
        }
    };
}
```

<!--more-->

<font color=red>
第16-19行代码 <br>
    if (buffer.size == 0) {<br>
        &nbsp;&nbsp;&nbsp;&nbsp;long count = source.read(buffer, Segment.SIZE);<br>
        &nbsp;&nbsp;&nbsp;&nbsp;if (count == -1) return -1;<br>
    }    <br>
</font>

这个地方两个buffer应该是同步的，前提是网络理想化和buffer设置的缓存大小一致。那么每次都会进入标红部分的从间接buffer读取数据的步骤。
如果使用者bufer缓存小于间接buffer缓存，会出现一次读不完的情况，那么标红部分就可能不满足条件所以不进入。

注：加入拦截器之后，返回的responseBody就已经是拦截器中的responseBody了，标红部分影响了使用间接buffer缓存做“通知”的部分，然后onResponse部分一般用于流的直接处理（所谓的使用者buffer缓存部分），调用堆栈为：
    
    onResponse(Call call, Response response)
    -->
    while ((len = in.read(buffer)) != -1) {
        Log.e("while test", "save");
        mappedBuffer.put(buffer, 0, len);
    }
    
    in.read(buffer)
    -->
    
<br>

标红部分<font color=red>
    if (buffer.size == 0) {
        &nbsp;&nbsp;&nbsp;&nbsp;long count = source.read(buffer, Segment.SIZE);
        &nbsp;&nbsp;&nbsp;&nbsp;if (count == -1) return -1;
    }
</font>

```java
@Override public int read(byte[] data, int offset, int byteCount) throws IOException {
    if (closed) throw new IOException("closed");
    checkOffsetAndCount(data.length, offset, byteCount);

    if (buffer.size == 0) {
        long count = source.read(buffer, Segment.SIZE);
        if (count == -1) return -1;
    }

    return buffer.read(data, offset, byteCount);
}
```
    
最后在buffer.read(data, offset, byteCount)处返回到while循环中的in.read(buffer)中去。

而标红部分会调用到拦截器中重写的ResponseBody的public BufferedSource source() {}方法中去（一般会在这里“<font color=blue>使用间接buffer缓存做“通知”</font>”）。