# 四次挥手

用于断开连接，常称为四次挥手

![image-20211111214955779](../image/image-20211111214955779.png)

1. 断开前A和B都处于ESTABLISHED状态
2. 断开的时候A先发起断开请求FIN，并进入FIN_WAIT_1的状态
3. B收到A的断开请求后，发送应答ACK，并进入CLOSED_WAIT状态
4. A收到B的应答进入FIN_WAIT_2状态，如果这时候B直接断开，则A将永远停留在这个状态，TCP协议里面并没有对这个状态处理，但Linux有，可以调整tcp_fin_timeout这个参数，设置一个时间阈值来切换状态
5. 如果B没有直接断开，则B也会发送一个断开请求FIN + ACK给A，并且切换到LAST_ACK状态
6. A在收到B的断开请求后会发送一个应答ACK给B，这个最后的应答ACK B可能会收不到，此是B就会重新发送一个断开请求FIN + ACK给A，如果A此是已经断开了，那么B就永远不会收到A的ACK应答，所以TCP协议要求A最后需要等待一段时间，切换到TIME_WAIT状态，这个时间要足够长，确保信息从B到A再到B可以完成传递
7. B在收到A的应答ACK后切换到CLOSED
8. A在经过足够长的等待时间并且B没有再次发送断开请求的情况下自动切换到CLOSED状态



## A直接断开的问题

如果A直接断开，A的端口就空出来了，但B不知道，B原来发过的很多包很可能已经在路上了，如果A的端口被一个新的应用抢占，这个新的应用会收到上个连接中B发过来的包，虽然序列号是重新生成的，但这里要上一个双保险，防止产生混乱，因此需要等足够长的时间，等到原来B发送的所有包都传递结束，才能空处端口来

### 等待时间

等待的时间设置为2MSL，MSL是Maximum Segment Lifetime，报文最大生存时间，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃

因为TCP报文基于IP协议，IP头中有一个TTL域，是IP数据报可以经过的最大路由数，每经过一个路由TTL减1，当此值为0时数据报被丢弃，同时发送ICMP报文通知源主机

协议规定MSL为2min，实际应用中常用的是30s，1min和2min

