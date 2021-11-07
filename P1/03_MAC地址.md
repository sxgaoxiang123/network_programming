# MAC地址概念

link/ether fa:16:3e:fb:98:34 brd ff:ff:ff:ff:ff:ff

这个称为MAC地址，是一个网卡的物理地址，用十六进制的6个byte表示

MAC地址全局唯一，不会有两个网卡有相同的MAC地址，这是网卡自生产出来所具备的物理属性





# IP地址和MAC地址的区别

- IP地址类似收货地址，用于定位，多用于信息收发
- MAC地址类似身份证，多用于身份确认（MAC地址具有一定的定位功能，只限于在信息到达IP地址后）





# 网络设备状态标识

## <BROADCAST, MULTICAST, UP, LOWER_UP>

这个叫做net_device flags，网络设备的状态标识

- UP：表示网卡处于启动的状态
- BROADCAST：表示这个网卡有广播地址，可以发送广播包
- MULTICAST：表示网卡可以发送多播包
- LOWER_UP：表示L1是启动的，也即网线插着呢
- MTU1500：是指最大传输单元MTU为1500，这是以太网的默认值，以太网规定正文部分不允许超过1500个字节。正文里有IP的头、TCP的头、HTTP的头。如果放不下，就需要分片传输



## qdisc pfifo_fast

qdisc全称是queueing discipline，叫做排队规则。

内核如果需要通过某个网络接口发送数据包，它都需要按照qdisc（排队规则）把数据包加入队列。

最简单的qdisc是pfifo，它不对进入的数据包做任何处理，数据包采用先入先出的形式通过队列。

