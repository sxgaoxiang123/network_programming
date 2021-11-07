# 一个新机器如何获取IP地址

![image-20211107210456833](C:\Users\gaoxiang7\AppData\Roaming\Typora\typora-user-images\image-20211107210456833.png)

1. 新来的机器使用IP地址0.0.0.0发送了一个广播包，目标IP是255.255.255.255。

   广播包封装了UDP，UDP封装了BOOTP。DHCP是BOOTP的增强版

2. 在这个广播包里面，新人大喊：我是新来的（Boot request），我的MAC地址是这个，我还没有IP，谁能租给我一个IP地址

![image-20211107211011944](C:\Users\gaoxiang7\AppData\Roaming\Typora\typora-user-images\image-20211107211011944.png)

3. 如果网络里配置了DHCP Server的话，它就相当于IP地址的管理员，它立刻可以获知有一个”新人“，需要租给它一个IP地址，这个过程称为**DHCP Offer**

4. DHCP Server为此客户保留为它提供的IP地址，不会为其他DHCP客户分配此IP

5. DHCP Server仍然使用广播地址作为目的地址，因为此时请求分配IP的客户没有自己的IP。
6. DHCP Server回复客户我分配了一个可用的IP给你
7. 除此之外，服务器还发送了子网掩码、网关和**IP地址租用期**等信息

![image-20211107212430524](C:\Users\gaoxiang7\AppData\Roaming\Typora\typora-user-images\image-20211107212430524.png)

8. 如果有多个DHCP Server，这个新机器会收到多个IP地址，它会选择其中一个DHCP Offer，一般是最先到达的那一个，并向网络发送一个DHCP Request广播数据包
9. 数据包中包含了客户端的MAC地址、接受到的IP地址、提供此地址的DHCP服务器地址，并请求撤销其他提供的IP地址，以便剩余的IP可以动态流转

![image-20211107212708887](C:\Users\gaoxiang7\AppData\Roaming\Typora\typora-user-images\image-20211107212708887.png)

10. 当DHCP Server接收到客户机的DHCP request之后，会广播返回给客户机一个DHCP ACK消息包，表明已经接受客户机的选择，并将这一IP地址的合法租用信息和其他的配置信息都放入该广播包，发给客户机

