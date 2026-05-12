# k8s资源调度

# ReplicaSet 控制器

前面我们一起学习了 Pod 的原理和一些基本使用，但是在实际使用的时候并不会直接使用 Pod，而是会使用各种控制器来满足我们的需求，Kubernetes 中运行了一系列控制器来确保集群的当前状态与期望状态保持一致，它们就是 Kubernetes 的大脑。例如，ReplicaSet 控制器负责维护集群中运行的 Pod 数量；Node 控制器负责监控节点的状态，并在节点出现故障时及时做出响应。总而言之，在 Kubernetes 中，每个控制器只负责某种类型的特定资源。



## 控制器

Kubernetes 控制器会监听资源的 `创建/更新/删除` 事件，并触发 `Reconcile` 调谐函数作为响应，整个调整过程被称作 `“Reconcile Loop”（调谐循环）` 或者 `“Sync Loop”（同步循环）`。Reconcile 是一个使用资源对象的命名空间和资源对象名称来调用的函数，使得资源对象的实际状态与 资源清单中定义的状态保持一致。调用完成后，Reconcile 会将资源对象的状态更新为当前实际状态。我们可以用下面的一段伪代码来表示这个过程：

```go
for {
  desired := getDesiredState()  // 期望的状态
  current := getCurrentState()  // 当前实际状态
  if current == desired {  // 如果状态一致则什么都不做
    // nothing to do
  } else {  // 如果状态不一致则调整编排，到一致为止
    // change current to desired status
  }
}
```

这个编排模型就是 Kubernetes 项目中的一个通用编排模式，即：`控制循环（control loop）`。



## ReplicaSet

假如我们现在有一个 Pod 正在提供线上的服务，我们来想想一下我们可能会遇到的一些场景：

- 某次运营活动非常成功，网站访问量突然暴增
- 运行当前 Pod 的节点发生故障了，Pod 不能正常提供服务了

第一种情况，可能比较好应对，活动之前我们可以大概计算下会有多大的访问量，提前多启动几个 Pod 副本，活动结束后再把多余的 Pod 杀掉，虽然有点麻烦，但是还是能够应对这种情况的。

第二种情况，可能某天夜里收到大量报警说服务挂了，然后起来打开电脑在另外的节点上重新启动一个新的 Pod，问题可以解决。

但是如果我们都人工的去解决遇到的这些问题，似乎又回到了以前刀耕火种的时代了是吧？如果有一种工具能够来帮助我们自动管理 Pod 就好了，Pod 挂了自动帮我在合适的节点上重新启动一个 Pod，这样是不是遇到上面的问题我们都不需要手动去解决了。

而 ReplicaSet 这种资源对象就可以来帮助我们实现这个功能，`ReplicaSet（RS）` 的主要作用就是维持一组 Pod 副本的运行，保证一定数量的 Pod 在集群中正常运行，ReplicaSet 控制器会持续监听它所控制的这些 Pod 的运行状态，在 Pod 发送故障数量减少或者增加时会触发调谐过程，始终保持副本数量一定。

和 Pod 一样我们仍然还是通过 YAML 文件来描述我们的 ReplicaSet 资源对象，如下 YAML 文件是一个常见的 ReplicaSet 定义：

```yaml
# nginx-rs.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
  namespace: default
spec:
  replicas: 3 # 期望的 Pod 副本数量，默认值为1
  selector: # Label Selector，必须匹配 Pod 模板中的标签
    matchLabels:
      app: nginx
  template: # Pod 模板
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

上面的 YAML 文件结构和我们之前定义的 Pod 看上去没太大两样，有常见的 apiVersion、kind、metadata，在 spec 下面描述 ReplicaSet 的基本信息，其中包含 3 个重要内容：

- replicas：表示期望的 Pod 的副本数量
- selector：Label Selector，用来匹配要控制的 Pod 标签，需要和下面的 Pod 模板中的标签一致
- template：Pod 模板，实际上就是以前我们定义的 Pod 内容，相当于把一个 Pod 的描述以模板的形式嵌入到了 ReplicaSet 中来。

Pod 模板

Pod 模板这个概念非常重要，因为后面我们讲解到的大多数控制器，都会使用 Pod 模板来统一定义它所要管理的 Pod。更有意思的是，我们还会看到其他类型的对象模板，比如 Volume 的模板等。

上面就是我们定义的一个普通的 ReplicaSet 资源清单文件，ReplicaSet 控制器会通过定义的 Label Selector 标签去查找集群中的 Pod 对象：

![replicaset top](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/1662426531708.jpg)

我们直接来创建上面的资源对象：

```shell
➜  ~ kubectl apply -f nginx-rs.yaml
replicaset.apps/nginx-rs created

➜  ~ kubectl get rs nginx-rs
NAME       DESIRED   CURRENT   READY   AGE
nginx-rs   3         3         3       17m
```

通过查看 RS 可以看到当前资源对象的描述信息，包括`DESIRED`、`CURRENT`、`READY`的状态值，创建完成后，可以利用如下命令查看下 Pod 列表：

```shell
➜  ~ kubectl get pods -l app=nginx
NAME             READY   STATUS              RESTARTS   AGE
nginx-rs-nxklf   1/1     Running   0          52s
nginx-rs-t46qc   1/1     Running   0          52s
nginx-rs-xfqrn   1/1     Running   0          52s
```

可以看到现在有 3 个 Pod，这 3 个 Pod 就是我们在 RS 中声明的 3 个副本，比如我们删除其中一个 Pod：

```shell
➜  ~ kubectl delete pod nginx-rs-xfqrn
pod "nginx-rs-xfqrn" deleted
```

然后再查看 Pod 列表：

```shell
➜  ~ kubectl get pods -l app=nginx
NAME             READY   STATUS    RESTARTS   AGE
nginx-rs-nxklf   1/1     Running   0          3m19s
nginx-rs-t46qc   1/1     Running   0          3m19s
nginx-rs-xsb59   1/1     Running   0          10s
```

可以看到又重新出现了一个 Pod，这个就是上面我们所说的 ReplicaSet 控制器为我们做的工作，我们在 YAML 文件中声明了 3 个副本，然后现在我们删除了一个副本，就变成了两个，这个时候 ReplicaSet 控制器监控到控制的 Pod 数量和期望的 3 不一致，所以就需要启动一个新的 Pod 来保持 3 个副本，这个过程上面我们说了就是`调谐`的过程。同样可以查看 RS 的描述信息来查看到相关的事件信息：

```bash
➜  ~ kubectl describe rs nginx-rs
Name:         nginx-rs
Namespace:    default
Selector:     app=nginx
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{},"name":"nginx-rs","namespace":"default"},"spec":{"replicas":3,"se...
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: nginx-rs-xfqrn
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: nginx-rs-nxklf
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: nginx-rs-t46qc
  Normal  SuccessfulCreate  14m   replicaset-controller  Created pod: nginx-rs-xsb59
```

可以发现最开始通过 ReplicaSet 控制器创建了 3 个 Pod，后面我们删除了 Pod 后， ReplicaSet 控制器又为我们创建了一个 Pod，和上面我们的描述是一致的。如果这个时候我们把 RS 资源对象的 Pod 副本更改为 2 `spec.replicas=2`，这个时候我们来更新下资源对象：

```shell
➜  ~ kubectl apply -f rs.yaml
replicaset.apps/nginx-rs configured

➜  ~ kubectl get rs nginx-rs
NAME       DESIRED   CURRENT   READY   AGE
nginx-rs   2         2         2       27m

$ kubectl describe rs nginx-rs
Name:         nginx-rs
Namespace:    default
Selector:     app=nginx
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{},"name":"nginx-rs","namespace":"default"},"spec":{"replicas":2,"se...
Replicas:     2 current / 2 desired
Pods Status:  2 Running / 1 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  27m   replicaset-controller  Created pod: nginx-rs-xfqrn
  Normal  SuccessfulCreate  27m   replicaset-controller  Created pod: nginx-rs-nxklf
  Normal  SuccessfulCreate  27m   replicaset-controller  Created pod: nginx-rs-t46qc
  Normal  SuccessfulCreate  24m   replicaset-controller  Created pod: nginx-rs-xsb59
  Normal  SuccessfulDelete  7s    replicaset-controller  Deleted pod: nginx-rs-xsb59
```

可以看到 Replicaset 控制器在发现我们的资源声明中副本数变更为 2 后，就主动去删除了一个 Pod，这样副本数就和期望的始终保持一致了：

```shell
➜  ~ kubectl get pods -l app=nginx
NAME             READY   STATUS    RESTARTS   AGE
nginx-rs-nxklf   1/1     Running   0          30m
nginx-rs-t46qc   1/1     Running   0          30m
```

我们可以随便查看一个 Pod 的描述信息可以看到这个 Pod 的所属控制器信息：

```shell
➜  ~ kubectl describe pod nginx-rs-xsb59
Name:               nginx-rs-xsb59
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               ydzs-node1/10.151.30.22
Start Time:         Fri, 15 Nov 2019 14:18:10 +0800
Labels:             app=nginx
Annotations:        <none>
Status:             Running
IP:                 10.244.1.148
Controlled By:      ReplicaSet/nginx-rs
.......
```

另外被 ReplicaSet 持有的 Pod 有一个 `metadata.ownerReferences` 指针指向当前的 ReplicaSet，表示当前 Pod 的所有者，这个引用主要会被集群中的**垃圾收集器**使用以清理失去所有者的 Pod 对象。这个 `ownerReferences` 和数据库中的`外键`是不是非常类似。可以通过将 Pod 资源描述信息导出查看：

```shell
➜  ~ kubectl get pod nginx-rs-xsb59 -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2021-11-03T06:18:10Z"
  generateName: nginx-rs-
  labels:
    app: nginx
  name: nginx-rs-xsb59
  namespace: default
  ownerReferences: # 
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: nginx-rs
    uid: 4a3121fa-b5ae-4def-b2d2-bf17bc06b7b7
  resourceVersion: "1781596"
  selfLink: /api/v1/namespaces/default/pods/nginx-rs-xsb59
  uid: 0a4cae9a-105b-4024-ae96-ee516bfb2d23
......
```

我们可以看到 Pod 中有一个 `metadata.ownerReferences` 的字段指向了 ReplicaSet 资源对象。如果要彻底删除 Pod，我们就只能删除 RS 对象：

```shell
➜  ~ kubectl delete rs nginx-rs
# 或者执行 kubectl delete -f nginx-rs.yaml
```

这就是 ReplicaSet 对象的基本使用。



## Replication Controller（可忽略）

Replication Controller 简称 RC，实际上 RC 和 RS 的功能几乎一致，RS 算是对 RC 的改进，目前唯一的一个区别就是 RC 只支持基于等式的 selector（env=dev 或 environment != qa），但 RS 还支持基于集合的 selector（version in (v1.0, v2.0)），这对复杂的运维管理就非常方便了。

比如上面资源对象如果我们要使用 RC 的话，对应的 selector 是这样的：

```yaml
selector:
  app: nginx
```

RC 只支持单个 Label 的等式，而 RS 中的 Label Selector 支持 `matchLabels` 和 `matchExpressions` 两种形式：

```yaml
selector:
  matchLabels:
    app: nginx
# 或
selector:
  matchExpressions: # 该选择器要求 Pod 包含名为 app 的标签
    - key: app
      operator: In
      values: # 并且标签的值必须是 nginx
        - nginx
```

总的来说 RS 是新一代的 RC，所以以后我们不使用 RC，直接使用 RS 即可，他们的功能都是一致的，但是实际上在实际使用中我们也不会直接使用 RS，而是使用更上层的类似于 Deployment 这样的资源对象。



# 无状态应用管理 Deployment

前面我们学习了 ReplicaSet 控制器，了解到该控制器是用来维护集群中运行的 Pod 数量的，但是往往在实际操作的时候，我们反而不会去直接使用 RS，而是会使用更上层的控制器，比如我们今天要学习的主角 Deployment，Deployment 一个非常重要的功能就是实现了 Pod 的滚动更新，比如我们应用更新了，我们只需要更新我们的容器镜像，然后修改 Deployment 里面的 Pod 模板镜像，那么 Deployment 就会用**滚动更新（Rolling Update）**的方式来升级现在的 Pod，这个能力是非常重要的，因为对于线上的服务我们需要做到不中断服务，所以滚动更新就成了必须的一个功能。而 Deployment 这个能力的实现，依赖的就是上节课我们学习的 ReplicaSet 这个资源对象，实际上我们可以通俗的理解就是**每个 Deployment 就对应集群中的一次部署**，这样就更好理解了。

## Deployment 概述

使用命令生成一个现成的 Deployment 的 Yaml：

```bash
kubectl create deploy nginx --replicas=3 --image=registry.cn-beijing.aliyuncs.com/dotbalo/nginx:stable --dry-run=client -oyaml > 
deploy.yaml
```

Deployment 资源对象的格式和 ReplicaSet 几乎一致，如下资源对象就是一个常见的 Deployment 资源类型：

```yaml
# nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: default
spec:
  replicas: 3 # 期望的 Pod 副本数量，默认值为1
  selector: # Label Selector，必须匹配 Pod 模板中的标签
    matchLabels:
      app: nginx
  template: # Pod 模板
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

我们这里只是将类型替换成了 Deployment，我们可以先来创建下这个资源对象：

```shell
➜  ~ kubectl apply -f nginx-deploy.yaml
deployment.apps/nginx-deploy created

➜  ~ kubectl get deployment
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           58s
```

创建完成后，查看 Pod 状态：

```shell
➜  ~ kubectl get pods -l app=nginx
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-85ff79dd56-7r76h   1/1     Running   0          41s
nginx-deploy-85ff79dd56-d5gjs   1/1     Running   0          41s
nginx-deploy-85ff79dd56-txc4h   1/1     Running   0          41s
```

到这里我们发现和之前的 RS 对象是否没有什么两样，都是根据`spec.replicas`来维持的副本数量，我们随意查看一个 Pod 的描述信息：

```shell
➜  ~ kubectl describe pod nginx-deploy-85ff79dd56-txc4h
Name:               nginx-deploy-85ff79dd56-txc4h
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               node1/10.151.30.22
Start Time:         Sat, 16 Nov 2019 16:01:25 +0800
Labels:             app=nginx
                    pod-template-hash=85ff79dd56
Annotations:        podpreset.admission.kubernetes.io/podpreset-time-preset: 2062768
Status:             Running
IP:                 10.244.1.166
Controlled By:      ReplicaSet/nginx-deploy-85ff79dd56
......
Events:
  Type    Reason     Age        From                 Message
  ----    ------     ----       ----                 -------
  Normal  Scheduled  <unknown>  default-scheduler    Successfully assigned default/nginx-deploy-85ff79dd56-txc4h to node1
  Normal  Pulling    2m         kubelet, node1  Pulling image "nginx"
  Normal  Pulled     117s       kubelet, node1  Successfully pulled image "nginx"
  Normal  Created    117s       kubelet, node1  Created container nginx
  Normal  Started    116s       kubelet, node1  Started container nginx
```

我们仔细查看其中有这样一个信息 `Controlled By: ReplicaSet/nginx-deploy-85ff79dd56`，什么意思？是不是表示当前我们这个 Pod 的控制器是一个 ReplicaSet 对象啊，我们不是创建的一个 Deployment 吗？为什么 Pod 会被 RS 所控制呢？那我们再去看下这个对应的 RS 对象的详细信息如何呢：

```shell
➜  ~ kubectl describe rs nginx-deploy-85ff79dd56
Name:           nginx-deploy-85ff79dd56 
Namespace:      default
Selector:       app=nginx,pod-template-hash=85ff79dd56
Labels:         app=nginx
                pod-template-hash=85ff79dd56
Annotations:    deployment.kubernetes.io/desired-replicas: 3
                deployment.kubernetes.io/max-replicas: 4
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/nginx-deploy
Replicas:       3 current / 3 desired
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
......
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  4m52s  replicaset-controller  Created pod: nginx-deploy-85ff79dd56-7r76h
  Normal  SuccessfulCreate  4m52s  replicaset-controller  Created pod: nginx-deploy-85ff79dd56-d5gjs
  Normal  SuccessfulCreate  4m52s  replicaset-controller  Created pod: nginx-deploy-85ff79dd56-txc4h
```

其中有这样的一个信息：`Controlled By: Deployment/nginx-deploy`，明白了吧？意思就是我们的 Pod 依赖的控制器 RS 实际上被我们的 Deployment 控制着呢，我们可以用下图来说明 Pod、ReplicaSet、Deployment 三者之间的关系：

![Deployment](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/1662426664683.jpg)

通过上图我们可以很清楚的看到，定义了 3 个副本的 Deployment 与 ReplicaSet 和 Pod 的关系，就是一层一层进行控制的。ReplicaSet 作用和之前一样还是来保证 Pod 的个数始终保存指定的数量，所以 Deployment 中的容器 `restartPolicy=Always` 是唯一的就是这个原因，因为容器必须始终保证自己处于 Running 状态，ReplicaSet 才可以去明确调整 Pod 的个数。而 Deployment 是通过管理 ReplicaSet 的数量和属性来实现`水平扩展/收缩`以及`滚动更新`两个功能的。



## 水平伸缩

`水平扩展/收缩`的功能比较简单，因为 ReplicaSet 就可以实现，所以 Deployment 控制器只需要去修改它所控制的 ReplicaSet 的 Pod 副本数量就可以了。比如现在我们把 Pod 的副本调整到 4 个，那么 Deployment 所对应的 ReplicaSet 就会自动创建一个新的 Pod 出来，这样就水平扩展了，我们可以使用一个新的命令 `kubectl scale` 命令来完成这个操作：

```shell
➜  ~ kubectl scale deployment nginx-deploy --replicas=4
deployment.apps/nginx-deployment scaled
```

扩展完成后可以查看当前的 RS 对象：

```shell
➜  ~ kubectl get rs
NAME                      DESIRED   CURRENT   READY   AGE
nginx-deploy-85ff79dd56   4         4         3       40m
```

可以看到期望的 Pod 数量已经变成 4 了，只是 Pod 还没准备完成，所以 READY 状态数量还是 3，同样查看 RS 的详细信息：

```shell
➜  ~ kubectl describe rs nginx-deploy-85ff79dd56
Name:           nginx-deploy-85ff79dd56
Namespace:      default
Selector:       app=nginx,pod-template-hash=85ff79dd56
......
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  40m   replicaset-controller  Created pod: nginx-deploy-85ff79dd56-7r76h
  Normal  SuccessfulCreate  40m   replicaset-controller  Created pod: nginx-deploy-85ff79dd56-d5gjs
  Normal  SuccessfulCreate  40m   replicaset-controller  Created pod: nginx-deploy-85ff79dd56-txc4h
  Normal  SuccessfulCreate  17s   replicaset-controller  Created pod: nginx-deploy-85ff79dd56-tph9g
```

可以看到 ReplicaSet 控制器增加了一个新的 Pod，同样的 Deployment 资源对象的事件中也可以看到完成了扩容的操作：

```shell
➜  ~ kubectl describe deploy nginx-deploy
Name:                   nginx-deploy
Namespace:              default
......
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deploy-85ff79dd56 (4/4 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  43m    deployment-controller  Scaled up replica set nginx-deploy-85ff79dd56 to 3
  Normal  ScalingReplicaSet  3m16s  deployment-controller  Scaled up replica set nginx-deploy-85ff79dd56 to 4
```



## 滚动更新

如果只是`水平扩展/收缩`这两个功能，就完全没必要设计 Deployment 这个资源对象了，Deployment 最突出的一个功能是支持`滚动更新`，比如现在我们需要把应用容器更改为 `nginx:1.7.9` 版本，修改后的资源清单文件如下所示：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: default
spec:
  replicas: 3 # 注意这里的副本数为3，刚才临时设置为了4
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  strategy:
    type: RollingUpdate # 指定更新策略：RollingUpdate和Recreate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
```

前后相比较，除了更改了镜像之外，我们还指定了更新策略：

```yaml
minReadySeconds: 5
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

- `minReadySeconds`：表示 Kubernetes 在等待设置的时间后才进行升级，如果没有设置该值，Kubernetes 会假设该容器启动起来后就提供服务了，如果没有设置该值，在某些极端情况下可能会造成服务不正常运行，默认值就是 0。
- `type=RollingUpdate`：表示设置更新策略为滚动更新，只有`Recreate`和`RollingUpdate`两个值，`Recreate`表示全部重新创建，默认值就是`RollingUpdate`。
- `maxSurge`：表示升级过程中最多可以比原先设置多出的 Pod 数量，例如：`maxSurage=1，replicas=5`，就表示 Kubernetes 会先启动一个新的 Pod，然后才删掉一个旧的 Pod，整个升级过程中最多会有`5+1`个 Pod。
- `maxUnavaible`：表示升级过程中最多有多少个 Pod 处于无法提供服务的状态，当`maxSurge`不为 0 时，该值也不能为 0，例如：`maxUnavaible=1`，则表示 Kubernetes 整个升级过程中最多会有 1 个 Pod 处于无法服务的状态。

现在我们来直接更新上面的 Deployment 资源对象：

```shell
kubectl apply -f nginx-deploy.yaml
```

record 参数

我们可以添加一个额外的 `--record` 参数来记录下我们的每次操作所执行的命令，以方便后面查看。

更新后，我们可以执行下面的 `kubectl rollout status` 命令来查看我们此次滚动更新的状态：

```shell
➜  ~ kubectl rollout status deployment/nginx-deploy
Waiting for deployment "nginx-deploy" rollout to finish: 2 out of 3 new replicas have been updated...
```

从上面的信息可以看出我们的滚动更新已经有两个 Pod 已经更新完成了，在滚动更新过程中，我们还可以执行如下的命令来暂停更新：

```shell
➜  ~ kubectl rollout pause deployment/nginx-deploy
deployment.apps/nginx-deploy paused
```

这个时候我们的滚动更新就暂停了，此时我们可以查看下 Deployment 的详细信息：

```shell
➜  ~ kubectl describe deploy nginx-deploy
Name:                   nginx-deploy
Namespace:              default
CreationTimestamp:      Sat, 16 Nov 2019 16:01:24 +0800
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deploy","namespace":"default"},"spec":{"minReadySec...
Selector:               app=nginx
Replicas:               3 desired | 2 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        5
RollingUpdateStrategy:  1 max unavailable, 1 max surge
......
OldReplicaSets:  nginx-deploy-85ff79dd56 (2/2 replicas created)
NewReplicaSet:   nginx-deploy-5b7b9ccb95 (2/2 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  26m    deployment-controller  Scaled up replica set nginx-deploy-85ff79dd56 to 4
  Normal  ScalingReplicaSet  3m44s  deployment-controller  Scaled down replica set nginx-deploy-85ff79dd56 to 3
  Normal  ScalingReplicaSet  3m44s  deployment-controller  Scaled up replica set nginx-deploy-5b7b9ccb95 to 1
  Normal  ScalingReplicaSet  3m44s  deployment-controller  Scaled down replica set nginx-deploy-85ff79dd56 to 2
  Normal  ScalingReplicaSet  3m44s  deployment-controller  Scaled up replica set nginx-deploy-5b7b9ccb95 to 2
```

![Deployment RollingUpdate](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/1662426687515.jpg)

我们仔细观察 Events 事件区域的变化，上面我们用 `kubectl scale` 命令将 Pod 副本调整到了 4，现在我们更新的时候是不是声明又变成 3 了，所以 Deployment 控制器首先是将之前控制的 `nginx-deploy-85ff79dd56` 这个 RS 资源对象进行缩容操作，然后滚动更新开始了，可以发现 Deployment 为一个新的 `nginx-deploy-5b7b9ccb95` RS 资源对象首先新建了一个新的 Pod，然后将之前的 RS 对象缩容到 2 了，再然后新的 RS 对象扩容到 2，后面由于我们暂停滚动升级了，所以没有后续的事件了，大家有看明白这个过程吧？这个过程就是滚动更新的过程，启动一个新的 Pod，杀掉一个旧的 Pod，然后再启动一个新的 Pod，这样滚动更新下去，直到全都变成新的 Pod，这个时候系统中应该存在 4 个 Pod，因为我们设置的策略`maxSurge=1`，所以在升级过程中是允许的，而且是两个新的 Pod，两个旧的 Pod：

```shell
➜  ~ kubectl get pods -l app=nginx
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-5b7b9ccb95-k6pkh   1/1     Running   0          11m
nginx-deploy-5b7b9ccb95-l6lmx   1/1     Running   0          11m
nginx-deploy-85ff79dd56-7r76h   1/1     Running   0          75m
nginx-deploy-85ff79dd56-txc4h   1/1     Running   0          75m
```

查看 Deployment 的状态也可以看到当前的 Pod 状态：

```shell
➜  ~ kubectl get deployment
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   4/3     2            4           75m
```

这个时候我们可以使用`kubectl rollout resume`来恢复我们的滚动更新：

```shell
➜  ~ kubectl rollout resume deployment/nginx-deploy
deployment.apps/nginx-deploy resumed

➜  ~ kubectl rollout status deployment/nginx-deploy
Waiting for deployment "nginx-deploy" rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx-deploy" successfully rolled out
```

看到上面的信息证明我们的滚动更新已经成功了，同样可以查看下资源状态：

```shell
➜  ~ kubectl get pod -l app=nginx
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-5b7b9ccb95-gmq7v   1/1     Running   0          115s
nginx-deploy-5b7b9ccb95-k6pkh   1/1     Running   0          15m
nginx-deploy-5b7b9ccb95-l6lmx   1/1     Running   0          15m

➜  ~ kubectl get deployment
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           79m
```



## 版本回滚

这个时候我们查看 ReplicaSet 对象，可以发现会出现两个：

```shell
➜  ~ kubectl get rs -l app=nginx
NAME                      DESIRED   CURRENT   READY   AGE
nginx-deploy-5b7b9ccb95   3         3         3       18m
nginx-deploy-85ff79dd56   0         0         0       81m
```

从上面可以看出滚动更新之前我们使用的 RS 资源对象的 Pod 副本数已经变成 0 了，而滚动更新后的 RS 资源对象变成了 3 个副本，我们可以导出之前的 RS 对象查看：

```yaml
➜  ~ kubectl get rs nginx-deploy-85ff79dd56 -o yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    deployment.kubernetes.io/desired-replicas: "3"
    deployment.kubernetes.io/max-replicas: "4"
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2019-11-16T08:01:24Z"
  generation: 5
  labels:
    app: nginx
    pod-template-hash: 85ff79dd56
  name: nginx-deploy-85ff79dd56
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: nginx-deploy
    uid: b0fc5614-ef58-496c-9111-740353bd90d4
  resourceVersion: "2140545"
  selfLink: /apis/apps/v1/namespaces/default/replicasets/nginx-deploy-85ff79dd56
  uid: 8eca2998-3610-4f80-9c21-5482ba579892
spec:
  replicas: 0
  selector:
    matchLabels:
      app: nginx
      pod-template-hash: 85ff79dd56
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
        pod-template-hash: 85ff79dd56
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  observedGeneration: 5
  replicas: 0
```

我们仔细观察这个资源对象里面的描述信息除了副本数变成了 `replicas=0` 之外，和更新之前没有什么区别吧？大家看到这里想到了什么？有了这个 RS 的记录存在，是不是我们就可以回滚了啊？而且还可以回滚到前面的任意一个版本，这个版本是如何定义的呢？我们可以通过命令 `rollout history` 来获取：

```shell
➜  ~ kubectl rollout history deployment nginx-deploy
deployment.apps/nginx-deploy
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

其实 `rollout history` 中记录的 `revision` 是和 `ReplicaSets` 一一对应。如果我们手动删除某个 `ReplicaSet`，对应的`rollout history`就会被删除，也就是说你无法回滚到这个`revison`了，同样我们还可以查看一个`revison`的详细信息：

```shell
➜  ~ kubectl rollout history deployment nginx-deploy --revision=1
deployment.apps/nginx-deploy with revision #1
Pod Template:
  Labels:       app=nginx
        pod-template-hash=85ff79dd56
  Containers:
   nginx:
    Image:      nginx
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

假如现在要直接回退到当前版本的前一个版本，我们可以直接使用如下命令进行操作：

```shell
➜  ~ kubectl rollout undo deployment nginx-deploy
```

当然也可以回退到指定的`revision`版本：

```shell
➜  ~ kubectl rollout undo deployment nginx-deploy --to-revision=1
deployment "nginx-deploy" rolled back
```

回滚的过程中我们同样可以查看回滚状态：

```shell
➜  ~ kubectl rollout status deployment/nginx-deploy
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deploy" rollout to finish: 2 of 3 updated replicas are available...
Waiting for deployment "nginx-deploy" rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx-deploy" successfully rolled out
```

这个时候查看对应的 RS 资源对象可以看到 Pod 副本已经回到之前的 RS 里面去了。

```shell
➜  ~ kubectl get rs -l app=nginx
NAME                      DESIRED   CURRENT   READY   AGE
nginx-deploy-5b7b9ccb95   0         0         0       31m
nginx-deploy-85ff79dd56   3         3         3       95m
```

不过需要注意的是回滚的操作滚动的`revision`始终是递增的：

```shell
➜  ~ kubectl rollout history deployment nginx-deploy
deployment.apps/nginx-deploy
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```



## 历史版本清理策略

在很早之前的 Kubernetes 版本中，默认情况下会为我们暴露下所有滚动升级的历史记录，也就是 ReplicaSet 对象，但一般情况下没必要保留所有的版本，毕竟会存在 etcd 中，我们可以通过配置 `deployment.spec.revisionHistoryLimit` 属性来设置保留的历史记录数量，新版本中该值默认为 10，如果希望多保存几个版本可以设置该字段。



# 有状态应用管理 StatefulSet

前面我们学习了 Deployment 和 ReplicaSet 两种资源对象得使用，在实际使用的过程中，Deployment 并不能编排所有类型的应用，对**无状态服务**编排是非常容易的，但是对于**有状态服务**就无能为力了。我们需要先明白一个概念：什么是有状态服务，什么是无状态服务。

- `无状态服务（Stateless Service）`：该服务运行的实例不会在本地存储需要持久化的数据，并且多个实例对于同一个请求响应的结果是完全一致的，比如前面我们讲解的 WordPress 实例，我们是不是可以同时启动多个实例，但是我们访问任意一个实例得到的结果都是一样的吧？因为他唯一需要持久化的数据是存储在 MySQL 数据库中的，所以我们可以说 WordPress 这个应用是无状态服务，但是 MySQL 数据库就不是了，因为他需要把数据持久化到本地。
- `有状态服务（Stateful Service）`：就和上面的概念是对立的了，该服务运行的实例需要在本地存储持久化数据，比如上面的 MySQL 数据库，你现在运行在节点 A，那么他的数据就存储在节点 A 上面的，如果这个时候你把该服务迁移到节点 B 去的话，那么就没有之前的数据了，因为他需要去对应的数据目录里面恢复数据，而此时没有任何数据。

现在对`有状态`和`无状态`有一定的认识了吧，比如我们常见的 WEB 应用，是通过 Session 来保持用户的登录状态的，如果我们将 Session 持久化到节点上，那么该应用就是一个有状态的服务了，因为我现在登录进来你把我的 Session 持久化到节点 A 上了，下次我登录的时候可能会将请求路由到节点 B 上去了，但是节点 B 上根本就没有我当前的 Session 数据，就会被认为是未登录状态了，这样就导致我前后两次请求得到的结果不一致了。所以一般为了横向扩展，我们都会把这类 WEB 应用改成无状态的服务，怎么改？将 Session 数据存入一个公共的地方，比如 Redis 里面，是不是就可以了，对于一些客户端请求 API 的情况，我们就不使用 Session 来保持用户状态，改成用 Token 也是可以的。

无状态服务利用我们前面的 Deployment 可以很好的进行编排，对应有状态服务，需要考虑的细节就要多很多了，容器化应用程序最困难的任务之一，就是设计有状态分布式组件的部署体系结构。由于无状态组件没有预定义的启动顺序、集群要求、点对点 TCP 连接、唯一的网络标识符、正常的启动和终止要求等，因此可以很容易地进行容器化。诸如数据库，大数据分析系统，分布式 key/value 存储、消息中间件需要有复杂的分布式体系结构，都可能会用到上述功能。为此，Kubernetes 引入了 `StatefulSet` 这种资源对象来支持这种复杂的需求。`StatefulSet` 类似于 `ReplicaSet`，但是它可以处理 Pod 的启动顺序，为保留每个 Pod 的状态设置唯一标识，具有以下几个功能特性：

- 稳定的、唯一的网络标识符
- 稳定的、持久化的存储
- 有序的、优雅的部署和缩放
- 有序的、优雅的删除和终止
- 有序的、自动滚动更新

## Headless Service

在我们学习 StatefulSet 对象之前，我们还必须了解一个新的概念：`Headless Service`。`Service` 其实在之前我们和大家提到过，Service 是应用服务的抽象，通过 Labels 为应用提供负载均衡和服务发现，每个 Service 都会自动分配一个 cluster IP 和 DNS 名，在集群内部我们可以通过该地址或者通过 FDQN 的形式来访问服务。比如，一个 Deployment 有 3 个 Pod，那么我就可以定义一个 Service，有如下两种方式来访问这个 Service：

- cluster IP 的方式，比如：当我访问 10.109.169.155 这个 Service 的 IP 地址时，10.109.169.155 其实就是一个 VIP，它会把请求转发到该 Service 所代理的 Endpoints 列表中的某一个 Pod 上。具体原理我们会在后面的 Service 章节中和大家深入了解。
- Service 的 DNS 方式，比如我们访问`“mysvc.mynamespace.svc.cluster.local”`这条 DNS 记录，就可以访问到 mynamespace 这个命名空间下面名为 mysvc 的 Service 所代理的某一个 Pod。

对于 DNS 这种方式实际上也有两种情况：

- 第一种就是普通的 Service，我们访问`“mysvc.mynamespace.svc.cluster.local”`的时候是通过集群中的 DNS 服务解析到的 mysvc 这个 Service 的 cluster IP 的，**普通 Service 并不提供 Pod 级别的 DNS 记录**，`podname.mysvc.mynamespace.svc.cluster.local` 这种格式，在**普通 Service（ClusterIP）** 下是**不存在**的。
- 第二种情况就是`Headless Service`，对于这种情况，我们访问`“mysvc.mynamespace.svc.cluster.local”`的时候是直接解析到的 mysvc 代理的某一个具体的 Pod 的 IP 地址，中间少了 cluster IP 的转发，这就是二者的最大区别，**Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 的记录方式解析到后面的 Pod 的 IP 地址。**

比如我们定义一个如下的 `Headless Service`：(headless-svc.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
  labels:
    app: nginx
spec:
  ports:
    - name: http
      port: 80
  clusterIP: None
  selector:
    app: nginx
```

实际上 `Headless Service` 在定义上和普通的 Service 几乎一致, 只是他的 `clusterIP=None`，所以，这个 Service 被创建后并不会被分配一个 cluster IP，而是会以 DNS 记录的方式暴露出它所代理的 Pod，而且还有一个非常重要的特性，对于 `Headless Service` 所代理的所有 Pod 的 IP 地址都会绑定一个如下所示的 DNS 记录：

```shell
# .cluster.local为集群域（Cluster Domin）
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

这个 DNS 记录正是 Kubernetes 集群为 Pod 分配的一个唯一标识，只要我们知道 Pod 的名字，以及它对应的 Service 名字，就可以组装出这样一条 DNS 记录访问到 Pod 的 IP 地址，这个能力是非常重要的，接下来我们就来看下 StatefulSet 资源对象是如何结合 Headless Service 提供服务的。



### 为什么StatefulSet要搭配Headless Service？

**1.StatefulSet 的核心特性：稳定的身份标识**

StatefulSet 管理的 Pod 具有 **固定且唯一的身份标识**，包括：

- **固定名称**：Pod 名称格式为 `<statefulset-name>-<序号>`（如 `mysql-0`、`mysql-1`），重启或重建后名称不变。
- **稳定网络标识**：需要通过 DNS 始终能访问到同一实例（即使 Pod 重建、IP 变化）。

例如，MySQL 主从集群中，从库需要始终连接到 `mysql-0`（主库），即使 `mysql-0` 重建后 IP 改变，也必须通过固定标识访问。

**2.Headless Service 解决：稳定的 DNS 解析**

普通 Service（ClusterIP 类型）会分配一个虚拟 IP，并通过该 IP 实现对后端 Pod 的负载均衡。但这种模式无法满足 StatefulSet 的需求：

- 普通 Service 无法直接访问某个具体的 Pod 实例（只能通过虚拟 IP 负载均衡到任意实例）。
- 若 Pod 重建，IP 会变化，普通 Service 无法保证 “名称→IP” 的映射稳定。

而 **Headless Service（`clusterIP: None`）** 不分配虚拟 IP，而是通过 Kubernetes DNS 直接为每个关联的 Pod 生成 **固定的 DNS 记录**，格式为：

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

例如，StatefulSet 名称为 `mysql`，Headless Service 名称为 `mysql-svc`，则：

- `mysql-0` 的 DNS 记录为 `mysql-0.mysql-svc.default.svc.cluster.local`
- `mysql-1` 的 DNS 记录为 `mysql-1.mysql-svc.default.svc.cluster.local`

Headless Service 的功劳就在于：它实时监控 Pod IP 的变化，并自动更新 DNS 记录，确保 `mysql-0` 这个名字永远指向那个最新的、正确的 IP。

**3.Headless Service 支持：有序访问与集群发现**

对于分布式集群（如 Kafka、ZooKeeper），节点间需要知道集群成员列表，Headless Service 可以提供 **集群成员发现** 的能力：

- Headless Service 会在 DNS 中维护所有关联 Pod 的 A 记录（IP 地址）。
- 应用通过查询 Headless Service 的 DNS 名称（如 `mysql-svc.default.svc.cluster.local`），可获取所有 Pod 的 IP 列表，实现集群自动发现。

此外，StatefulSet 的 **有序部署 / 伸缩 / 更新** 特性（如先启动 `mysql-0` 再启动 `mysql-1`），结合 Headless Service 的固定 DNS，可确保节点按顺序加入集群（如从库等待主库就绪后再启动）。

**总结**

**StatefulSet 负责给 Pod 一个 “不变的名字”，Headless Service 负责给这个名字绑定 “不变的 DNS 地址”**，两者结合才能让有状态服务在 Pod 动态变化的情况下，保持稳定的网络通信和集群关系。

### Headless Service与普通Service的区别

**普通 Service (ClusterIP)：** 

它的 DNS 记录指向的是 **Service 自己的虚拟 IP (VIP)**。

- 当你访问 `my-svc` 时，流量先到 VIP，然后由系统（iptables/IPVS）**随机负载均衡**到一个后端 Pod。
- 你无法在代码里说：“我只想连接 `web-1`”，因为普通 Service 只有一个入口。**普通 Service 并不提供 Pod 级别的 DNS 记录**，`podname.mysvc.mynamespace.svc.cluster.local` 这种格式，在**普通 Service（ClusterIP）** 下是**不存在**的。

**Headless Service：** 

它的 DNS 记录直接指向 **所有后端 Pod 的具体 IP 列表**。

- **Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 的记录方式解析到后面的 Pod 的 IP 地址。**

- 更重要的是，它为每个 Pod 准备了专属的“分机号”：`<pod-name>.<service-name>`。这让你能精确访问 `mysql-0`。

**总结**

普通 Service（ClusterIP）会在集群内部创建一个 VIP，可以通过这个 VIP 或者这个 service 的 DNS 记录（mysvc.mynamespace.svc.cluster.local）去访问这个 service 代理的 Pod，service 会自动将流量负载均衡到这些 Pod 上，但是客户端无法访问某一个指定的 Pod，只能在所有匹配 selector 指定标签的 Pod 中随机选一个访问。

Headless Service 则是不会在集群内部创建 VIP，而是只有一个 DNS 记录，客户端可以通过 podname 拼接这个DNS记录来实现访问某一个指定的 Pod（podname.mysvc.mynamespace.svc.cluster.local），而且它还能监控 Pod IP 的变化，并自动更新 DNS 记录，确保 Pod 名永远指向那个最新的 IP。

## StatefulSet

在开始之前，我们先准备两个 1G 的存储卷（PV），在后面的课程中我们也会和大家详细讲解 PV 和 PVC 的使用方法的，这里我们先不深究：（pv.yaml）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/pv001

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/pv002
```

然后直接创建 PV 即可：

```shell
➜  ~ kubectl apply -f pv.yaml
persistentvolume "pv001" created
persistentvolume "pv002" created

➜  ~ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv001     1Gi        RWO            Recycle          Available                                      12s
pv002     1Gi        RWO            Recycle          Available                                      11s
```

可以看到成功创建了两个 PV 对象，状态是：`Available`。



### 特性

然后接下来声明一个如下所示的 StatefulSet 资源清单：（nginx-sts.yaml）

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: default
spec:
  serviceName: 'nginx'
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
          image: nginx:1.7.9
          ports:
            - name: web
              containerPort: 80
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates: # statefulset特有的pvc模板，为每个pod示例都创建一个pvc
    - metadata:
        name: www
      spec:
        accessModes: ['ReadWriteOnce']
        resources:
          requests:
            storage: 1Gi
```

从上面的资源清单中可以看出和我们前面的 Deployment 基本上也是一致的，也是通过声明的 Pod 模板来创建 Pod 的，另外上面资源清单中和 `volumeMounts` 进行关联的不是 `volumes` 而是一个新的属性：`volumeClaimTemplates`，该属性会自动创建一个 PVC 对象，其实这里就是一个 PVC 的模板，和 Pod 模板类似，PVC 被创建后会自动去关联当前系统中和他合适的 PV 进行绑定。除此之外，还多了一个 `serviceName: "nginx"` 的字段，`serviceName` 就是管理当前 `StatefulSet` 的服务名称，该服务必须在 StatefulSet 之前存在，并且负责该集合的网络标识，Pod 会遵循以下格式获取 DNS/主机名：`pod-specific-string.serviceName.default.svc.cluster.local`，其中 `pod-specific-string` 由 StatefulSet 控制器管理。

![StatefulSet](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/1662426977822.jpg)

`StatefulSet` 的拓扑结构和其他用于部署的资源对象其实比较类似，比较大的区别在于 `StatefulSet` 引入了 PV 和 PVC 对象来持久存储服务产生的状态，这样所有的服务虽然可以被杀掉或者重启，但是其中的数据由于 PV 的原因不会丢失。

> 注意
>
> 由于我们这里用 `volumeClaimTemplates` 声明的模板是挂载点的方式，并不是 volume，所以实际上上当于把 PV 的存储挂载到容器中，所以会覆盖掉容器中的数据，在容器启动完成后我们可以手动在 PV 的存储里面新建 index.html 文件来保证容器的正常访问，当然也可以进入到容器中去创建，这样更加方便：
>
> ```bash
> $ for i in 0 1; do kubectl exec web-$i -- sh -c 'echo hello $(hostname) > /usr/share/nginx/html/index.html'; done
> ```

现在我们优先创建上面定义的 `Headless Service`：

```shell
➜  ~ kubectl apply -f headless-svc.yaml
service/nginx created

➜  ~ kubectl get service nginx
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   None         <none>        80/TCP    9s
```

`Headless Service` 创建完成后就可以来创建对应的 StatefulSet 对象了：

```shell
➜  ~ kubectl apply -f nginx-sts.yaml
statefulset.apps/web created

➜  ~ kubectl get pvc
NAME        STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pv001    1Gi        RWO                           10m
www-web-1   Bound    pv002    1Gi        RWO                           6m26s
```

可以看到这里通过 Volume 模板自动生成了两个 PVC 对象，也自动和 PV 进行了绑定。这个时候我们可以快速通过一个 `--watch` 参数来查看 Pod 的创建过程：

```shell
➜  ~ kubectl get pods -l app=nginx --watch
NAME                      READY   STATUS              RESTARTS   AGE
web-0                     0/1     ContainerCreating   0          1s
web-0                     1/1     Running             0          2s
web-1                     0/1     Pending             0          0s
web-1                     0/1     Pending             0          0s
web-1                     0/1     ContainerCreating   0          0s
web-1                     1/1     Running             0          6s
```

我们仔细观察整个过程出现了两个 Pod：`web-0` 和 `web-1`，而且这两个 Pod 是按照顺序进行创建的，`web-0` 启动起来后 `web-1` 才开始创建。如同上面 StatefulSet 概念中所提到的，StatefulSet 中的 Pod 拥有一个具有稳定的、独一无二的身份标志。这个标志基于 StatefulSet 控制器分配给每个 Pod 的唯一顺序索引。Pod 的名称的形式为`<statefulset name>-<ordinal index>`。我们这里的对象拥有两个副本，所以它创建了两个 Pod 名称分别为：web-0 和 web-1，我们可以使用 `kubectl exec` 命令进入到容器中查看它们的 hostname：

```shell
➜  ~ kubectl exec web-0 hostname
web-0

➜  ~ kubectl exec web-1 hostname
web-1
```

>  顺序
>
> StatefulSet 中 Pod 副本的创建会按照序列号**升序**处理，副本的更新和删除会按照序列号**降序**处理。

![image-20250922141026783](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/image-20250922141026783.png)

可以看到，这两个 Pod 的 hostname 与 Pod 名字是一致的，都被分配了对应的编号。我们随意查看一个 Pod 的描述信息：

```shell
➜  ~ kubectl describe pod web-0
Name:               web-0
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               ydzs-node3/10.151.30.57
Start Time:         Sun, 17 Nov 2019 12:32:50 +0800
Labels:             app=nginx
                    controller-revision-hash=web-6c5c7fd59b
                    statefulset.kubernetes.io/pod-name=web-0
Annotations:        podpreset.admission.kubernetes.io/podpreset-time-preset: 2062768
Status:             Running
IP:                 10.244.3.98
Controlled By:      StatefulSet/web
......
```

我们可以看到`Controlled By: StatefulSet/web`，证明我们的 Pod 是直接受到 StatefulSet 控制器管理的。

现在我们创建一个 busybox（该镜像中有一系列的工具）的容器，在容器中用 DNS 的方式来访问一下这个 `Headless Service`，由于我们这里只是单纯的为了测试，所以没必要写资源清单文件来声明，用`kubectl run`命令启动一个测试的容器即可：

```shell
➜  ~ kubectl run -it --image busybox:1.28.3 test --restart=Never --rm /bin/sh
If you don't see a command prompt, try pressing enter.
/ #
```

> busybox 镜像
>
> `busybox` 最新版本的镜像有 BUG，会出现 `nslookup` 提示无法解析的问题，我们这里使用老一点的镜像版本 `1.28.3` 即可。

如果对 `kubectl run` 命令的使用参数不清楚，我们可以使用 `kubectl run --help` 命令查看可使用的参数。我们这里使用 `kubectl run` 命令启动了一个以 busybox 为镜像的 Pod，`--rm` 参数意味着我们退出 Pod 后就会被删除，和之前的 `docker run` 命令用法基本一致，现在我们在这个 Pod 容器里面可以使用 `nslookup` 命令来尝试解析下上面我们创建的 `Headless Service`：

```shell
/ # nslookup nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.244.1.175 web-1.nginx.default.svc.cluster.local
Address 2: 10.244.4.83 web-0.nginx.default.svc.cluster.local

/ # ping nginx
PING nginx (10.244.1.175): 56 data bytes
64 bytes from 10.244.1.175: seq=0 ttl=62 time=1.076 ms
64 bytes from 10.244.1.175: seq=1 ttl=62 time=1.029 ms
64 bytes from 10.244.1.175: seq=2 ttl=62 time=1.075 ms
```

我们直接解析 `Headless Service` 的名称，可以看到得到的是两个 Pod 的解析记录，但实际上如果我们通过`nginx`这个 DNS 去访问我们的服务的话，并不会随机或者轮询背后的两个 Pod，而是访问到一个固定的 Pod，所以不能代替普通的 Service。如果分别解析对应的 Pod 呢？

```shell
$ / # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.4.83 web-0.nginx.default.svc.cluster.local

/ # nslookup web-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.1.175 web-1.nginx.default.svc.cluster.local
```

可以看到解析 `web-0.nginx` 的时候解析到了 `web-0` 这个 Pod 的 IP，`web-1.nginx` 解析到了 `web-1` 这个 Pod 的 IP，而且这个 DNS 地址还是稳定的，因为 Pod 名称就是固定的，比如我们这个时候去删掉 `web-0` 和 `web-1` 这两个 Pod：

```shell
➜  ~ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

删除完成后再看 Pod 状态：

```shell
➜  ~ kubectl get pods -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          42s
web-1   1/1     Running   0          39s
```

可以看到 StatefulSet 控制器仍然会按照顺序创建出两个 Pod 副本出来，而且 Pod 的唯一标识依然没变，所以这两个 Pod 的网络标识还是固定的，我们依然可以通过`web-0.nginx`去访问到`web-0`这个 Pod，虽然 Pod 已经重建了，对应 Pod IP 已经变化了，但是访问这个 Pod 的地址依然没变，并且他们依然还是关联的之前的 PVC，数据并不会丢失：

```shell
/ # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.3.98 web-0.nginx.default.svc.cluster.local
/ # nslookup web-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.1.176 web-1.nginx.default.svc.cluster.local
```

通过 `Headless Service`，StatefulSet 就保证了 Pod 网络标识的唯一稳定性，由于 Pod IP 并不是固定的，所以我们访问`有状态应用`实例的时候，就必须使用 DNS 记录的方式来访问了，所以很多同学偶尔有固定的 Pod IP 的需求，或许可以用这种方式来代替。

最后我们可以通过删除 StatefulSet 对象来删除所有的 Pod，仔细观察也会发现是按照倒序的方式进行删除的：

```shell
➜  ~ kubectl delete statefulsets web
statefulset.apps "web" deleted

➜  ~ kubectl get pods --watch
NAME    READY   STATUS    RESTARTS   AGE
web-1   1/1   Terminating   0     3h/31m
web-0   1/1   Terminating   0     3h/31m
```



### StatefulSet 扩容和缩容 

和 Deployment 类似，可以通过更新 replicas 字段扩容/缩容 StatefulSet，也可以使用 kubectl scale、kubectl edit 和 kubectl patch 来扩容/缩容一个 StatefulSet。

（1）扩容

将上述创建的 sts 副本增加到 5 个：

```bash
# kubectl scale sts web --replicas=5
statefulset.apps/web scaled
```

查看扩容后 Pod 的状态：

```bash
# kubectl get po
NAME READY STATUS RESTARTS AGE
web-0 1/1 Running 0 2m58s
web-1 1/1 Running 0 2m48s
web-2 1/1 Running 0 116s
web-3 1/1 Running 0 79s
web-4 1/1 Running 0 53s
```

也可使用以下命令动态查看：

```bash
kubectl get pods -w -l app=nginx
```

（2）缩容

首先打开另一个终端动态查看缩容的流程：

```bash
# kubectl get pods -w -l app=nginx
NAME READY STATUS RESTARTS AGE
web-0 1/1 Running 0 4m37s
web-1 1/1 Running 0 4m27s
web-2 1/1 Running 0 3m35s
web-3 1/1 Running 0 2m58s
web-4 1/1 Running 0 2m32s
```

在另一个终端将副本数改为 3：

```bash
# kubectl scale sts web --replicas=3
statefulset.apps/web patched
```

此时可以看到第一个终端显示 web-4 和 web-3 的 Pod 正在被有序的删除（或终止）：

```bash
# kubectl get pods -w -l app=nginx
NAME READY STATUS RESTARTS AGE
web-0 1/1 Running 0 4m37s
web-1 1/1 Running 0 4m27s
web-2 1/1 Running 0 3m35s
web-3 1/1 Running 0 2m58s
web-4 1/1 Running 0 2m32s
web-0 1/1 Running 0 5m8s
web-0 1/1 Running 0 5m11s
web-4 1/1 Terminating 0 3m36s
web-4 0/1 Terminating 0 3m38s
web-4 0/1 Terminating 0 3m47s
web-4 0/1 Terminating 0 3m47s
web-3 1/1 Terminating 0 4m13s
web-3 0/1 Terminating 0 4m14s
web-3 0/1 Terminating 0 4m22s
web-3 0/1 Terminating 0 4m22s
```



### StatefulSet 更新策略

（1）OnDelete 策略

OnDelete（手动删除后重建，1.7 之前的版本的默认策略）更新策略当我们选修改 StatefulSet的.spec.template 字段时，StatefulSet 控制器不会自动更新 Pod，必须手动删除 Pod 才能使控制器创建新的 Pod。

（2）RollingUpdate 策略

RollingUpdate（滚动更新，默认）更新策略会自动更新一个 StatefulSet 中所有的 Pod，采用与序号索引相反（倒序）的顺序进行滚动更新。

![image-20250922141054715](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/image-20250922141054715.png)

查看更新策略：

```bash
# kubectl get sts web -o yaml | grep -A 1 "updateStrategy"
 updateStrategy:
 type: RollingUpdate
```

然后改变容器的镜像触发滚动更新：

```bash
# kubectl set image sts web nginx=registry.cn￾beijing.aliyuncs.com/dotbalo/canary:v1
statefulset.apps/web patched
```

在更新过程中可以使用 kubectl rollout status sts/$name 来查看滚动更新的状态：

```bash
# kubectl rollout status sts/web
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 1 pods at revision 
web-56b5798f76...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 2 pods at revision 
web-56b5798f76...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
statefulset rolling update complete 3 pods at revision web-56b5798f76...
```



### StatefulSet 分段更新

![image-20250922145517918](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/image-20250922145517918.png)

比如我们定义一个分区"partition":3（定义为代表只有序号大于等于 3 的才会更新），可以使用 patch 或 edit 直接对 StatefulSet 进行设置：

```bash
# kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate", "rollingUpdate":{"partition":3}}}}'
statefulset "web" patched
```

然后再次使用 patch 改变容器的镜像：

```bash
# kubectl set image statefulset web nginx=registry.cn-beijing.aliyuncs.com/dotbalo/canary:v2
```

此时，因为 Pod 最大的序号小于分区 3，所以 Pod 不会被更新。

将分区改为 2，此时会自动更新 web-2，但是不会更新 web-0 和 web-1：

```bash
# kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"type":"RollingUpdate", "rollingUpdate":{"partition":2}}}}'
statefulset "web" patched
```

按照上述方式，可以实现分阶段更新，类似于灰度/金丝雀发布。查看最终的结果如下：

```bash
# kubectl get po -oyaml | grep image
dotbalo/canary:v1
dotbalo/canary:v1
dotbalo/canary:v2
```



### StatefulSet 回滚 

StatefulSet 的更新和回滚与 Deployment 类似，唯一的区别是历史版本在 ControllerRevision中保存。

```bash
# 查看更新状态
kubectl rollout status sts/<daemonset-name>

# 列出所有修订版本
kubectl rollout history sts <daemonset-name>

# 回滚到指定 revision
kubectl rollout undo sts <statefulset-name> --to-revision=<revision>
```



### 删除 StatefulSet

删除StatefulSet有两种方式，即级联删除和非级联删除。使用非级联方式删除 StatefulSet时，StatefulSet 的 Pod 不会被删除；使用级联删除时，StatefulSet 和它的 Pod 都会被删除。

（1）非级联删除（了解）

使用 kubectl delete sts xxx 删除 StatefulSet 时，只需提供--cascade=false 参数，就会采用
非级联删除，此时删除 StatefulSet 不会删除它的 Pod：

```bash
# kubectl get po 
NAME READY STATUS RESTARTS AGE
web-0 1/1 Running 0 16m
web-1 1/1 Running 0 16m
web-2 1/1 Running 0 11m

# kubectl delete statefulset web --cascade=false # 采用非级联删除
statefulset.apps "web" deleted

# kubectl get sts # 查看此时 sts 已经被删除
No resources found.

# kubectl get po # 该 StatefulSet 管理的 Pod 并未被删除
NAME READY STATUS RESTARTS AGE
web-0 1/1 Running 0 16m
web-1 1/1 Running 0 16m
web-2 1/1 Running 0 11m
```

由于此时删除了 StatefulSet，它管理的 Pod 变成了“孤儿”Pod，因此单独删除 Pod 时，该Pod 不会被重建：

```bash
# kubectl get po
NAME READY STATUS RESTARTS AGE
web-0 1/1 Running 0 16m
web-1 1/1 Running 0 16m
web-2 1/1 Running 0 11m

# kubectl delete po web-0
pod "web-0" deleted

# kubectl get po
NAME READY STATUS RESTARTS AGE
web-1 1/1 Running 0 18m
web-2 1/1 Running 0 12m
```

再次创建 sts：

```bash
# kubectl apply -f sts-web.yaml 
statefulset.apps/web created

# kubectl get po
NAME READY STATUS RESTARTS AGE
web-0 1/1 Running 0 32s
web-1 1/1 Running 0 19m
```

（2）级联删除

省略--cascade=false 参数即为级联删除：

```bash
# kubectl delete statefulset web
statefulset.apps "web" deleted

# kubectl get po
No resources found.
```

也可以使用-f 指定创建 StatefulSet 和 Service 的 yaml 文件，直接删除 StatefulSet 和 Service（此文件将 StatefulSet 和 Service 写在了一起）：

```bash
# kubectl delete -f sts-web.yaml 
service "nginx" deleted
Error from server (NotFound): error when deleting "sts-web.yaml": 
statefulsets.apps "web" not found # 因为 StatefulSet 已经被删除，所以会提示该StatefulSet 不存在
```



### StatefulSet 并发 Pod 管理 

StatefulSet 可以通过.spec.podManagementPolicy 字段配置 Pod 的管理策略，目前支持如下的两种方式：

OrderdReady：有序管理，默认方式

- **创建**：严格按 Pod 序号从低到高（0→1→2…）依次创建，前一个 Pod 就绪（Ready）后才会创建下一个。
- **更新（滚动更新）**：按序号从高到低（n→n-1→…→0）依次删除旧 Pod 并创建新 Pod，前一个新 Pod 就绪后才会处理下一个。
- **删除**：按序号从高到低（n→n-1→…→0）**串行删除**（并非 “同时删除”），前一个 Pod 完全终止后才会删除下一个。

Parallel：并发管理，Pod 创建和删除同时并行启动和删除，更新还是倒序更新。

- **创建**：所有 Pod 同时并行创建，无需等待前一个 Pod 就绪。
- **更新（滚动更新）**：按序号从高到低（n→n-1→…→0）依次处理（与 `OrderedReady` 相同），但会同时进行删除旧 Pod 并创建新 Pod的操作（无需等待前一个新 Pod 就绪）。
- **删除**：所有 Pod **同时并行删除**（而非 “更新还是倒序更新” 的模糊描述）。

示例：

```yaml
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
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  podManagementPolicy: Parallel #
  serviceName: "nginx"
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
          image: registry.cn-beijing.aliyuncs.com/dotbalo/nginx:stable
          ports:
            - containerPort: 80
              name: web
```



### StatefulSet 内部通信

StatefulSet 创建的 Pod 一般使用 Headless Service（无头服务）进行 Pod 之前的通信，和普通的 Service 的区别在于 Headless Service 没有 ClusterIP，它使用的是 Endpoint 进行互相通信，Headless 一般的格式为：

statefulSetName-{0..N-1}.serviceName.namespace.svc.cluster.local

- serviceName 为 Headless Service 的名字，创建 StatefulSet 时，必须指定 Headless Service
  名称；
- 0..N-1 为 Pod 所在的序号，从 0 开始到 N-1；
- statefulSetName 为 StatefulSet 的名字；
- namespace 为服务所在的命名空间；
- .cluster.local 为 Cluster Domain（集群域）。

使用任意一个 Pod 即可通过唯一的网络标识符访问 StatefulSet 的节点：

```bash
kubectl exec -ti web-2 -- curl web-2.nginx  # 同一个 namespace 可以省略.namespace.svc.cluster.local

# 不同 Namespace 的访问，需要加上 Pod 所在的 Namespace：
ping web-2.nginx.default
```



### 管理策略

对于某些分布式系统来说，StatefulSet 的顺序性保证是不必要和/或者不应该的，这些系统仅仅要求唯一性和身份标志。为了解决这个问题，我们只需要在声明 StatefulSet 的时候重新设置 `spec.podManagementPolicy` 的策略即可。

默认的管理策略是 `OrderedReady`，表示让 StatefulSet 控制器遵循上文演示的顺序性保证。除此之外，还可以设置为 `Parallel` 管理模式，表示让 StatefulSet 控制器并行的终止所有 Pod，在启动或终止另一个 Pod 前，不必等待这些 Pod 变成 Running 和 Ready 或者完全终止状态。



### 更新策略

前面课程中我们学习了 Deployment 的升级策略，在 StatefulSet 中同样也支持两种升级策略：`onDelete` 和 `RollingUpdate`，同样可以通过设置 `.spec.updateStrategy.type` 进行指定。

- `OnDelete`: 该策略表示当更新了 `StatefulSet` 的模板后，只有手动删除旧的 Pod 才会创建新的 Pod。
- `RollingUpdate`：该策略表示当更新 StatefulSet 模板后会自动删除旧的 Pod 并创建新的 Pod，如果更新发生了错误，这次“滚动更新”就会停止。不过需要注意 StatefulSet 的 Pod 在部署时是顺序从 0~n 的，而在滚动更新时，这些 Pod 则是按逆序的方式即 n~0 一次删除并创建。

另外`SatefulSet` 的滚动升级还支持 `Partitions`的特性，可以通过`.spec.updateStrategy.rollingUpdate.partition` 进行设置，在设置 partition 后，SatefulSet 的 Pod 中序号大于或等于 partition 的 Pod 会在 StatefulSet 的模板更新后进行滚动升级，而其余的 Pod 保持不变，这个功能是不是可以实现**灰度发布**？大家可以去手动验证下。

在实际的项目中，其实我们还是很少会去直接通过 StatefulSet 来部署我们的有状态服务的，除非你自己能够完全能够 hold 住，对于一些特定的服务，我们可能会使用更加高级的 Operator 来部署，比如 etcd-operator、prometheus-operator 等等，这些应用都能够很好的来管理有状态的服务，而不是单纯的使用一个 StatefulSet 来部署一个 Pod 就行，因为对于有状态的应用最重要的还是数据恢复、故障转移等等。



# 守护进程集 DaemonSet

通过该控制器的名称我们可以看出它的用法：`Daemon`，就是用来部署守护进程的，`DaemonSet`用于在每个 Kubernetes 节点中将守护进程的副本作为后台进程运行，说白了就是在每个节点部署一个 Pod 副本，当节点加入到 Kubernetes 集群中，Pod 会被调度到该节点上运行，当节点从集群中被移除后，该节点上的这个 Pod 也会被移除，当然，如果我们删除 DaemonSet，所有和这个对象相关的 Pods 都会被删除。那么在哪种情况下我们会需要用到这种业务场景呢？其实这种场景还是比较普通的，比如：

- 集群存储守护程序，如 glusterd、ceph 要部署在每个节点上以提供持久性存储；
- 节点监控守护进程，如 Prometheus 监控集群，可以在每个节点上运行一个 `node-exporter` 进程来收集监控节点的信息；
- 日志收集守护程序，如 fluentd 或 logstash，在每个节点上运行以收集容器的日志
- 节点网络插件，比如 flannel、calico，在每个节点上运行为 Pod 提供网络服务。

这里需要特别说明的一个就是关于 DaemonSet 运行的 Pod 的调度问题，正常情况下，Pod 运行在哪个节点上是由 Kubernetes 的调度器策略来决定的，然而，由 DaemonSet 控制器创建的 Pod 实际上提前已经确定了在哪个节点上了（Pod 创建时指定了`.spec.nodeName`），所以：

- `DaemonSet` 并不关心一个节点的 `unshedulable` 字段，这个我们会在后面的调度章节和大家讲解的。
- `DaemonSet` 可以创建 Pod，即使调度器还没有启动。

下面我们直接使用一个示例来演示下，在每个节点上部署一个 Nginx Pod：

```yaml
# nginx-ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ds
  namespace: default
spec:
  selector:
    matchLabels:
      k8s-app: nginx
  template:
    metadata:
      labels:
        k8s-app: nginx
    spec:
      containers:
        - image: nginx:1.7.9
          name: nginx
          ports:
            - name: http
              containerPort: 80
```

然后直接创建即可：

```shell
➜  ~ kubectl apply -f nginx-ds.yaml
daemonset.apps/nginx-ds created
```

创建完成后，我们查看 Pod 的状态：

```shell
➜  ~ kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
master1   Ready    control-plane,master   18d   v1.22.2
node1     Ready    <none>                 18d   v1.22.2
node2     Ready    <none>                 18d   v1.22.2

➜  ~ kubectl get pods -l k8s-app=nginx -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
nginx-ds-5b2m7   1/1     Running   0          15s   10.244.2.165   node2   <none>           <none>
nginx-ds-jfr89   1/1     Running   0          15s   10.244.1.170   node1   <none>           <none>
```

我们观察可以发现除了 master1 节点之外的 2 个节点上都有一个相应的 Pod 运行，因为 master1 节点上默认被打上了`污点`，所以默认情况下不能调度普通的 Pod 上去，后面讲解调度器的时候会和大家学习如何调度上去。

![image-20250922165655754](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/image-20250922165655754.png)

基本上我们可以用下图来描述 DaemonSet 的拓扑图：

![DaemonSet](k8s%E8%B5%84%E6%BA%90%E8%B0%83%E5%BA%A6.assets/1662427459142.jpg)

集群中的 Pod 和 Node 是一一对应 d 的，而 DaemonSet 会管理全部机器上的 Pod 副本，负责对它们进行更新和删除。

那么，DaemonSet 控制器是如何保证每个 Node 上有且只有一个被管理的 Pod 呢？

- 首先控制器从 Etcd 获取到所有的 Node 列表，然后遍历所有的 Node。
- 根据资源对象定义是否有调度相关的配置，然后分别检查 Node 是否符合要求。
- 在可运行 Pod 的节点上检查是否已有对应的 Pod，如果没有，则在这个 Node 上创建该 Pod；如果有，并且数量大于 1，那就把多余的 Pod 从这个节点上删除；如果有且只有一个 Pod，那就说明是正常情况。

实际上当我们学习了资源调度后，我们也可以自己用 Deployment 来实现 DaemonSet 的效果，这里我们明白 DaemonSet 如何使用的即可，当然该资源对象也有对应的更新策略，有 `OnDelete` 和 `RollingUpdate` 两种方式，默认是滚动更新。



## DaemonSet 更新策略 

如果添加了新节点或修改了节点标签（Label），DaemonSet 将立刻向新匹配上的节点添加Pod，同时删除不能匹配的节点上的 Pod。

DaemonSet 更新策略和 StatefulSet 类似，也有 OnDelete 和 RollingUpdate 两种方式。

查看上一节创建的 DaemonSet 更新方式：

```bash
# kubectl get ds nginx-ds -o go-template='{{.spec.updateStrategy.type}}{{"\n"}}'
RollingUpdate
```

命令式更新，和之前 Deployment、StatefulSet 方式一致

```bash
kubectl edit ds/<daemonset-name>

kubectl patch ds/<daemonset-name> -p=<strategic-merge-patch>

kubectl set image ds/<daemonset-name><container-name>= <container-new-image> --record=true
```



## DaemonSet 回滚

DaemonSet 的更新和回滚与 Deployment 类似，唯一的区别是历史版本在 ControllerRevision 中保存。

```bash
# 查看更新状态
kubectl rollout status ds/<daemonset-name>

# 列出所有修订版本
kubectl rollout history daemonset <daemonset-name>

# 查看controllerrevisions下的版本号
kubectl get controllerrevisions

# 回滚到指定 revision
kubectl rollout undo daemonset <daemonset-name> --to-revision=<revision>
```



## 指定节点部署 pod

如果指定了.spec.template.spec.nodeSelector，DaemonSet Controller 将在与 Node Selector（节点选择器）匹配的节点上创建 Pod，比如部署在磁盘类型为 ssd 的节点上（需要提前给节点定义标签 Label）：

```yaml
nodeSelector:
  disktype: ssd
```

给节点定义标签

```bash
kubectl lable node k8s-node01 disktype=ssd
```



