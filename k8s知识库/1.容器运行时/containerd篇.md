# containerd篇

## docker与containerd的关系

从 Docker 1.11 版本开始，Docker 容器运行就不是简单通过 Docker Daemon 来启动了，而是通过集成 containerd、runc 等多个组件来完成的。Docker 是 CS 架构，守护进程负责和 Docker Client 端交互，并管理 Docker 镜像和容器。现在的架构中组件 containerd 负责集群节点上容器的生命周期管理，并向上为 Docker Daemon 提供 gRPC 接口。

![image-20250917154347503](containerd%E7%AF%87.assets/image-20250917154347503.png)

当我们要创建一个容器时，请求 `containerd` 来创建一个容器，containerd 收到请求后，创建一个叫做 `containerd-shim` 的进程，让这个进程去操作容器，我们指定容器进程是需要一个父进程来做状态收集、维持 stdin 等 fd 打开等工作的，假如这个父进程就是 containerd，那如果 containerd 挂掉的话，整个宿主机上所有的容器都得退出了，而引入 `containerd-shim` 这个垫片就可以来规避这个问题了。

然后创建容器需要做一些 namespaces 和 cgroups 的配置，以及挂载 root 文件系统等操作，这些操作其实已经有了标准的规范，那就是 OCI（开放容器标准），`runc` 就是它的一个参考实现（Docker 被逼无耐将 `libcontainer` 捐献出来改名为 `runc` 的），这个标准其实就是一个文档，主要规定了容器镜像的结构、以及容器需要接收哪些操作指令，比如 create、start、stop、delete 等这些命令。`runc` 就可以按照这个 OCI 文档来创建一个符合规范的容器，既然是标准肯定就有其他 OCI 实现，比如 Kata、gVisor 这些容器运行时都是符合 OCI 标准的。

所以真正启动容器是通过 `containerd-shim` 去调用 `runc` 来启动容器的，`runc` 启动完容器后本身会直接退出，`containerd-shim` 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程。



## CRI

`CRI`（Container Runtime Interface 容器运行时接口）本质上就是 Kubernetes 定义的一组与容器运行时进行交互的接口，所以只要实现了这套接口的容器运行时都可以对接到 Kubernetes 平台上来。不过 Kubernetes 推出 CRI 这套标准的时候还没有现在的统治地位，所以有一些容器运行时可能不会自身就去实现 CRI 接口，于是就有了 `shim（垫片）`， 一个 shim 的职责就是作为适配器将各种容器运行时本身的接口适配到 Kubernetes 的 CRI 接口上，其中 `dockershim` 就是 Kubernetes 对接 Docker 到 CRI 接口上的一个垫片实现。

![image-20250917154812293](containerd%E7%AF%87.assets/image-20250917154812293.png)

Kubelet 通过 gRPC 框架与容器运行时或 shim 进行通信，其中 kubelet 作为客户端，CRI shim（也可能是容器运行时本身）作为服务器。

不过这里同样也有一个例外，由于 Docker 当时的江湖地位很高，Kubernetes 是直接内置了 `dockershim` 在 kubelet 中的，所以如果你使用的是 Docker 这种容器运行时的话是不需要单独去安装配置适配器之类的

![image-20250917155453253](containerd%E7%AF%87.assets/image-20250917155453253.png)

现在如果我们使用的是 Docker 的话，当我们在 Kubernetes 中创建一个 Pod 的时候，首先就是 kubelet 通过 CRI 接口调用 `dockershim`，请求创建一个容器，kubelet 可以视作一个简单的 CRI Client, 而 dockershim 就是接收请求的 Server，不过他们都是在 kubelet 内置的。

`dockershim` 收到请求后, 转化成 Docker Daemon 能识别的请求, 发到 Docker Daemon 上请求创建一个容器，请求到了 Docker Daemon 后续就是 Docker 创建容器的流程了，去调用 `containerd`，然后创建 `containerd-shim` 进程，通过该进程去调用 `runc` 去真正创建容器。

不难发现使用 Docker 的话其实是调用链比较长的，真正容器相关的操作其实 containerd 就完全足够了，对于 Kubernetes 来说 Docker 太过于复杂笨重了，因为都是通过接口去操作容器的，所以自然也就可以将容器运行时切换到 containerd 来。

自 containerd 1.1 版本起，它已**原生内置了 CRI 支持**（即 `containerd-cri` 插件），可以直接响应 Kubernetes 的 CRI 调用。因此现代 Kubernetes 与 containerd 的交互链简化为：`Kubernetes → containerd（直接通过 CRI）→ 底层运行时（如 runc）`。

![image-20250917155050419](containerd%E7%AF%87.assets/image-20250917155050419.png)



## 关于containerd

Containerd是一种容器运行时，可以管理容器的整个生命周期，包含镜像的传输、容器的运行和销毁、容器的监控，同时也可以管理更底层的存储和网络等。Containerd属于Docker引擎中的一部分，在2016年12月从Docker Engine中剥离，成为了一个可以独立使用的容器运行时（Runtime），并且在2017年捐赠给了CNCF，成为了CNCF的顶级项目之一。Containerd是一个开源的容器运行时项目，同时也是CNCF体系中已经毕业的容器运行时。



### containerd客户端工具对比

![image-20250917160620520](containerd%E7%AF%87.assets/image-20250917160620520.png)



### Containerd配置insecure registry

```bash
vim /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
	[plugins."io.containerd.grpc.v1.cri".registry.mirrors."harbro镜像仓库的ip"]
		endpoint = ["http://harbro镜像仓库的ip"]
systemctl restart containerd
```

**三个注意事项**

- 需要把K8s所有的节点都需要进行更改

- config.toml的不安全仓库配置并不会对ctr命令生效

- docker拉取的镜像和containerd没有任何关系

**ctr 想拉取镜像需要单独配置**

```bash
# config.toml的不安全仓库配置并不会对ctr命令生效
ctr i pull 仓库链接/命名空间/镜像名:标签 --plain-http
```



### containerd命名空间管理

Containerd的Namespace是一个强大的工具，主要用于实现容器之间的资源隔离、访问控制和安全性。可以实现多个容器在同一台主机上独立运行而不会相互干扰，从而提高了系统的可扩展性和可管理性。

**Containerd的命令空间和Kubernetes的命令空间是两个不同的概念！**

```bash
# 命令帮助
ctr ns -h

# 查看containerd有哪些命名空间
ctr ns ls [-q]

# 新建命名空间
[root@k8s-master01 ~]# ctr ns c test
[root@k8s-master01 ~]# ctr ns ls
NAME   LABELS 
k8s.io        
test          

# 给命名空间打标签
[root@k8s-master01 ~]# ctr ns label test a=b
[root@k8s-master01 ~]# ctr ns ls
NAME   LABELS 
k8s.io        
test   a=b    

# 删除指定命名空间
[root@k8s-master01 ~]# ctr ns rm test
test
[root@k8s-master01 ~]# ctr ns ls
NAME   LABELS 
k8s.io        
```

**ctr在k8s所有的节点都导入了镜像，为什么还是提示本地没有这个镜像？**

```bash
# 因为k8s在containerd中的命名空间就是k8s.io，并且想要正常使用，还必须给所有节点的k8s.io都导入镜像
[root@k8s-master01 ~]# ctr ns ls
NAME   LABELS 
k8s.io        

[root@k8s-master01 ~]# ctr -n k8s.io i ls
REF                                                                                                                                                 TYPE                                                      DIGEST                                                                  SIZE      PLATFORMS                                                                    LABELS                                                          
registry.cn-beijing.aliyuncs.com/dotbalo/cni:v3.29.1                                                                                                application/vnd.docker.distribution.manifest.v2+json      sha256:0f28c2e3158e40e59207bed46096acf65ab3e8abbbc1855e1e6e4f17f588527a 93.1 MiB  linux/amd64                                                                  io.cri-containerd.image=managed                                 
registry.cn-beijing.aliyuncs.com/dotbalo/cni@sha256:0f28c2e3158e40e59207bed46096acf65ab3e8abbbc1855e1e6e4f17f588527a                                application/vnd.docker.distribution.manifest.v2+json      sha256:0f28c2e3158e40e59207bed46096acf65ab3e8abbbc1855e1e6e4f17f588527a 93.1 MiB  linux/amd64                                                                  io.cri-containerd.image=managed                                 
registry.cn-beijing.aliyuncs.com/dotbalo/node:v3.29.1                                                                                               application/vnd.docker.distribution.manifest.v2+json      sha256:aeeaedbed4e066951ef21c227507271ffafc04099ed4a501d025ed2675a425e9 136.1 MiB linux/amd64                                                                  io.cri-containerd.image=managed                                 
registry.cn-beijing.aliyuncs.com/dotbalo/node@sha256:aeeaedbed4e066951ef21c227507271ffafc04099ed4a501d025ed2675a425e9                               application/vnd.docker.distribution.manifest.v2+json      sha256:aeeaedbed4e066951ef21c227507271ffafc04099ed4a501d025ed2675a425e9 136.1 MiB linux/amd64                                                                  io.cri-containerd.image=managed             # 后面省略
```



### containerd镜像管理

```bash
# 拉取镜像
ctr i pull registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1


# 查看镜像
[root@k8s-master01 ~]# ctr -n default i ls
REF                                                 TYPE                                                 DIGEST                                                                  SIZE     PLATFORMS   LABELS 
registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1 application/vnd.docker.distribution.manifest.v2+json sha256:dad22fecd8043fb4178d4cf9252c938cda16dbc898ea00ba2c0808407a28a607 16.0 MiB linux/amd64 -      


# 推送镜像
ctr i push xxx --user --http-plain


# 导出镜像
[root@k8s-master01 ~]# ctr i export counter-v1.tar registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1
[root@k8s-master01 ~]# ls
anaconda-ks.cfg  counter-v1.tar  k8s-ha-install

# 导入镜像到指定的namespace
[root@k8s-master01 ~]# ctr -n counter i import counter-v1.tar 
unpacking registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1 (sha256:dad22fecd8043fb4178d4cf9252c938cda16dbc898ea00ba2c0808407a28a607)...done


# 将镜像根目录挂载到指定目录
[root@k8s-master01 ~]# ctr -n default i mount registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1 /mnt/
sha256:f5e9a7ba8d8e4eb4d8b329780284d433b6170b95001c1b046e111fa0995c3bf6
/mnt/
[root@k8s-master01 ~]# ls /mnt
bin  docker-entrypoint.d   etc   lib    mnt  proc  run   srv  tmp  var
dev  docker-entrypoint.sh  home  media  opt  root  sbin  sys  usr

# 取消挂载
[root@k8s-master01 ~]# ctr -n default i unmount /mnt/
/mnt/
[root@k8s-master01 ~]# ls /mnt


# 删除镜像
[root@k8s-master01 ~]# ctr -n counter i rm registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1
registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1

[root@k8s-master01 ~]# ctr -n counter i ls
REF TYPE DIGEST SIZE PLATFORMS LABELS 


# 修改镜像标签
ctr i tag
```



### containerd容器管理

```bash
# 启动容器
[root@k8s-master01 ~]# ctr -n default c create registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1 counter


# 查看容器运行状态
[root@k8s-master01 ~]# ctr -n default c ls
CONTAINER    IMAGE                                                  RUNTIME                  
counter      registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1    io.containerd.runc.v2 


# 查看容器信息（类似于docker inspect）
[root@k8s-master01 ~]# ctr c info counter
{
    "ID": "counter",
    "Labels": {
        "io.containerd.image.config.stop-signal": "SIGQUIT",
        "maintainer": "NGINX Docker Maintainers \u003cdocker-maint@nginx.com\u003e"
    },
    "Image": "registry.cn-beijing.aliyuncs.com/dotbalo/counter:v1",
    "Runtime": {


# 删除容器
[root@k8s-master01 ~]# ctr c rm counter

[root@k8s-master01 ~]# ctr c ls
CONTAINER    IMAGE    RUNTIME   
```



### nerdctl安装

```bash
# 获取版本：
https://github.com/containerd/nerdctl/releases
# 下载工具：
wget https://github.com/containerd/nerdctl/releases/download/v1.7.6/nerdctl-1.7.6-linux-amd64.tar.gz

# 安装：
[root@k8s-master01 ~]# ls
anaconda-ks.cfg  k8s-ha-install  nerdctl-1.7.6-linux-amd64.tar.gz

[root@k8s-master01 ~]# tar -zxf nerdctl-1.7.6-linux-amd64.tar.gz

[root@k8s-master01 ~]# ls
anaconda-ks.cfg                   containerd-rootless.sh  nerdctl
containerd-rootless-setuptool.sh  k8s-ha-install          nerdctl-1.7.6-linux-amd64.tar.gz

[root@k8s-master01 ~]# mv nerdctl /usr/local/bin/

[root@k8s-master01 ~]# nerdctl version
WARN[0000] unable to determine buildctl version: exec: "buildctl": executable file not found in $PATH 
Client:
 Version:	v1.7.6
 OS/Arch:	linux/amd64
 Git commit:	845e989f69d25b420ae325fedc8e70186243fd93
 buildctl:
  Version:	

Server:
 containerd:
  Version:	1.7.27
  GitCommit:	05044ec0a9a75232cad458027ca83437aae3f4da
 runc:
  Version:	1.2.5
  GitCommit:	v1.2.5-0-g59923ef
```

nerdctl的命令和docker几乎一样，区别就是多了一个命名空间的概念

```bash
[root@k8s-master01 ~]# nerdctl ns ls
NAME       CONTAINERS    IMAGES    VOLUMES    LABELS
default    0             1         0              
k8s.io     27            30        0              

[root@k8s-master01 ~]# nerdctl -n k8s.io ps
CONTAINER ID    IMAGE                                                                                  COMMAND                   CREATED        STATUS    PORTS    NAMES
057712d3e779    registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.16-0                      "etcd --advertise-cl…"    4 hours ago    Up                 k8s://kube-system/etcd-k8s-master01/etcd
099ee883fd75    registry.cn-beijing.aliyuncs.com/dotbalo/node:v3.29.1                                  "start_runit"             4 hours ago    Up                 k8s://kube-system/calico-node-gv747/calico-node
0cae2f44a976    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.32.9             "kube-scheduler --au…"    4 hours ago    Up                 k8s://kube-system/kube-scheduler-k8s-master01/kube-scheduler
17877ab83eb3    registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8                          "/pause"                  4 hours ago    Up                 k8s://kube-system/kube-apiserver-k8s-master01
1a2268d67960    registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8                          "/pause"                  4 hours ago    Up                 k8s://kube-system/etcd-k8s-master01
1b0f7fe0a680    registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8                          "/pause"                  4 hours ago    Up                 k8s://kube-system/kube-controller-manager-k8s-master01
38aa579eea8e    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.32.9             "kube-apiserver --ad…"    4 hours ago    Up                 k8s://kube-system/kube-apiserver-k8s-master01/kube-apiserver
3b8ce170d534    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.32.9                 "/usr/local/bin/kube…"    4 hours ago    Up                 k8s://kube-system/kube-proxy-78t8r/kube-proxy
6cb055deb17f    registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8                          "/pause"                  4 hours ago    Up                 k8s://kube-system/kube-proxy-78t8r
7b23876d4e4a    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.32.9    "kube-controller-man…"    4 hours ago    Up                 k8s://kube-system/kube-controller-manager-k8s-master01/kube-controller-manager
c6b5618557a9    registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8                          "/pause"                  4 hours ago    Up                 k8s://kube-system/kube-scheduler-k8s-master01
d6feb5bcb361    registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.8                          "/pause"                  4 hours ago    Up                 k8s://kube-system/calico-node-gv747
```











