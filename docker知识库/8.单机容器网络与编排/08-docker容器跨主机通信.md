# 08-docker容器跨主机通信

# **一、Docker网络基本原理**

直观上看，要实现网络通信，机器需要至少一个网络接口（物理接口或虚拟接口）与外界相通，并可以收发数据包；

> 此外，如果不同子网之间要进行通信，需要额外的路由机制。

[Docker](http://book.bikongge.com/sre/10-云原生容器编排/docker-all/www.yuchaoit.cn)中的网络接口默认都是虚拟的接口。虚拟接口的最大优势就是转发效率极高。

这是因为Linux通过在内核中进行数据复制来实现虚拟接口之间的数据转发，即发送接口的发送缓存中的数据包将被直接复制到接收接口的接收缓存中，而无需通过外部物理网络设备进行交换。

> 对于本地系统和容器内系统来看，虚拟接口跟一个正常的以太网卡相比并无区别，只是它速度要快得多。

Docker容器网络就很好地利用了Linux虚拟网络技术，在本地主机和容器内分别创建一个虚拟接口，并让它们彼此连通（这样的一对接口叫做veth pair）。

![image-20220828143051625](http://book.bikongge.com/sre/2024-linux/image-20220828143051625.png)

一般情况下，Docker创建一个容器的时候，会具体执行如下操作：

1.创建一对虚拟接口，分别放到本地主机和新容器的命名空间中；

2.本地主机一端的虚拟接口连接到默认的docker0网桥或指定网桥上，并具有一个以veth开头的唯一名字，如veth1234；

3.容器一端的虚拟接口将放到新创建的容器中，并修改名字作为eth0。这个接口只在容器的命名空间可见；

4.从网桥可用地址段中获取一个空闲地址分配给容器的eth0（例如172.17.0.2/16），并配置默认路由网关为docker0网卡的内部接口docker0的IP地址（例如172.17.42.1/16）。

完成这些之后，容器就可以使用它所能看到的eth0虚拟网卡来连接其他容器和访问外部网络。

用户也可以通过docker network命令来手动管理网络。

# **二、Docker网络默认模式**

按docker官方的说法，docker容器的网络有五种模式：

```
1）bridge模式，--net=bridge(默认)
这是dokcer网络的默认设置，为容器创建独立的网络命名空间，容器具有独立的网卡等所有单独的网络栈，是最常用的使用方式。
在docker run启动容器的时候，如果不加--net参数，就默认采用这种网络模式。安装完docker，系统会自动添加一个供docker使用的网桥docker0，我们创建一个新的容器时，
容器通过DHCP获取一个与docker0同网段的IP地址，并默认连接到docker0网桥，以此实现容器与宿主机的网络互通。

2）host模式，--net=host
这个模式下创建出来的容器，直接使用容器宿主机的网络命名空间。
将不拥有自己独立的Network Namespace，即没有独立的网络环境。它使用宿主机的ip和端口。

3）none模式，--net=none
为容器创建独立网络命名空间，但不为它做任何网络配置，容器中只有lo，用户可以在此基础上，对容器网络做任意定制。
这个模式下，dokcer不为容器进行任何网络配置。需要我们自己为容器添加网卡，配置IP。
因此，若想使用pipework配置docker容器的ip地址，必须要在none模式下才可以。

4）其他容器模式（即container模式），--net=container:NAME_or_ID
与host模式类似，只是容器将与指定的容器共享网络命名空间。
这个模式就是指定一个已有的容器，共享该容器的IP和端口。除了网络方面两个容器共享，其他的如文件系统，进程等还是隔离开的。

5）用户自定义：docker 1.9版本以后新增的特性，允许容器使用第三方的网络实现或者创建单独的bridge网络，提供网络隔离能力。
```

## bridge模式

bridge模式是docker默认的，也是开发者最常使用的网络模式。在这种模式下，docker为容器创建独立的网络栈，保证容器内的进程使用独立的网络环境， 实现容器之间、容器与宿主机之间的网络栈隔离。

同时，通过宿主机上的docker0网桥，容器可以与宿主机乃至外界进行网络通信。 其网络模型可以参考下图：

![image-20220828143324510](http://book.bikongge.com/sre/2024-linux/image-20220828143324510.png)

从上面的网络模型可以看出，容器从原理上是可以与宿主机乃至外界的其他机器通信的。

同一宿主机上，容器之间都是连接掉docker0这个网桥上的，它可以作为虚拟交换机使容器可以相互通信。

然而，由于宿主机的IP地址与容器veth pair的 IP地址均不在同一个网段，故仅仅依靠veth pair和namespace的技术，还不足以使宿主机以外的网络主动发现容器的存在。

为了使外界可以方位容器中的进程，docker采用了端口绑定的方式，也就是通过iptables的NAT，将宿主机上的端口流量转发到容器内的端口上。

```
举一个简单的例子，使用下面的命令创建容器，并将宿主机的3306端口绑定到容器的3306端口：
docker run -tid --name db -p 3306:3306 MySQL

在宿主机上，可以通过iptables -t nat -L -n，查到一条DNAT规则：

DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:3306 to:172.17.0.5:3306

上面的172.17.0.5即为bridge模式下，创建的容器IP。

很明显，bridge模式的容器与外界通信时，必定会占用宿主机上的端口，从而与宿主机竞争端口资源，对宿主机端口的管理会是一个比较大的问题。

同时，由于容器与外界通信是基于三层上iptables NAT，性能和效率上的损耗是可以预见的。
```

# **三、容器跨主机通信**

## 概述

就目前Docker自身默认的网络来说，单台主机上的不同Docker容器可以借助docker0网桥直接通信，这没毛病

> 而不同主机上的Docker容器之间只能通过在主机上用映射端口的方法来进行通信，有时这种方式会很不方便，甚至达不到我们的要求。宿主机端口不够了，冲突了怎么办？

![image-20220828144202651](http://book.bikongge.com/sre/2024-linux/image-20220828144202651.png)

因此位于不同物理机上的Docker容器之间直接使用本身的IP地址进行通信很有必要。

再者说，如果将Docker容器起在不同的物理主机上，我们不可避免的会遭遇到Docker容器的跨主机通信问题。本文就来尝试一下。

## 容器跨主机通信方案

![image-20220828144533100](http://book.bikongge.com/sre/2024-linux/image-20220828144533100.png)

## 方案1：基于iptables的静态路由

此时两台主机上的Docker容器如何直接通过IP地址进行通信？

一种直接想到的方案便是通过分别在各自主机中 **添加路由** 来实现两个centos容器之间的直接通信。我们来试试吧

### 方案原理图

由于使用容器的IP进行路由，就需要避免不同主机上的容器使用了相同的IP，为此我们应该为不同的主机分配不同的子网来保证。

于是我们构造一下两个容器之间通信的路由方案，如下图所示。

![image-20220828145417021](http://book.bikongge.com/sre/2024-linux/image-20220828145417021.png)

```
核心逻辑就是，利用iptables实现数据包转发

1. 主机A的容器数据包，网关指向机器B
```

### 数据包头

![image-20220828145937372](http://book.bikongge.com/sre/2024-linux/image-20220828145937372.png)

```
配置说明步骤

1. 两个宿主机的容器，分别有自己的网段，不得冲突了
2. 两台机器，都配置了静态路由规则，互相传递数据
3. 内核，防火墙要支持数据包转发
```

## 实践

### docker-200

```
1.修改docker0的网段
cat> /etc/docker/daemon.json  <<'EOF'
{
"bip":"192.168.100.1/24",
  "registry-mirrors" : [
    "https://ms9glx6x.mirror.aliyuncs.com"
  ],
  "insecure-registries":["http://10.0.0.200"]
}
EOF

2.重载docker
systemctl daemon-reload
systemctl restart docker

ip a |grep 192.168.100
```

### docker-201

```
1.修改docker0的网段
cat> /etc/docker/daemon.json  <<'EOF'
{
"bip":"192.168.200.1/24",
  "registry-mirrors" : [
    "https://ms9glx6x.mirror.aliyuncs.com"
  ]
}
EOF

2.重载docker
systemctl daemon-reload
systemctl restart docker

ip a |grep 192.168.200
```

## 添加静态路由，以及iptables

### docker-200

```
[root@docker-200 ~]#route -n
# 本机去往192.168.200.0网段的数据包，告诉它的网关是10.0.0.201，数据包的下一跳
route add -net 192.168.200.0/24  gw 10.0.0.201

[root@docker-200 ~]#route -n |grep 192.168.200
192.168.200.0   10.0.0.201      255.255.255.0   UG    0      0        0 ens33



# iptables规则，允许流量转发
iptables -A FORWARD -s 10.0.0.0/24 -j ACCEPT
```

### Docker-201

```
route add -net 192.168.100.0/24 gw 10.0.0.200
iptables -A FORWARD -s 10.0.0.0/24 -j ACCEPT
```

## 启动容器测试通信

![image-20220828163139296](http://book.bikongge.com/sre/2024-linux/image-20220828163139296.png)

### docker-200

```
docker run -it busybox /bin/sh

/ # ifconfig |grep 192.168
          inet addr:192.168.100.2  Bcast:192.168.100.255  Mask:255.255.255.0
/ #
```

### docker-201

```
[root@docker-201 ~]#docker run -it busybox /bin/sh
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
5cc84ad355aa: Pull complete 
Digest: sha256:5acba83a746c7608ed544dc1533b87c737a0b0fb730301639a0179f9344b1678
Status: Downloaded newer image for busybox:latest
/ # 
/ # ifconfig |grep 192.168
          inet addr:192.168.200.2  Bcast:192.168.200.255  Mask:255.255.255.0
/ #
```

### 通信结果

![image-20220828163225369](http://book.bikongge.com/sre/2024-linux/image-20220828163225369.png)

## tcpdump抓取数据包

![image-20220828164006535](http://book.bikongge.com/sre/2024-linux/image-20220828164006535.png)

```
yum install tcpdump -y
```

### 200机器

```
[root@docker-200 ~]#tcpdump -i ens33 -nn icmp
```

### 201机器

```
[root@docker-201 ~]#tcpdump -i ens33 -nn icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), capture size 262144 bytes
```