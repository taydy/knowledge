# IP

IP 地址是一个网卡在网络世界的通讯地址，相当于现实世界的门牌号码。

一个 IP 地址被分割为四个部分，每个部分 8 个 bit，总共 32 位。IP 地址被分为了 5 类：

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb6q67ktdfj30ua0cqtah.jpg)

在网络地址中，至少在当时设计的时候，对于 A、B、C 类主要分为两部分，前面一部分是网络号，后面一部分是主机号。

A、B、C 三类地址所能包含的主机数量如下所示：

![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb6q9gio4cj31080aatck.jpg)

很明显，C 类地址能你包含的最大主机数量实在太少了，只有 254 个。

D 类地址是**组播地址**。使用这一类地址，属于某个组的机器都能收到。这有点类似在公司里面大家都加入了一个邮件组。发送邮件，加入这个组的都能收到。

## 无类型域间选路（CIDR）

现在有一种折中的方式叫作**无类型域间选路**，简称 **CIDR**。

这种做法将 32 位的 IP 地址一分为二，前面是**网络号**，后面是**主机号**。比如 IP 10.100.122.2/24，中间有一个斜杠，斜杠后面有个数字 24。这种地址表示形式就是 CIDR。24 的意思是，32 位中，前 24 位是网络号，后 8 位是主机号。

伴随着 CID存在的，一个是**广播地址**，10.100.122.255。如果发送这个地址，所有 10.100.122 网络里面的机器都可以收到。

另一个是**子网掩码**，255.255.255.0。将子网掩码和 IP 地址进行 AND 计算，结果为 10.100.122.0, 这就是网络号。**将子网掩码和 IP 地址按位计算 AND，就可得到网络号。**



*Tips: 求 16.158.165.91/22 这个 CIDR 的网络第一个地址、子网掩码和广播地址？*

/22 不是 8 的整数倍，不好办，只能先变成二进制来看。16.158 的部分不会动，它占了前 16 位。中间的 165，变为二进制为‭10100101‬。除了前面的 16 位，还剩 6 位。所以，这 8 位中前 6 位是网络号，16.158.<101001>，而 <01>.91 是机器号。

第一个地址是 16.158.<101001><00>.1，即 16.158.164.1。子网掩码是 255.255.<111111><00>.0，即 255.255.252.0。广播地址为 16.158.<101001><11>.255，即 16.158.167.255。



## MAC 地址

MAC 地址是一个网卡的物理地址，用十六进制，6 个 byte 表示，比如 82:38:b0:75:d0:61。

MAC 地址是全局唯一的，不会有两个网卡有相同的 MAC 地址。

MAC 地址的通信范围局限在一个子网里面。例如 192.168.0.2/24 访问 192.168.0.3/24 可以用 MAC 地址。一旦跨子网，即从 192.168.0.2/24 到 192.168.1.2/24，MAC 地址就不行了，需要 IP 地址起作用。



## IP ADDR

Windows 上查看 IP 地址命令为 ipconfig，Linux 上是 ifconfig 和 ip addr。

```shell
[root@yoyadoc ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
258: vethd464a58@if257: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP 
    link/ether 82:38:b0:75:d0:61 brd ff:ff:ff:ff:ff:ff link-netnsid 1
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:16:3e:00:13:79 brd ff:ff:ff:ff:ff:ff
    inet 172.19.213.173/20 brd 172.19.223.255 scope global eth0
       valid_lft forever preferred_lft forever
```

在 IP 地址后面有个 **scope**，对于 eth0 这张网卡来说，该值 global，说明这张网卡是可以对外的，可以接收来自各个地方的包。对于 lo 来说是 host，说明这张网卡仅仅可以供本机相互通信。

lo 全称是 **loopback**，又称**环回接口**，往往会被分配到 127.0.0.1 这个地址。这个地址用于本机通信，经过内核处理后直接返回，不会出现在任何网络中。

在 IP 地址的上一行是 link/ether 82:38:b0:75:d0:61 brd ff:ff:ff:ff:ff:ff，这个就是 **MAC地址**。

在 eth0 后面的 <BROADCAST,MULTICAST,UP,LOWER_UP> 叫做 **net_device flags**，**网络设备的状态标识**。

UP 表示网卡处于启动状态；BROADCAST 表示这个网卡有广播地址，可以发送广播包；MULTICAST 表示网卡可以发送多播包；LOWER_UP 表示 L1 是启动的，也即插着网线。

MTU 1500 表示最大传输单元 MTU 为 1500，这是以太网的默认值。

qdisc 全称是 **queueing discipline**，中文叫**排队规则**。内核如果需要通过某个网络接口发送数据包，它都需要按照为这个接口配置的 qdisc（排队规则）把数据包加入队列。

最简单的 qdisc 是 pfifo，它不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列。pfifo_fast 稍微复杂一些，它的队列包括三个波段（band）。在每个波段里面，使用先进先出规则。

三个波段（band）的优先级也不相同。band 0 的优先级最高，band 2 的最低。如果 band 0 里面有数据包，系统就不会处理 band 1 里面的数据包，band 1 和 band 2 之间也是一样。数据包是按照服务类型(Type of Service, TOS)被分配到三个波段（band）里面的。TOS 是 IP 头里面的一个字段，代表了当前的包是高优先级的，还是低优先级的。



> net-tools 通过 procfs(/proc) 和 ioctl 系统调用去访问和改变内核网络配置，而 iproute2 则通过 netlink 套接字接口与内核通讯。抛开性能而言，iproute2 的用户接口比 net-tools 显得更加直观。比如，各种网络资源（如 link、IP 地址、路由和隧道等）均使用合适的对象抽象去定义，使得用户可使用一致的语法去管理不同的对象。



## 动态主机配置协议(DCHP)

IP 地址可以用命令行配置，设置好后将网卡 up 一下即可。

```shell
## net-tools
$ sudo ifconfig eth1 10.0.0.1/24
$ sudo ifconfig eth1 up

## iproute2
$ sudo ip addr add 10.0.0.1/24 dev eth1
$ sudo ip link set up eth1
```

手动配置自由度太大，很可能配置的 IP 会重复或者与其他局域网机器不在一个子网里，造成无法正常通信。所以，真正配置的时候，一定不是直接用命令配置的，而是放在一个配置文件里。**不同系统的配置文件格式不同，单是无非就是 CIDR、子网掩码、广播地址和网关地址**。

这里使用的自动配置协议叫作**动态主机配置协议（Dynamic Host Configuration Protocol）**，简称 **DCHP**。

有了这个协议，网络管理员就轻松多了。他只需要配置一段共享的 IP 地址。每一台新接入的机器都通过 DHCP 协议，来这个共享的 IP 地址里申请，然后自动配置好就可以了。等人走了，或者用完了，还回去，这样其他的机器也能用。

**如果是数据中心里面的服务器，IP 一旦配置好，基本不会变，这就相当于买房自己装修。DHCP 的方式就相当于租房。你不用装修，都是帮你配置好的。你暂时用一下，用完退租就可以了。**

### DCHP 工作方式

1. **DCHP Discover**

   当一台机器新加入一个网络的时候，使用 IP 地址 0.0.0.0 发送一个广播包，目的 IP 地址为 255.255.255.255。广播包封装了 UDP，UDP 封装了 BOOTP。

   ![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb7mwlc3v9j30do08g752.jpg)

2. **DCHP Offer**

   DCHP Server 收到新机器的请求后，选择一个空闲的 IP 给新机器，并保留它提供的 IP 地址，从而不会为其他 DCHP 客户分配此 IP 地址。

   ![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb7n0ht9anj30c708gt9n.jpg)

3. **Select DCHP Offer**

   新机器收到回复后，会选择其中一个 DCHP Offer，一般是最先到达的那个，并且会向网络发送一个 DCHP Request 广播数据包，包中包含客户端的 MAC 地址、接收的租约中的 IP 地址、提供此租约的 DCHP 服务器地址等。

   ![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb7ndbgl5qj30dk08fmy2.jpg)

   此时，由于还没有得到 DHCP Server 的最后确认，客户端仍然使用 0.0.0.0 为源 IP 地址、255.255.255.255 为目标地址进行广播。在 BOOTP 里面，接受某个 DHCP Server 的分配的 IP。

4. **DCHP Server Ack**

   当 DHCP Server 接收到客户机的 DHCP request 之后，会广播返回给客户机一个 DHCP ACK 消息包，表明已经接受客户机的选择，并将这一 IP 地址的合法租用信息和其他的配置信息都放入该广播包，发给客户机。

   ![](https://tva1.sinaimg.cn/large/006tNbRwgy1gb7nev6od6j30cn08n0tn.jpg)

### IP 地址的收回和续租

客户机会在租期过去 50% 的时候，直接向为其提供 IP 地址的 DHCP Server 发送 DHCP request 消息包。客户机接收到该服务器回应的 DHCP ACK 消息包，会根据包中所提供的新的租期以及其他已经更新的 TCP/IP 参数，更新自己的配置。这样，IP 租用更新就完成了。