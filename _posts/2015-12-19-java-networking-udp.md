---
layout: post
title: Java网络教程：UDP DatagramSocket
subtitle: 'java, UDP'
date: 2015-12-19T00:00:00.000Z
author: kylo
catalog: true
tags:
  - UDP
  - java
---

[原文地址](http://tutorials.jenkov.com/java-networking/udp-datagram-sockets.html "Java Networking: UDP DatagramSocket") 译者：Kylo

数据报套接字（DatagramSocket）是Java的网络通讯方式，它通过UDP而不是TCP实现。UDP位于IP层的上层，可以使用数据报套接字来发送和接收UDP数据包。

# UDP vs. TCP

UDP的工作原理和TCP稍有不同。通过TCP发送数据时，需要先创建一个TCP连接。只要连接建立，TCP就能确保你的数据成功到达接收方，否则它将告知你发生了错误。

使用UDP的话，只需把数据包发送给网络上的IP地址。你无法确保该数据包是否成功到达，你也无法确保数据包以期望的顺序到达。这意味着UDP在协议开销上（没有数据流的完整性检查）比TCP小。

UDP非常适合一些并不强调数据包完整的数据传输。比如，网络直播的电视信号传输。你想要的是到达客户端的信号尽可能的接近现场（_译者注：这里强调及时性_）。所以，丢失一两个画面并无大碍。你并不希望直播是为了确保每一个画面都可靠传到而延迟，宁愿忽略这些丢失的画面，也要确保画面总是最新的。

网络上的实时监控摄像也是个很好的例子。谁关心过去发生的事呢，你只想要监控现在。你不想只是因为为了把所有的画面都展现给监控人员而延迟30秒吧。这和录像有一点不同。在你往磁盘录入影像的时候，可能并不想丢掉任何一个画面。你宁愿慢点，也不愿过后无法回看和检查这些画面，尤其是有重要的事情发生的时候。

# 通过DatagramSocket发送数据

通过Java的DatagramSocket发送数据，首先要创建一个DatagramPacket。代码如下：

```java
byte[] buffer = new byte[65508];
InetAddress address = InetAddress.getByName("jenkov.com");
DatagramPacket packet = new DatagramPacket(buffer, buffer.length, address, 9000);
```

byte[]数组buffer就是要在UDP数据报中发送的数据。上面的65508是单个UDP数据包能发送的最大容量。DatagramPacket构造方法中的长度是buffer中的数据的长度，buffer中所有超出最大容量的数据将被丢弃。

InetAddress实例目标节点（比如服务器）的地址。InetAddress类代表一个IP地址（网络地址），getByName()方法返回一个InetAddress实例，这个实例带有与给定的主机名对应的IP地址。

port参数是服务器到接收方的数据监听UDP端口。UDP和TCP端口是不同的。一个主机可以拥有不同的进程分别监听UDP80端口和TCP80端口。

要发送DatagramPacket，需要创建用于发送数据的DatagramSocket。代码如下：

```java
DatagramSocket datagramSocket = new DatagramSocket();
```

发送数据要调用send()方法，如下：

```java
datagramSocket.send(packet);
```

下面是完整实例：

```java
DatagramSocket datagramSocket = new DatagramSocket();

byte[] buffer = "0123456789".getBytes();
InetAddress receiverAddress = InetAddress.getLocalHost();

DatagramPacket packet = new DatagramPacket(
buffer, buffer.length, receiverAddress, 80);
datagramSocket.send(packet);
```

# 通过DatagramSocket接收数据

通过DatagramSocket接收数据，首先要创建一个DatagramPacket，然后通过DatagramSocket's receive()方法写入。代码如下：

```java
DatagramSocket datagramSocket = new DatagramSocket(80);

byte[] buffer = new byte[10];
DatagramPacket packet = new DatagramPacket(buffer, buffer.length);

datagramSocket.receive(packet);
```

注意DatagramSocket实例化时通过构造方法传入的80。这个参数是DatagramSocket接收UDP数据包的UDP端口。正如之前所说，TCP和UDP端口是不同的，这里就不再赘述了，两个进程分别监听TCP80端口和UDP80端口不会有任何的冲突。

第二，这里创建了一个byte[]数组buffer和DatagramPacket实例。注意这里的DatagramPacket并没有目标节点的相关信息，这和创建用于发送的DatagramPacket不同。这是因为现在的DatagramPacket是用来接收数据的，而不是发送，所以并不需要目标地址。

最后，调用了DatagramSocket的receive()方法。这个方法阻塞了线程，知道接收到一个DatagramPacket。

接收到的数据位于DatagramPacket的byte[]数组buffer中，buffer可以通过调用方法来获得：

```java
byte[] buffer = packet.getData();
```

接收到的数据有多大，就留给你去探索了。你现在使用的协议应该指明一个UDP数据包传输的数据大小和一个结束标识符。

一个真正的服务器程序很可能在一个循环中调用receive()方法，然后把接收到的DatagramPacket传递给一个工作线程池，就像TCP服务器处理接收到的连接那样（详情参见[Java多线程服务器](http://tutorials.jenkov.com/java-multithreaded-servers/index.html)）。
