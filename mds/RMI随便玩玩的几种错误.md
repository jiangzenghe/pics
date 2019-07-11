---
title: RMI随便玩玩的几种错误
date: 2018-01-18 19:05:04
tags: java
---

最近在看HeadFirst 设计模式，看到代理的部分，在书中有涉及RMI的代码如下：

```java
public static void main(String[] args) {
    try {
         IService service = new ServiceImpl();
    
         Context nammingContext = new InitialContext();
    
         nammingContext.rebind("rmi://localhost/service01", service);
    
    } catch (Exception e) {
         e.printStackTrace();
    }
}
```

<!--more-->

在调试的时候，出现了几种错误，在此记录以下<br>

### 错误1：
```
javax.naming.ServiceUnavailableException [Root exception is java.rmi.ConnectException: Connection refused to host: localhost; nested exception is: 

java.net.ConnectException: Connection refused]

at com.sun.jndi.rmi.registry.RegistryContext.rebind(RegistryContext.java:163)

at com.sun.jndi.toolkit.url.GenericURLContext.rebind(GenericURLContext.java:251)

at javax.naming.InitialContext.rebind(InitialContext.java:433)

at com.test.rmi.error.catcher.TestRemoteServer.main(TestRemoteServer.java:18)

Caused by: java.rmi.ConnectException: Connection refused to host: localhost; nested exception is: 

java.net.ConnectException: Connection refused

at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:619)

at sun.rmi.transport.tcp.TCPChannel.createConnection(TCPChannel.java:216)

at sun.rmi.transport.tcp.TCPChannel.newConnection(TCPChannel.java:202)

at sun.rmi.server.UnicastRef.newCall(UnicastRef.java:342)

at sun.rmi.registry.RegistryImpl_Stub.rebind(Unknown Source)

at com.sun.jndi.rmi.registry.RegistryContext.rebind(RegistryContext.java:161)

... 3 more

Caused by: java.net.ConnectException: Connection refused

at java.net.PlainSocketImpl.socketConnect(Native Method)

at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)

at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)

at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)

at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)

at java.net.Socket.connect(Socket.java:589)

at java.net.Socket.connect(Socket.java:538)

at java.net.Socket.<init>(Socket.java:434)

at java.net.Socket.<init>(Socket.java:211)

at sun.rmi.transport.proxy.RMIDirectSocketFactory.createSocket(RMIDirectSocketFactory.java:40)

at sun.rmi.transport.proxy.RMIMasterSocketFactory.createSocket(RMIMasterSocketFactory.java:148)

at sun.rmi.transport.tcp.TCPEndpoint.newSocket(TCPEndpoint.java:613)


... 8 more
```

原因：未启动rmiregistry
命令start rmiregistry

### 错误2：
```
javax.naming.CommunicationException [Root exception is java.rmi.ServerException: RemoteException occurred in server thread; nested exception is:
java.rmi.UnmarshalException: error unmarshalling arguments; nested exception is:
java.lang.ClassNotFoundException: com.test.rmi.server.IService]
at com.sun.jndi.rmi.registry.RegistryContext.rebind(RegistryContext.java:142)
at com.sun.jndi.toolkit.url.GenericURLContext.rebind(GenericURLContext.java:231)
at javax.naming.InitialContext.rebind(InitialContext.java:408)
at com.test.rmi.server.TestRmiServer.main(TestRmiServer.java:70)
Caused by: java.rmi.ServerException: RemoteException occurred in server thread; nested exception is:
java.rmi.UnmarshalException: error unmarshalling arguments; nested exception is:
java.lang.ClassNotFoundException: com.test.rmi.server.IService
at sun.rmi.server.UnicastServerRef.oldDispatch(UnicastServerRef.java:400)
at sun.rmi.server.UnicastServerRef.dispatch(UnicastServerRef.java:248)
at sun.rmi.transport.Transport$1.run(Transport.java:159)
at java.security.AccessController.doPrivileged(Native Method)
at sun.rmi.transport.Transport.serviceCall(Transport.java:155)
at sun.rmi.transport.tcp.TCPTransport.handleMessages(TCPTransport.java:535)
at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run0(TCPTransport.java:790)
at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run(TCPTransport.java:649)
at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:895)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:918)
at java.lang.Thread.run(Thread.java:662)
at sun.rmi.transport.StreamRemoteCall.exceptionReceivedFromServer(StreamRemoteCall.java:255)
at sun.rmi.transport.StreamRemoteCall.executeCall(StreamRemoteCall.java:233)
at sun.rmi.server.UnicastRef.invoke(UnicastRef.java:359)
at sun.rmi.registry.RegistryImpl_Stub.rebind(Unknown Source)
at com.sun.jndi.rmi.registry.RegistryContext.rebind(RegistryContext.java:140)
... 3 more
Caused by: java.rmi.UnmarshalException: error unmarshalling arguments; nested exception is:
java.lang.ClassNotFoundException: com.test.rmi.server.IService
at sun.rmi.registry.RegistryImpl_Skel.dispatch(Unknown Source)
at sun.rmi.server.UnicastServerRef.oldDispatch(UnicastServerRef.java:390)
at sun.rmi.server.UnicastServerRef.dispatch(UnicastServerRef.java:248)
at sun.rmi.transport.Transport$1.run(Transport.java:159)
at java.security.AccessController.doPrivileged(Native Method)
at sun.rmi.transport.Transport.serviceCall(Transport.java:155)
at sun.rmi.transport.tcp.TCPTransport.handleMessages(TCPTransport.java:535)
at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run0(TCPTransport.java:790)
at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run(TCPTransport.java:649)
at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:895)
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:918)
at java.lang.Thread.run(Thread.java:662)
Caused by: java.lang.ClassNotFoundException: com.test.rmi.server.IService
at java.net.URLClassLoader$1.run(URLClassLoader.java:202)
at java.security.AccessController.doPrivileged(Native Method)
at java.net.URLClassLoader.findClass(URLClassLoader.java:190)
at java.lang.ClassLoader.loadClass(ClassLoader.java:306)
at java.lang.ClassLoader.loadClass(ClassLoader.java:247)
at java.lang.Class.forName0(Native Method)
at java.lang.Class.forName(Class.java:249)
at sun.rmi.server.LoaderHandler.loadProxyInterfaces(LoaderHandler.java:709)
at sun.rmi.server.LoaderHandler.loadProxyClass(LoaderHandler.java:653)
at sun.rmi.server.LoaderHandler.loadProxyClass(LoaderHandler.java:590)
at java.rmi.server.RMIClassLoader$2.loadProxyClass(RMIClassLoader.java:628)
at java.rmi.server.RMIClassLoader.loadProxyClass(RMIClassLoader.java:294)
at sun.rmi.server.MarshalInputStream.resolveProxyClass(MarshalInputStream.java:238)
at java.io.ObjectInputStream.readProxyDesc(ObjectInputStream.java:1528)
at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1490)
at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:1729)
at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1326)
at java.io.ObjectInputStream.readObject(ObjectInputStream.java:348)
... 12 more
```

原因：虽然启动了start rmiregistry，但是main的启动是通过eclipse启动，未在eclipse project的/build/classes或/bin目录下启动
无法找到定义的IService类，因为它是定义为remote的
要从folder is the root of your built files目录启动start rmiregistry命令。