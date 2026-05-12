# 五、跨主机通信-macvlan

```
参考资料
https://ipwithease.com/what-is-macvlan/
```

![image-20220829171033826](http://book.bikongge.com/sre/2024-linux/image-20220829171033826.png)

## macvlan 用于 Docker 网络

```
如何理解macvlan技术
这种技术原理是 将物理网卡（ens33）虚拟成多个网卡

需要linux内核支持3.9+

[root@docker-200 ~]#lsmod |grep macvlan
macvlan                19239  1 macvtap

macvlan默认使用bridge模式，实现数据包转发。

这里更多是属于云计算网络的知识范畴。
```

在 Docker 中，macvlan 是众多 Docker 网络模型中的一种，并且是一种跨主机的网络模型，作为一种驱动（driver）启用（-d 参数指定）

> Docker macvlan 只支持 bridge 模式。

首先准备两个主机节点的 Docker 环境，搭建如下拓扑图示：

![image-20220829170547911](http://book.bikongge.com/sre/2024-linux/image-20220829170547911.png)

## 创建基于同一个macvlan网段的通信

```
分别在2个主机执行命令，创建macvlan网络环境即可
docker network create -d macvlan --subnet=172.16.10.0/24 --gateway=172.16.10.1 -o parent=ens33 macvlan_1


这条命令中，

-d 指定 Docker 网络 driver
--subnet 指定 macvlan 网络所在的网络
--gateway 指定网关
-o parent 指定用来分配 macvlan 网络的物理网卡
之后可以看到当前主机的网络环境，其中出现了 macvlan 网络：

[root@docker-200 ~]#docker network ls
NETWORK ID     NAME        DRIVER    SCOPE
266e88b88c07   bridge      bridge    local
f3971eca0c0f   host        host      local
ca0e2d298a44   macvlan_1   macvlan   local
ff6208d689fb   none        null      local
```

![image-20220829170802710](http://book.bikongge.com/sre/2024-linux/image-20220829170802710.png)

## 分别启动两个机器的容器

```
docker-200 启动容器，且指定用macvlan网络
--ip 指定容器 c1 使用的 IP，这样做的目的是防止自动分配，造成 IP 冲突
--network 指定 macvlan 网络

docker run -it --name www.yuchaoit.cn_macvlan --ip=172.16.10.2  --network macvlan_1  busybox 


docker-201机器
docker run -it --name www.yuchaoit.cn_macvlan --ip=172.16.10.3  --network macvlan_1  busybox
```

![image-20220829171346419](http://book.bikongge.com/sre/2024-linux/image-20220829171346419.png)

### 抓包看数据包走向

```
[root@docker-200 ~]#tcpdump -i ens33 -nn icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens33, link-type EN10MB (Ethernet), capture size 262144 bytes
01:14:20.937537 IP 172.16.10.3 > 172.16.10.2: ICMP echo request, id 8, seq 73, length 64
01:14:20.937569 IP 172.16.10.2 > 172.16.10.3: ICMP echo reply, id 8, seq 73, length 64
01:14:21.169355 IP 172.16.10.2 > 172.16.10.3: ICMP echo request, id 8, seq 68, length 64
01:14:21.170491 IP 172.16.10.3 > 172.16.10.2: ICMP echo reply, id 8, seq 68, length 64


可见，是直接基于ens33网卡转发的流量
```

> macvlan还支持不同网段之间的连接



# 六、跨主机通信方案-consul注册中心

```
官网

https://www.consul.io/docs

https://www.consul.io/docs/intro/usecases/what-is-a-service-mesh  服务网格
```

![image-20220829172530934](http://book.bikongge.com/sre/2024-linux/image-20220829172530934.png)

## 二进制部署consul

```
[root@docker-200 ~]#
[root@docker-200 ~]#wget https://releases.hashicorp.com/consul/1.4.4/consul_1.4.4_linux_amd64.zip
--2022-08-30 01:28:19--  https://releases.hashicorp.com/consul/1.4.4/consul_1.4.4_linux_amd64.zip
Resolving releases.hashicorp.com (releases.hashicorp.com)... 13.249.167.10, 13.249.167.20, 13.249.167.41, ...
Connecting to releases.hashicorp.com (releases.hashicorp.com)|13.249.167.10|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 34755554 (33M) [application/zip]
Saving to: ‘consul_1.4.4_linux_amd64.zip’

100%[===========================================================================>] 34,755,554  19.8MB/s   in 1.7s   

2022-08-30 01:28:22 (19.8 MB/s) - ‘consul_1.4.4_linux_amd64.zip’ saved [34755554/34755554]

[root@docker-200 ~]#


[root@docker-200 ~]#
[root@docker-200 ~]#unzip consul_1.4.4_linux_amd64.zip  
Archive:  consul_1.4.4_linux_amd64.zip
  inflating: consul                  
[root@docker-200 ~]#
[root@docker-200 ~]#mv consul /usr/bin/consul
[root@docker-200 ~]#
[root@docker-200 ~]#chmod +x /usr/bin/consul 
[root@docker-200 ~]#
[root@docker-200 ~]#consul -v
Consul v1.4.4
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
[root@docker-200 ~]#
[root@docker-200 ~]#

后台运行consul

[root@docker-200 ~]#nohup consul agent -server -bootstrap -ui -data-dir /var/lib/consul -client=10.0.0.200 -bind=10.0.0.200 &>/var/log/consul.log &
[1] 51445


$ 检查consul运行

[root@docker-200 ~]#jobs -l
[1]+ 51445 Running                 nohup consul agent -server -bootstrap -ui -data-dir /var/lib/consul -client=10.0.0.200 -bind=10.0.0.200 &>/var/log/consul.log &
[root@docker-200 ~]#


查看运行日志
tail -f /var/log/consul.log
```

## docker-200修改docker启动文件

```
# 注意去掉flannel的配置
[root@docker-200 ~]#vim /lib/systemd/system/docker.service


[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
containerd=/run/containerd/containerd.sock
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store consul://10.0.0.200:8500 --cluster-advertise 10.0.0.200:2375
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always


重启docker-200
[root@docker-200 ~]#systemctl daemon-reload
[root@docker-200 ~]#systemctl restart docker
[root@docker-200 ~]#
```

![image-20220829173702143](http://book.bikongge.com/sre/2024-linux/image-20220829173702143.png)

## 同样的重启docker201

```
修改当前机器docker地址信息即可。
# 参数
--cluster-store 指定了consul服务发现地址
--cluster-advertise 指定了本机服务注册地址，也可以用本机内网地址10.0.0.x代替
# 2375：未加密的docker socket,远程root无密码访问主机
# 表示将docker暴露在可访问的tcp的2375端口，通过ip:port远程管理docker
# 然后将自己的信息，注册到consul里 10.0.0.200:8500

#########
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
containerd=/run/containerd/containerd.sock
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store consul://10.0.0.200:8500 --cluster-advertise 10.0.0.201:2375
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

重启
#
[root@docker-201 ~]#vim /lib/systemd/system/docker.service
[root@docker-201 ~]#systemctl daemon-reload
[root@docker-201 ~]#systemctl restart docker
```

## 访问consul-ui

创建完后通过浏览器访问一下，可以看到这两台会自动注册上来，这样的话这两个主机之间就会进行通信。

![image-20220829175919626](http://book.bikongge.com/sre/2024-linux/image-20220829175919626.png)

## 容器网络了解（关于docker内置的overlay网络）

```
DOCKER的内置OVERLAY网络

内置跨主机的网络通信一直是Docker备受期待的功能，在1.9版本之前，社区中就已经有许多第三方的工具或方法尝试解决这个问题，例如Macvlan、Pipework、Flannel、Weave等。

虽然这些方案在实现细节上存在很多差异，但其思路无非分为两种： 二层VLAN网络和Overlay网络

简单来说，二层VLAN网络解决跨主机通信的思路是把原先的网络架构改造为互通的大二层网络，通过特定网络设备直接路由，实现容器点到点的之间通信。这种方案在传输效率上比Overlay网络占优，然而它也存在一些固有的问题。

这种方法需要二层网络设备支持，通用性和灵活性不如后者。

由于通常交换机可用的VLAN数量都在4000个左右，这会对容器集群规模造成限制，远远不能满足公有云或大型私有云的部署需求； 大型数据中心部署VLAN，会导致任何一个VLAN的广播数据会在整个数据中心内泛滥，大量消耗网络带宽，带来维护的困难。

相比之下，Overlay网络是指在不改变现有网络基础设施的前提下，通过某种约定通信协议，把二层报文封装在IP报文之上的新的数据格式。这样不但能够充分利用成熟的IP路由协议进程数据分发；而且在Overlay技术中采用扩展的隔离标识位数，能够突破VLAN的4000数量限制支持高达16M的用户，并在必要时可将广播流量转化为组播流量，避免广播数据泛滥。

因此，Overlay网络实际上是目前最主流的容器跨节点数据传输和路由方案。

容器在两个跨主机进行通信的时候，是使用overlay network这个网络模式进行通信；如果使用host也可以实现跨主机进行通信，直接使用这个物理的ip地址就可以进行通信。overlay它会虚拟出一个网络比如10.0.2.3这个ip地址。在这个overlay网络模式里面，有一个类似于服务网关的地址，然后把这个包转发到物理服务器这个地址，最终通过路由和交换，到达另一个服务器的ip地址。
```

### 图解overlay+consul实现跨主机网络通信

![image-20220829175149787](http://book.bikongge.com/sre/2024-linux/image-20220829175149787.png)

要实现overlay网络，我们会有一个服务发现。

比如说consul，会定义一个ip地址池，比如10.0.2.0/24之类的。

上面会有容器，容器的ip地址会从上面去获取。

获取完了后，会通过ens33来进行通信，这样就实现跨主机的通信。

## docker-200创建overlay网络

```
[root@docker-200 ~]#docker network create -d overlay --subnet=10.0.2.1/24 www.yuchaoit.cn_overlay_net 
5989e980ca834bf84257938c5dc533401a398e06db2ecba67c2f8763f5f5fdcb

[root@docker-200 ~]#docker network ls
NETWORK ID     NAME                          DRIVER    SCOPE
dfb82c33d669   bridge                        bridge    local
f3971eca0c0f   host                          host      local
ca0e2d298a44   macvlan_1                     macvlan   local
ff6208d689fb   none                          null      local
5989e980ca83   www.yuchaoit.cn_overlay_net   overlay   global
[root@docker-200 ~]#

在docker-200上创建的网络信息，会立即同步到docker-201上。
consul任意节点配置的k/v 都会同步到其他节点。
```

![image-20220829180146708](http://book.bikongge.com/sre/2024-linux/image-20220829180146708.png)

> 在docker-200上创建的网络信息，会立即同步到docker-201上。 consul任意节点配置的k/v 都会同步到其他节点。

![image-20220829180345247](http://book.bikongge.com/sre/2024-linux/image-20220829180345247.png)

## 测试跨主机的通信

![image-20220829180826546](http://book.bikongge.com/sre/2024-linux/image-20220829180826546.png)

到此，我们这里实现了跨主机通信，是通过overlay network这种网络模式进行通信的。