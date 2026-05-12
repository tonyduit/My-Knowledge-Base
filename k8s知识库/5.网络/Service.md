# 东西流量管理Service

# Label 和 Selector

## Label

创建一个测试资源：

```bash
kubectl create deploy nginx --image=registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12
```

### 添加标签（Label）

命令格式：

```bash
kubectl label 资源类型 [资源名称] [资源名称] key=value
```

具体操作：

```bash
# 指定单个资源添加标签：
kubectl label deploy nginx version=v1
deployment.apps/nginx labeled

# 查看标签：
kubectl get deploy nginx --show-labels
NAME    READY  UP-TO-DATE  AVAILABLE    AGE    LABELS
nginx   1/1        1           1        101s   app=nginx,version=v1

# 指定多个资源添加标签：
kubectl label deploy --all svc=true
deployment.apps/nginx labeled

# 根据已有标签过滤之后再添加标签：
kubectl label deploy -l app=nginx svc2=true
deployment.apps/nginx labeled

# 同时添加多个标签：
kubectl label deploy -l app=nginx a=b c=d
deployment.apps/nginx labeled
```



### 修改标签（Label） 

```bash
# 已经存在的标签名，不允许直接进行修改：
kubectl label deploy -l app=nginx version=v2
error: 'version' already has a value (v1), and --overwrite is false

# 如需修改，可以使用 --overwrite 参数：
kubectl label deploy -l app=nginx version=v2 --overwrite
deployment.apps/nginx labeled
```



### 删除标签（Label） 

```bash
# 删除 Key 为 version 的标签：
kubectl label deploy nginx version-
deployment.apps/nginx unlabeled

kubectl get deploy nginx --show-labels
NAME   READY    UP-TO-DATE AVAILABLE   AGE    LABELS
nginx  1/1 		1 		  1 	     8m31s   a=b,app=nginx,c=d,svc2=true,svc=true
```



## Selector 选择器

```bash
# 首先使用--show-labels 查看指定资源目前已有的 Label：
# kubectl get deploy --show-labels
NAME READY UP-TO-DATE AVAILABLE AGE LABELS
nginx 1/1 1 1 11m app=nginx,svc2=true,svc=true

# 查询 app 为 nginx 的 Deployment：
# kubectl get deploy -l app=nginx
NAME READY UP-TO-DATE AVAILABLE AGE
nginx 1/1 1 1 12m

# 查询 app 为 nginx 或 backend 的 Deployment：
# kubectl get deploy -l 'app in (nginx, backend)' --show-labels
NAME READY UP-TO-DATE AVAILABLE AGE LABELS
nginx 1/1 1 1 13m app=nginx,svc2=true,svc=true

# 查询 app 为 nginx 或 backend 但不包括 version=v1 的 Deployment：
# kubectl get deploy -l 'app in (nginx, backend)' -l version!=v1 --show-labels 
NAME READY UP-TO-DATE AVAILABLE AGE LABELS
nginx 1/1 1 1 14m app=nginx,svc2=true,svc=true

# 查询 label 的 key 名为 app 的 Deployment：
# kubectl get deploy -l app --show-labels
NAME READY UP-TO-DATE AVAILABLE AGE LABELS
nginx 1/1 1 1 15m app=nginx,svc2=true,svc=true

# 查询所有空间下的资源：
# kubectl get deploy -l 'app in (nginx, kube-dns)' --show-labels -A
NAMESPACE NAME READY UP-TO-DATE AVAILABLE AGE LABELS
default nginx 1/1 1 1 22m app=nginx,svc2=true,svc=true
kube-system coredns 2/2 2 2 29d app=kube-dns,k8s-app=kube-dns
```



## 使用案例

公司与 xx 银行有一条专属的高速光纤通道，此通道只能与 192.168.7.0 网段进行通信，因此只能将与 xx 银行通信的应用部署到 192.168.7.0 网段所在的节点上，此时可以对节点添加 Label：

```bash
# kubectl label node k8s-node02 region=subnet7
node/k8s-node02 labeled
```

然后可以通过 Selector 对其筛选：

```bash
# kubectl get no -l region=subnet7
NAME        STATUS    ROLES     AGE     VERSION
k8s-node02  Ready     <none>    3d17h   v1.12.3
```

最后在 Deployment 或其他控制器中指定将 Pod 部署到该节点：

```yaml
template: 
  metadata:
    ......
  spec:
    containers:
     ......
    dnsPolicy: ClusterFirst
    nodeSelector:
      region: subnet7
    restartPolicy: Always
    ......
```



# Service

## Service 资源定义

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: CoreDNS
    name: kubernetes
  namespace: kube-system
spec:
  clusterIP: 10.96.0.10 # Service 的 IP，不需要手动指定
  ports: # Service 端口配置
    - name: dns # Service 端口名字
      port: 53 # Service 端口
      protocol: UDP # 代理协议
      targetPort: 53 # 目标端口，程序端口
    - name: dns-tcp
      port: 53
      protocol: TCP
      targetPort: 53
  selector: # 代理到哪些 Pod
    k8s-app: kube-dns
  sessionAffinity: None # 会话保持配置
  type: ClusterIP # Service 类型
```

Service 支持将一个接收端口映射到任意的 targetPort，如果 targetPort 为空，targetPort 将被设置为与 Port 字段相同的值。targetPort 可以设置为一个字符串，引用 Pod 的一个端口的名称，这样的话即使更改了 Pod 的端口，也不会对 Service 的访问造成影响。
Kubernetes Service 能够支持 TCP、UDP、SCTP 等协议，默认为 TCP 协议。

> 有用的知识
>
> 流控制传输协议（SCTP，Stream Control Transmission Protocol）是一种在网络连接两端之间同时传输多个数据流的协议。SCTP 提供的服务与 UDP 和 TCP 类似。



## Service 类型

Kubernetes Service 主要包括以下几种类型：

- ClusterIP：在集群内部使用，默认值，只能从集群中访问。
- NodePort：在所有安装了 Kube-Proxy 的节点上打开一个端口，此端口可以代理至后端Pod，可以通过 NodePort 从集群外部访问集群内的服务，格式为 NodeIP:NodePort。
- ExternalName：通过返回定义的 CNAME 别名，没有设置任何类型的代理，需要 1.7 或更高版本 kube-dns 支持。
- LoadBalancer：使用云提供商的负载均衡器公开服务，成本较高。



## 定义 Service

创建 Service 可以使用 expose 命令和通过 yaml 文件定义。通过 expose：

```bash
kubectl expose deploy nginx --port 80
```

通过 yaml 文件定义 Service 如下：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

创建服务：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:1.15.12
          ports:
            - containerPort: 80
```

Service 名访问测试：

```bash
# 创建测试pod
[root@k8s-master01 ~]# kubectl create deploy cluster-test --image=registry.cn-beijing.aliyuncs.com/dotbalo/debug-tools -- sleep 3600
deployment.apps/cluster-test created

# 进入测试pod
# kubectl exec -ti cluster-test-5dbf5c5d-tnb45 -- bash
(130 12:46 cluster-test-5dbf5c5d-tnb45:/) nslookup my-service
Server: 10.96.0.10
Address: 10.96.0.10#53
Name: my-service.default.svc.cluster.local
Address: 10.104.233.79

# 同一命名空间下可以省略namespace
(12:46 cluster-test-5dbf5c5d-tnb45:/) curl my-service
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
 body {
 width: 35em;
 margin: 0 auto;
 font-family: Tahoma, Verdana, Arial, sans-serif;
 }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed 
and
working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



## NodePort 类型

如果将 Service 的 type 字段设置为 NodePort，则 Kubernetes 将从 **--service-node-port-range** 参数指定的范围（默认为 30000-32767）中自动分配端口，也可以手动指定 NodePort，创建该 Service后，集群每个节点都将暴露一个端口，通过某个宿主机的 IP+端口即可访问到后端的应用。

kubeadm安装k8s，配置文件目录

```bash
[root@k8s-master01 ~]# vim /etc/kubernetes/manifests/kube-apiserver.yaml 
```

二进制安装k8s

```bash
# 需要使用如下命令找到配置文件所在位置
systemctl status kube-apiserver
# 或
ps -ef | grep apiserver
```

定义一个 NodePort 类型的 Service 格式如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

## LoadBalancer 与 NodePort 类型的区别

在 Kubernetes 的逻辑里，**`LoadBalancer` 类型其实就是 `NodePort` 的超集。**

**1. 揭秘：Service 的嵌套关系**

当你创建一个 `type: LoadBalancer` 的 Service 时，Kubernetes 在后台实际上做了三件事：

1. **自动分配一个 ClusterIP**（集群内部访问）。
2. **自动分配一个 NodePort**（在所有节点开孔，供外部进入）。
3. **调用云厂商 API**（创建一个外部 LB，并将后端指向所有节点的 `NodeIP:NodePort`）。

所以，如果你手动配置一个 `NodePort` Service，然后再去阿里云或 AWS 手动买一个负载均衡器（SLB/ELB），把后端服务器填上集群节点的 IP，端口填上那个 `30xxx`：**其最终的流量链路与 `type: LoadBalancer` 几乎完全一致。**

**2. 既然效果一样，为什么还要用 LoadBalancer 类型？**

虽然“物理链路”一样，但**“自动化程度”**和**“生命周期管理”**天差地别：

| **维度**       | **手动 NodePort + 外部 LB**            | **自动 LoadBalancer Service**                 |
| -------------- | -------------------------------------- | --------------------------------------------- |
| **配置复杂度** | 需跨平台操作（K8s 指令 + 云控制台）    | 一行 YAML 搞定所有配置                        |
| **节点同步**   | 节点增删、扩容后，需手动修改 LB 后端   | **自动同步**：K8s 会通知云厂商更新后端 IP     |
| **健康检查**   | 需要手动配置 LB 对 NodePort 的探测     | **内置集成**：云厂商能直接获取 Pod 的就绪状态 |
| **成本管控**   | 删除 Service 后，LB 仍计费（需手动删） | 删除 Service 时，LB 随之**自动释放**          |

**3. 一个非常关键的区别：流量的“入口点”**

虽然链路一样，但 `type: LoadBalancer` 有一个独有的特权：**`status.loadBalancer.ingress`**。

- **手动模式**：K8s 不知道外面有个 LB，所以你的 Service 永远不会显示外部 IP。
- **自动模式**：云厂商分配 IP 后，会回传给 K8s。这样你运行 `kubectl get svc` 时，能直接看到 `EXTERNAL-IP`。很多自动化工具（如 ExternalDNS）依赖这个字段来自动解析域名。

**4. 关于你担心的“性能损耗”**

既然两者本质都是通过 NodePort 进来的，那么它们面临的**内核转发损耗**是一模一样的。

**如何规避这个损耗？**

无论是手动还是自动，你都可以使用我之前提到的 **`externalTrafficPolicy: Local`**。

- **在 LoadBalancer 模式下**：云厂商的 LB 会非常聪明。它发现 Node-B 上没 Pod，就不会往 Node-B 发流量，流量只进 Node-A，然后直达 Pod。
- **在手动 NodePort 模式下**：你需要自己给外部 LB 配置健康检查，确保它只往“有 Pod 的节点”发流量。

## ExternalName 代理域名

### 基本使用

ExternalName Service 是 Service 的特例，它没有 Selector，也没有定义任何端口和 Endpoint，它通过返回该外部服务的别名来提供服务，和域名解析的 CNAME 类似。

比如可以定义一个 Service，后端设置为一个外部域名，这样通过 Service 的名称即可访问到该域名。使用 nslookup 解析以下文件定义的 Service，集群的 DNS 服务将返回一个值为www.taobao.com 的 CNAME 记录：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-externalname
spec:
  type: ExternalName
  externalName: www.taobao.com
```



### 使用案例

假设某个项目具备 DEV/UAT 两个环境，每个环境需要链接指定的数据库等基础组件。基础组件同样也是在 K8s 中按照不同的环境进行划分和部署，比如 DEV 环境所用的基础组件均在basic-component-dev 命名空间下，以此类推。

为了降低配置文件的维护复杂度，准备使用 ExternalName 类型的 Service 对基础组件的连接地址进行映射，这样就可以用同名的 Service 区分不同的环境，从而降低配置文件维护的复杂度。比如配置了在同一个项目的不同环境里面都配置一个同名的 Redis Service，类型为ExternalName，并且按照不同环境指向不同的基础组件地址，这样每个项目的不同环境，都可以用 Redis 这一个地址就可以访问到不同基础组件。

环境准备：

```bash
# 创建 Namespace
# kubectl create ns basic-component-dev
# kubectl create ns basic-component-uat

# 创建服务
# kubectl create deploy redis -n basic-component-dev --image=registry.cn-beijing.aliyuncs.com/dotbalo/redis:7.2.5
# kubectl create deploy redis -n basic-component-uat --image=registry.cn-beijing.aliyuncs.com/dotbalo/redis:7.2.5

# 创建 Service
# kubectl expose deploy redis --port 6379 -n basic-component-dev
# kubectl expose deploy redis --port 6379 -n basic-component-uat
```

访问测试：

```bash
# 创建一个专门用于测试的 Redis 客户端
# kubectl create deploy redis-cli --image=registry.cn-beijing.aliyuncs.com/dotbalo/redis:7.2.5

# 测试每个环境的 Redis 基础组件
# kubectl exec -ti redis-cli-57cc5fd584-hvxzq -- bash
root@redis-cli-57cc5fd584-hvxzq:/data# redis-cli -h redis.basic-component-dev
redis.basic-component-dev:6379> set a dev
OK

redis.basic-component-dev:6379> get a
"dev"

root@redis-cli-57cc5fd584-hvxzq:/data# redis-cli -h redis.basic-component-uat
redis.basic-component-uat:6379> set a uat
OK

redis.basic-component-uat:6379> get a
"uat"
```

创建项目的两个环境：

```bash
# kubectl create ns projecta-dev
# kubectl create ns projecta-uat
```

在每个项目的环境下，创建一个 externalName 类型的 Service，用于连接到不同环境的基础组件：

```yaml
# vim redis-externalname.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: projecta-dev
spec:
  type: ExternalName
  externalName: redis.basic-component-dev.svc.cluster.local
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: projecta-uat
spec:
  type: ExternalName
  externalName: redis.basic-component-uat.svc.cluster.local

# 创建两个service（两个service分别在两个namespace下）
# kubectl create -f redis-externalname.yaml
service/redis created
service/redis created
```

接下来在每个项目的环境下，创建两个 Redis 客户端，用于模拟需要链接 Redis 的应用程序：

```bash
# kubectl create deploy usercenter --image=registry.cn-beijing.aliyuncs.com/dotbalo/redis:7.2.5 -n projecta-dev
# kubectl create deploy usercenter --image=registry.cn-beijing.aliyuncs.com/dotbalo/redis:7.2.5 -n projecta-uat
```

测试每个环境下的 externalName：

```bash
# 开发环境
# kubectl get po -n projecta-dev
NAME READY STATUS RESTARTS AGE
usercenter-6685654cc4-pc6m9 1/1 Running 0 2m28s

# kubectl exec -ti usercenter-6685654cc4-pc6m9 -n projecta-dev -- bash
root@usercenter-6685654cc4-pc6m9:/data# redis-cli -h redis
redis:6379> get a
"dev"


# UAT 环境
# kubectl get po -n projecta-uat
NAME READY STATUS RESTARTS AGE
usercenter-6685654cc4-m9wrb 1/1 Running 0 3m19s

# kubectl exec -ti usercenter-6685654cc4-m9wrb -n projecta-uat -- bash
root@usercenter-6685654cc4-m9wrb:/data# redis-cli -h redis
redis:6379> get a
"uat"
```



## 使用 Service 代理 K8s 外部服务

使用场景：

- 希望在生产环境中使用某个固定的名称而非 IP 地址访问外部的中间件服务；
- 希望 Service 指向另一个 Namespace 中或其他集群中的服务；
- 正在将工作负载转移到 Kubernetes 集群，但是一部分服务仍运行在 Kubernetes 集群之外的 backend。

```yaml
# vim baidu-proxy-svc.yaml

# 因为外部服务（如你示例中的 140.205.94.189）不在 Kubernetes 集群内，没有 Pod 标签，无法通过选择器自动关联。
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc-external
  labels:
    app: nginx-svc-external
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  sessionAffinity: None
  type: ClusterIP
---
# 当 Service 没有 selector 时，Kubernetes 不会自动创建 Endpoints。此时需要手动创建与 Service 同名的 Endpoints 对象，显式指定外部服务的 IP 和端口，才能让 Service 知道要将流量转发到哪里。
apiVersion: v1
kind: Endpoints
metadata:
  name: nginx-svc-external
  labels:
    app: nginx-svc-external
subsets:
  - addresses:
      - ip: 140.205.94.189 # 输入你想代理的外部服务的IP
    ports:
      - name: http
        port: 80
        protocol: TCP

# 创建service
# kubectl create -f baidu-proxy-svc.yaml
```

> 注 意
>
> Endpoint IP 地址不能是 loopback（127.0.0.0/8）、link-local（169.254.0.0/16）或者 link-local 多播地址（224.0.0.0/24）。

访问没有 Selector 的 Service 与有 Selector 的 Service 的原理相同，通过 Service 名称即可访问，请求将被路由到用户定义的 Endpoints。



## 多端口 Service

有的程序可能会监听多个端口，Service 也支持同时代理多个端口。比如在 K8s 中部署一个RabbitMQ，它具有两个端口，5672 是程序连接用于数据交互的接口，15672 是 RabbitMQ 管理页面的端口。

首先在 K8s 上部署一个 RabbitMQ：

```bash
# kubectl create deploy rabbitmq --image=registry.cn-beijing.aliyuncs.com/dotbalo/rabbitmq:3-management
```

接下来可以创建一个 Service，把 5672 指向 Pod 的 5672,15672 指向 15672：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: rabbitmq
spec:
  selector:
    app: rabbitmq
  ports:
    - name: http
      protocol: TCP
      port: 15672
      targetPort: 15672
    - name: amqp
      protocol: TCP
      port: 5672
      targetPort: 5672
```

> 有用的知识
>
> RabbitMQ 是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件，提供生产者消费者模型）。RabbitMQ 服务器是用 Erlang 语言编写的，而集群和故障转移是构建在开放电信平台框架上的。



## 会话保持

K8s 的 Service 支持会话保持，但是目前仅支持基于客户端 IP 的会话保持：

```yaml
kind: Service
apiVersion: v1
metadata:
 name: nginx
spec:
 selector:
 app: nginx
 ports:
 - protocol: TCP
 port: 80
 targetPort: 80
 sessionAffinity: ClientIP # ClientIP：配置基于 IP 的会话保持， None：不开启会话保持
 sessionAffinityConfig: # 会话保持配置
 clientIP:
 timeoutSeconds: 10800 # 10800秒内固定将某一pod的流量转发给固定的客户端ip
```



## Headless Service

### 定义

Headless Service 是 Kubernetes 中一种特殊类型的 Service，它会直接暴露 Pod 的 IP 地址和DNS 记录给客户端，适用于有状态应用的服务发现和负载均衡以及需要直接访问 Pod IP 的应用场景。

Headless Service 不需要分配 ClusterIP，而是通过 DNS 记录直接返回 Pod 的 IP 地址，所以和普通 Service 最大的区别就是使用 nslookup 解析一个 Headless Service 返回的是 Pod IP， 而普通 Service 返回的是 Service 的 IP。



### 使用场景

1. 有状态应用的服务发现和负载均衡：有状态应用（如数据库、消息队列等）通常需要为每个 Pod 分配一个唯一的标识符（如 Pod 名称或 IP 地址），以便其他服务或其他节点可以连接到某个实例。Headless Service 可以满足这一需求，通过直接暴露 Pod 的 IP 地址和 DNS 记录，实现服务发现和负载均衡。
2. 需要直接访问 Pod IP 的应用：在某些情况下，客户端可能需要直接访问 Pod 的 IP 地址，而不需要通过 Service 的负载均衡机制，此时也可以通过 Headless Service 实现。
3. 分布式系统：在分布式系统中，各个节点之间需要直接通信，并且每个节点都有自己的身份和状态。Headless Service 可以为每个节点分配一个唯一的 DNS 实体名称，支持节点之间的直接交互和负载均衡



### 工作原理

当创建一个 Headless Service 时，Kubernetes 会执行以下操作：

1.创建 DNS 记录：为每个 Pod 创建一个 DNS 记录，该记录的名称基于 Service 名称、Pod名称和命名空间定义，格式为：

```bash
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

暴露 Pod IP：客户端可以通过查询 DNS 记录获取 Pod 的 IP 地址，并直接访问某个 Pod。

比如创建一个名为 my-headless-service 的 Headless Service，这个 Service 匹配了 app=my-app 标签的 Pod，该服务具有三个副本，每个副本的名字是 pod-0、pod-1 和 pod-2。此时可以通过如下 DNS 名字进行访问：

- pod-0.my-headless-service.default.svc.cluster.local
- pod-1.my-headless-service.default.svc.cluster.local
- pod-2.my-headless-service.default.svc.cluster.local



### Headless Service 使用

创建一个 StatefulSet 和 Headless Service：

```yaml
# vim sts-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx

# vim nginx-sts.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:stable
          ports:
            - containerPort: 80
              name: web
```

创建这两种资源

```bash
[root@k8s-master01 ~]# kubectl create -f sts-headless.yaml
service/nginx created

[root@k8s-master01 ~]# kubectl create -f nginx-sts.yaml
statefulset.apps/web created

# 查看创建结果
[root@k8s-master01 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   7d7h
nginx        ClusterIP   None         <none>        80/TCP    37m

[root@k8s-master01 ~]# kubectl get statefulsets
NAME   READY   AGE
web    2/2     40m
```

测试

```bash
# 查看statefulset两个pod的资源名以及他们的IP
[root@k8s-master01 ~]# kubectl -n default get pod -owide
NAME                          READY   STATUS    RESTARTS   AGE     IP               NODE         NOMINATED NODE   READINESS GATES
web-0                         1/1     Running   0          42m     192.168.85.201   k8s-node01   <none>           <none>
web-1                         1/1     Running   0          42m     192.168.58.208   k8s-node02   <none>           <none>

# 创建测试pod
[root@k8s-master01 ~]# kubectl create deploy cluster-test --image=registry.cn-beijing.aliyuncs.com/dotbalo/debug-tools -- sleep 3600
deployment.apps/cluster-test created

# 进入测试pod
[root@k8s-master01 ~]# kubectl exec -it cluster-test-578dfc54-cmmr9 -- bash
(10:29 cluster-test-578dfc54-cmmr9:/) 

# 分别解析statefulset的两个pod资源的dns记录
(10:29 cluster-test-578dfc54-cmmr9:/) nslookup web-0.nginx
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-0.nginx.default.svc.cluster.local
Address: 192.168.85.201

(10:31 cluster-test-578dfc54-cmmr9:/) nslookup web-1.nginx
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	web-1.nginx.default.svc.cluster.local
Address: 192.168.58.208

# 可见在headless service和pod中间没有service的IP这一中间层，是直接解析到具体pod的
```



## Service 代理模式

### Iptables 代理模式

Iptables 是 Linux 原生提供的一个功能强大的防火墙工具，可以用来设置、维护和检查 IPv4数据包，并且支持源/目地址转换等规则。在 iptables 代理模式下，kube-proxy 通过监听 Kubernetes API Server 中 Service 和 Endpoint 对象的变化，动态地更新节点上的 iptables 规则，以实现请求的转发。

**工作流程**：

1.当 Service 被创建或更新时，kube-proxy 会读取 Service 和 Endpoint 对象的信息，并生成相应的 iptables 规则

2.这些 iptables 规则被添加到内核的 netfilter 处理链中，以拦截和转发目标为 Service IP 地址的流量

3.当客户端访问 Service 的 IP 地址时，iptables 规则会将流量随机重定向到后端的一个或多个Pod

**优点与缺点**：

- 优点：iptables 是 Linux 内核的一部分，性能稳定、可靠，iptables 规则易于理解和维护，功能多。

- 缺点：随着 Service 数量的增加，iptables 规则的数量也会急剧增加，进而导致性能下降。iptables的更新操作可能会暂时锁定整个 iptables 规则表，影响网络性能。



### IPVS 代理模式

IPVS（IP Virtual Server）是一种基于内核的负载均衡器，提供了比 iptables 更高的转发性能。在 IPVS 代理模式下，kube-proxy 通过配置 IPVS 负载均衡器规则来代替使用 iptables。IPVS 使用更高效的数据结构（如 Hash 表）来存储和查找规则，可以在大量 Service 的情况下也能保持高性能。

**工作流程**：

1.当 Service 被创建或更新时，kube-proxy 会读取 Service 和 Endpoint 对象的信息，并配置IPVS 负载均衡策略

2.IPVS 负载均衡器会根据配置的调度算法（如轮询、最少连接等）将请求转发到后端的一个或多个 Pod 上

3.当客户端访问 Service 的 IP 地址时，请求会直接被 IPVS 处理并转发到后端 Pod

**优点与缺点**：

优点：IPVS 专为负载均衡设计，性能优于 iptables。并且支持多种调度算法，可以根据实际需求选择合适的算法，同时 IPVS 的更新操作对性能的影响较小

缺点：在某些情况下，IPVS 可能需要依赖 iptables 来实现一些额外的功能（如源地址 NAT）

**IPVS 负载均衡算法**：

- 轮询：rr，按顺序轮流将请求转发到后端的各个 Pod 上，实现请求的均匀分配

- 最少链接：lc，将新的请求转发到当前连接数最少的 Pod 上，以平衡各 Pod 的负载

- 源地址哈希：sh，根据请求的源 IP 地址进行哈希计算，将相同源地址的请求转发到同一个 Pod 上，实现会话保持

- 目的地址哈希：dh，根据请求的目的 IP 地址（即 Service 的 Cluster IP）和端口进行哈希计算，选择后端 Pod

- 无需队列等待：nq，如果后端 Pod 的队列为空，则直接选择该 Pod；如果所有 Pod 的队列都非空，则采用其他策略（如轮询或最少连接）来选择 Pod

- 最短期望延迟：sed，考虑 Pod 的当前连接数和连接请求的平均处理时间，选择预计处理时间最短的 Pod 来接收新请求

### **iptables与ipvs的区别**

**1. 规则的组织方式：线性链（Chains）**

`iptables` 的核心概念就是“链”（Chains），如 `PREROUTING`、`INPUT`、`FORWARD` 等。

- **逻辑上的链表**：当你为一个 Kubernetes Service 创建规则时，`kube-proxy` 会在自定义链中插入一系列规则。这些规则是**按顺序排列**的。
- **执行时的遍历**：当一个数据包进入内核网络栈时，它必须从链的**头部**开始，逐条匹配规则。
  - 如果第 1 条不匹配，看第 2 条；
  - 如果第 2 条不匹配，看第 3 条……
  - 直到找到匹配项或到达链尾。

**2. 算法复杂度：O(n) vs O(1)**

这是底层数据结构决定的性能差异：

- **iptables (O(n))**：

  由于是类似链表的线性扫描，匹配耗时与规则数量 $n$ 成正比。在 Kubernetes 这种动辄产生数千甚至上万条 Service 规则的环境下，数据包就像在玩一个“闯关游戏”，关卡越多，耗时越长。

- **IPVS / IPSET (O(1))**：

  相比之下，IPVS 使用的是**哈希表（Hash Table）**。无论你有 10 条还是 10,000 条规则，内核都能通过哈希计算直接“定位”到对应的处理动作，速度几乎是恒定的。

**3. 更新时的“全身而退”问题**

`iptables` 与链表关系的另一个体现是在**规则更新**上：

`iptables` 并不支持“增量更新”。在内核中，如果你想在 1000 条规则中间插入 1 条，它通常的做法是：

1. 把整个规则表（包含所有链和规则）从内核空间**拷贝**到用户空间。
2. 在用户空间进行修改（添加新规则）。
3. 将修改后的整个表重新**覆盖**回内核。

**结果**：当 Service 数量非常多时，每次 Pod 的上线或下线都会导致一次巨大的内核内存拷贝和计算压力，甚至引起短暂的网络抖动或 CPU 飙升。



## 更改 Service 代理模式

查看当前的代理模式：

```bash
# curl 127.0.0.1:10249/proxyMode
iptables
```

更改 proxy 的代理模式为 ipvs：

```bash
# kubectl edit cm kube-proxy -n kube-system （二进制安装方式配置文件在每个机器上）
...
    metricsBindAddress: ""
    mode: "ipvs"
    nftables:
...


# 重启 kube-proxy（二进制安装方式使用 systemctl restart kube-proxy）
# 两种方式
# 1.删除到所有pod，ds会自动创建新的
# 2.修改ds的配置，然后保存
kubectl edit ds kube-proxy -n kube-system
...
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-proxy
  template:
    metadata:
      # 这里随便加一个注释
      annotations:
        cause: "change proxy mode"
      creationTimestamp: null
      labels:
        k8s-app: kube-proxy
...
```

再次查看代理模式：

```bash
# curl 127.0.0.1:10249/proxyMode
ipvs
```

在机器上查看 ipvs 规则：

```bash
# yum install ipvsadm -y

# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
 -> RemoteAddress:Port Forward Weight ActiveConn InActConn
TCP 172.16.32.128:32000 rr
 -> 172.16.85.217:8443 Masq 1 0 0 
TCP 172.16.32.128:32001 rr
 -> 192.168.181.141:80 Masq 1 0 0
```

更改代理算法为最小连接数：

```bash
# kubectl edit cm kube-proxy -n kube-system
...
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: "lc"
      strictARP: false
...


# 重启 Proxy（自动添加注释）
# kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system
```

查看 ipvs 算法：

```bash
# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
 -> RemoteAddress:Port Forward Weight ActiveConn InActConn
TCP 172.16.32.128:32000 lc
 -> 172.16.85.217:8443 Masq 1 0 0 
TCP 172.16.32.128:32001 lc
 -> 192.168.181.141:80 Masq 1 0 0
```

