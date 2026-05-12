# 四、跨主机通信-flannel

## flannel网络插件

```
官网资料
https://github.com/flannel-io/flannel

flannel是一个专门用于跨主机网络通信的解决方案技术；
flannel将tcp数据包封装在另一种网络数据包里进行路由转发，以及通信；
flannel可以让不同的docker主机创建的容器，共同有用一个集群中唯一的ip地址。
```

## flannel原理

```
Flannel为每个host分配一个subnet，容器从subnet中分配IP，这些IP可以在host间路由，容器间无需使用nat和端口映射即可实现跨主机通信。

每个subnet都是从一个更大的IP池中划分的，flannel会在每个主机上运flanneld的agent，负责从池子中分配subnet。

Flannel使用etcd存放网络配置、已分配的subnet、host的IP等信息

Flannel数据包在主机间转发是由backend实现的，目前已经支持UDP、VxLAN、host-gw、AWS VPC和GCE路由等多种backend。
```

## 环境配置

```
1. 确保ntp时间正确
2. etcd安装
3. docker安装
4. 关闭默认防火墙，selinux
```

## flannel架构图

![image-20220829145215904](http://book.bikongge.com/sre/2024-linux/image-20220829145215904.png)

```
通信原理
1. 跨主机的容器，可以直接访问目标容器的ip，数据包从容器内部的veth0发出去
2.报文通过veth pair 发送给 宿主机的vethxxx
3.宿主机的vethxxx连接宿主机的虚拟交换机 docker0   ，数据包从虚拟网桥 bridge docker0 发出去。
4. 此时docker0发出的数据包被转发给flannel0这个虚拟网卡，这是一个p2p的虚拟网卡（https://zh.wikipedia.org/wiki/%E5%B0%8D%E7%AD%89%E7%B6%B2%E8%B7%AF）
然后报文被发给对端监听的另一个flanneld程序。

5.源主机的flanneld程序将原本的数据包封装后，根据自己维护的路由表，投递给目的地的flanneld程序，数据包到达目标机器之后再被解包，然后进入flannel0虚拟网卡，并且一层层的转发给docker0的虚拟网卡，docker0找到自己要连接的容器，最后到达目标容器。

6.注意点，每个节点的容器分配不能冲突，flannel通过etcd数据库存储每个节点的可用IP地址网段，以及通过修改docker的启动参数，--bip=x.x.x.x 实现限制容器的启动IP范围。
```

## 部署逻辑顺序

```
必须先启数据库etcd  > flannel  > docker ，因为flanneld需要去etcd读取容器的网络信息。
```

## 机器环境

```
这里用的etcd单机模式，以及也可以单独部署在一个服务器上。

10.0.0.200  docker-200  etcd  flannel,docker

10.0.0.201  docker-201  flannel docker
```

### docker-200配置

```
[root@docker-200 ~]#yum install etcd -y

配置文件
cat > /etc/etcd/etcd.conf << 'EOF'
# [member]
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.200:2379,http://127.0.0.1:2379"

# [cluster]
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.200:2379"
EOF

[root@docker-200 ~]#systemctl start etcd
[root@docker-200 ~]#systemctl enable etcd
Created symlink from /etc/systemd/system/multi-user.target.wants/etcd.service to /usr/lib/systemd/system/etcd.service.
[root@docker-200 ~]#


测试etcd功能
[root@docker-200 ~]#etcdctl  cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://10.0.0.200:2379
cluster is healthy

[root@docker-200 ~]#etcdctl -C http://10.0.0.200:2379 cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://10.0.0.200:2379
cluster is healthy
[root@docker-200 ~]#

读写etcd数据

[root@docker-200 ~]#etcdctl -C http://10.0.0.200:2379 set /testdir/k1 "hello www.yuchaoit.cn"
hello www.yuchaoit.cn
[root@docker-200 ~]#
[root@docker-200 ~]#etcdctl -C http://10.0.0.200:2379 get /testdir/k1
hello www.yuchaoit.cn
[root@docker-200 ~]#

列出etcd数据
[root@docker-200 ~]#etcdctl -C http://10.0.0.200:2379 ls /testdir/
/testdir/k1
[root@docker-200 ~]#

防火墙设置，放行etcd的数据包,2379,2380两个端口

[root@docker-200 ~]#iptables -A INPUT -p tcp -m tcp --dport 2379 -m state --state NEW,ESTABLISHED -j ACCEPT

[root@docker-200 ~]#iptables -A INPUT -p tcp -m tcp --dport 2380 -m state --state NEW,ESTABLISHED -j ACCEPT


[root@docker-200 ~]#iptables -nL 
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:2379 state NEW,ESTABLISHED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:2380 state NEW,ESTABLISHED
```

### 2台机器部署flannel

```
1.安装flannel
yum install flannel -y

2.配置flannel
cp /etc/sysconfig/flanneld{,.bak}

# ls /etc/sysconfig/fl*
/etc/sysconfig/flanneld  /etc/sysconfig/flanneld.bak

cat > /etc/sysconfig/flanneld << 'EOF'
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://10.0.0.200:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/atomic.io/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
EOF
```

### etcd配置（200机器写入）

根据架构图，添加etcd兼职，写入flannel配置，以及网络信息（加大网段范围）

```
etcdctl mk /atomic.io/network/config '{"Network":"192.168.0.0/16"}'

[root@docker-200 ~]#
[root@docker-200 ~]#etcdctl get  /atomic.io/network/config
{"Network":"192.168.0.0/16"}
[root@docker-200 ~]#
[root@docker-200 ~]#
[root@docker-200 ~]#
[root@docker-200 ~]#etcdctl ls
/atomic.io
/testdir
[root@docker-200 ~]#
```

### 启动2个机器的flanneld程序

```
systemctl start flanneld.service
systemctl enable flanneld.service
```

![image-20220829160940904](http://book.bikongge.com/sre/2024-linux/image-20220829160940904.png)

查看flannel环境

```
[root@docker-200 ~]#netstat -tunlp|grep flannel
udp        0      0 10.0.0.200:8285         0.0.0.0:*                           45542/flanneld      
[root@docker-200 ~]#

[root@docker-201 ~]#netstat -tunlp|grep flannel
udp        0      0 10.0.0.201:8285         0.0.0.0:*                           14731/flanneld      
[root@docker-201 ~]#
```

### 配置docker关联flannel网络

![image-20220829161724723](http://book.bikongge.com/sre/2024-linux/image-20220829161724723.png)

```
0.查看flannel对docker修改网段信息，2个机器执行
[root@docker-201 ~]#cat /run/flannel/docker 
DOCKER_OPT_BIP="--bip=192.168.101.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1472"
DOCKER_NETWORK_OPTIONS=" --bip=192.168.101.1/24 --ip-masq=true --mtu=1472"
[root@docker-201 ~]#



cat /run/flannel/docker 

1.修改docker配置文件

vim /usr/lib/systemd/system/docker.service
# 修改参数
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=/run/flannel/docker
ExecStart=/usr/bin/dockerd -H fd://  $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always


2.重启docker(踩坑记录，之前于超老师修改过 daemon.json， 我干！)
systemctl daemon-reload
systemctl restart docker
```

### 添加防火墙规则

```
确保FORWARD链都是ACCEPT策略即可
```

### 查看网络环境

```
ifconfig
```

![image-20220829162815197](http://book.bikongge.com/sre/2024-linux/image-20220829162815197.png)

### 运行容器测试跨主机通信情况

```
docker-200机器
1.运行容器
docker run -it busybox /bin/sh
2.查看网络
ifconfig
3.访问目标机器上的容器

=======

docker-201机器
1.运行容器
docker run -it busybox /bin/sh
2.查看网络
ifconfig
3.访问目标机器上的容器
```

### 访问通信，抓包结果

> 200 > 201机器

![image-20220829165034622](http://book.bikongge.com/sre/2024-linux/image-20220829165034622.png)

> 201 > 200

![image-20220829165555592](http://book.bikongge.com/sre/2024-linux/image-20220829165555592.png)

### 最终的网络架构图

![image-20220829165252171](http://book.bikongge.com/sre/2024-linux/image-20220829165252171.png)