# 三次握手

首先要建立一个连接，所以要先看连接维护问题

TCP 的连接建立，常常称为三次握手

```
A: 您好，我是A
B：您好，我是B
A：您好，B
```

三次握手除了双方建立连接外，还有就是要沟通TCP包的序号问题

A告诉B，发送包的序号从哪个开始，B同样告诉A，B发送的包序号起始是从哪个号开始的

每个连接都要有不同的序号，这个序号的其实序号是随时间变化的，可以看成一个32位计数器，每4微秒加1



## 握手过程

![image-20211111211348474](../image/image-20211111211348474.png)

为了维护连接，双方都要维护一个状态机，如上是连接建立过程中，双方的状态变化时序图

1. 客户端和服务端都处于CLOSED状态
2. 服务端主动监听某个端口，切换到LISTEN状态
3. 客户端主动发起连接SYN，之后处于SYN-SENT状态
4. 服务端收到发起的连接，返回SYN，并且ACK客户端的SYN，之后处于SYN-RCVD状态
5. 客户端收到服务端发送的SYN和ACK后，发送ACK的ACK，之后处于ESTABLISHED状态，此时客户端的一发一收已经成功
6. 服务端收到ACK的ACK后，处于ESTABLISHED状态，此时服务端的一发一收也已经成功

