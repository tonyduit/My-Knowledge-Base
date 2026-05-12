#  02_docker安装部署

# 解决Docker国内网络问题

2024年6月以来，大量Docker镜像网站停服，Docker无法下载安装
本仓库致力于解决国内网络原因无法使用Docker的问题。

### 特点：

- 使用Github Action将官网的安装脚本/安装包定时下载到本项目Release，供国内使用
- 官方安装包，安全可靠
- 每天自动定时同步，保证最新

# 1. Docker安装

## 1.1 Linux

一键安装命令（每天自动从官网定时同步）

```bash
sudo curl -fsSL https://github.com/tech-shrimp/docker_installer/releases/download/latest/linux.sh| bash -s docker --mirror Aliyun
```

> 备用（如果Github访问不了，可以使用Gitee的链接）

```bash
sudo curl -fsSL https://gitee.com/tech-shrimp/docker_installer/releases/download/latest/linux.sh| bash -s docker --mirror Aliyun
```

启动docker

```bash
sudo service docker start
```



## 1.2 Windows

任务栏搜索功能，启用"适用于Linux的Windows子系统" + "虚拟机平台"

![image-20240901191239568](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20240901191239568.png)

管理员权限打开命令提示符，安装wsl2

```
wsl --set-default-version 2
wsl --update --web-download
```

等待wsl安装成功

![wsl2成功](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/wsl2%E6%88%90%E5%8A%9F.png)

下载Windows版本安装包，进入此项目的Release
https://github.com/tech-shrimp/docker_installer/releases

下载Windows版本安装包

![windows安装包](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/windows%E5%AE%89%E8%A3%85%E5%8C%85.png)

双击安装即可

> 可选: 如果想自己指定安装目录，可以使用命令行的方式 参数 --installation-dir=D:\Docker可以指定安装位置

```
start /w "" "Docker Desktop Installer.exe" install --installation-dir=D:\Docker
```



## 1.3 Mac

进入此项目的Release，下载Mac系统的安装包
https://github.com/tech-shrimp/docker_installer/releases 

![mac安装包](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/mac%E5%AE%89%E8%A3%85%E5%8C%85.png)

注意区分CPU架构类型 Intel芯片选择x86_64, 苹果芯片选择arm64
下载好双击安装即可



# 2. Pull镜像

## 方案一 转存到阿里云

使用Github Action将国外的Docker镜像转存到阿里云私有仓库，供国内服务器使用，免费易用

- 支持DockerHub, gcr.io, k8s.io, ghcr.io等任意仓库
- 支持最大40GB的大型镜像
- 使用阿里云的官方线路，速度快

项目地址: https://github.com/tech-shrimp/docker_image_pusher



## 方案二 镜像站

现在只有很少的国内镜像站存活
不保证镜像齐全,且用且珍惜
以下三个镜像站背靠较大的开源项目，优先推荐

| 项目名称 | 项目地址                                        | 加速地址                                                     |
| -------- | ----------------------------------------------- | ------------------------------------------------------------ |
| 1Panel   | https://github.com/1Panel-dev/1Panel/           | [https://docker.1panel.live](https://docker.1panel.live/)    |
| Daocloud | https://github.com/DaoCloud/public-image-mirror | [https://docker.m.daocloud.io](https://docker.m.daocloud.io/) |
| 耗子面板 | https://github.com/TheTNB/panel                 | [https://hub.rat.dev](https://hub.rat.dev/)                  |

#### Linux配置镜像站

```
vim /etc/docker/daemon.json
```

输入下列内容

```
{
    "registry-mirrors": [
        "https://docker.m.daocloud.io",
        "https://docker.1panel.live",
        "https://hub.rat.dev"
    ]
}
```

重启docker

```
systemctl restart docker
```

一些其他操作

```
# 查看源
[root@docker-01 ~]#docker info |grep -i -A 1 mirrors
 Registry Mirrors:
  https://ms9glx6x.mirror.aliyuncs.com/

# 查看docker信息
docker info

# docker-client
which docker

# docker daemon
ps aux |grep docker

# containerd
ps aux|grep containerd
systemctl status containerd
```

#### Windows/Mac配置镜像站

Setting->Docker Engine->添加上换源的那一段，如下图 

![win加速](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/win%E5%8A%A0%E9%80%9F.png)



## 方案三 离线镜像

使用Github Action下载docker离线镜像 https://github.com/wukongdaily/DockerTarBuilder



## 方案四 使用一键脚本

bash -c "$(curl -sSLf https://xy.ggbond.org/xy/docker_pull.sh)" -s 完整镜像名



## 方案五 使用Cloudflare worker 自建镜像加速

https://github.com/cmliu/CF-Workers-docker.io



## 方案六 镜像站（最新）

两个节点

```
dockerpull.com
dockerproxy.cn
```

### 使用方法①

假如拉取原始镜像命令如下

```
docker pull whyour/qinglong:latest
```

仅需在原命令前缀加入加速镜像地址

```
docker pull dockerproxy.cn/whyour/qinglong:latest
```



### 使用方法②

一键设置镜像加速：修改文件 /etc/docker/daemon.json（如果不存在则创建）

```
/etc/docker/daemon.json
```

修改JSON文件 更改为以下内容 然后保存

```
{"registry-mirrors": ["https://dockerproxy.cn"]}
```

保存好之后 执行以下两条命令

```
sudo systemctl daemon-reload    # 重载systemd管理守护进程配置文件

sudo systemctl restart docker    # 重启 Docker 服务
```



# 3. 去哪里找镜像

https://docker.fxxk.dedyn.io/



# 4.关于docker网络

![1661830154498](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/1661830154498.png)



## **虚拟机桥接模式网络接口选择**

![image-20240922195441090](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20240922195441090.png)

**一、Realtek PCIe GbE Family Controller**

这通常是一种由瑞昱（Realtek）公司生产的以太网（有线网络）控制器。它负责主机通过有线网络进行连接，能够提供相对稳定和较高速度的数据传输。

**二、Microsoft Wi-Fi Direct Virtual Adapter #3、QMTAP Adapter V9**

可能与微软的 Wi-Fi Direct 技术相关，用于实现设备之间的直接无线连接。具体功能可能因系统和使用场景而异，但一般不是用于传统的互联网连接。

**三、Intel (R) Wi-Fi 6 AX200 160MHz**

这是英特尔的 Wi-Fi 6 无线网卡。它支持高速的无线网络连接，能够提供较高的带宽和较低的延迟，适用于连接到 Wi-Fi 网络。

**四、Microsoft Wi-Fi Direct Virtual Adapter #4**

类似于前面的 Microsoft Wi-Fi Direct Virtual Adapter #3，与微软的 Wi-Fi Direct 技术相关，具体用途可能因系统和场景而异。



# 5.初体验docker玩法

## Docker 服务端版本

```bash
[root@docker-01 ~]#docker version
Client: Docker Engine - Community
 Version:           20.10.17
 API version:       1.41
 Go version:        go1.17.11
 Git commit:        100c701
 Built:             Mon Jun  6 23:05:12 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.17
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.17.11
  Git commit:       a89b842
  Built:            Mon Jun  6 23:03:33 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.7
  GitCommit:        0197261a30bf81f1ee8e6a4dd2dea0ef95d67ccb
 runc:
  Version:          1.1.3
  GitCommit:        v1.1.3-0-g6724737
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```



## 启动第一个容器

```bash
# 1.这个过程是下载镜像，运行容器实例
[root@docker-01 ~]#docker run alpine /bin/echo "Hello docker by www.yuchaoit.cn"
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
59bf1c3509f3: Pull complete 
Digest: sha256:21a3deaa0d32a8057914f36584b5288d2e5ecc984380bc0118285c70fa8c9300
Status: Downloaded newer image for alpine:latest
Hello docker by www.yuchaoit.cn
```



## docker官网资源API的文档

```bash
https://docs.docker.com/registry/spec/api/

# 获取tags版本
https://hub.docker.com/v2/repositories/library/nginx/tags
```



## 镜像命令

### 搜索镜像

```bash
1. 选择官网认证标识的
2. 选择stars星星多的

# 搜索命令
docker search centos

# 获取镜像版本号
curl -L -s "https://registry.hub.docker.com/v2/repositories/library/mysql/tags?page_size=1024" | jq '.results[]["name"]' | sed 's/\"//g' | sort -u

# page_size参数允许你控制单次请求返回的数据量。如果你需要处理大量的标签，并且想要减少请求的次数，那么使用带有page_size参数的URL会更有效。如果没有特别的需求，使用默认的分页大小通常就足够了。
```



### 下载镜像

```bash
# 如果不加版本，默认下载的就是latest最新版的
[root@docker-01 ~]#docker pull centos
Using default tag: latest
latest: Pulling from library/centos
a1d0c7532777: Pull complete 
Digest: sha256:a27fd8080b517143cbbbab9dfb7c8571c40d67d534bbdee55bd6c473f432b177
Status: Downloaded newer image for centos:latest
docker.io/library/centos:latest


# 下载指定版本 centos7.9.2009
[root@docker-01 ~]#docker pull  centos:7.9.2009
7.9.2009: Pulling from library/centos
2d473b07cdd5: Pull complete 
Digest: sha256:9d4bcbbb213dfd745b58be38b13b996ebb5ac315fe75711bd618426a630e0987
Status: Downloaded newer image for centos:7.9.2009
docker.io/library/centos:7.9.2009


# 下载一个busybox镜像（提供了诸多linux调试工具的容器）
[root@docker-01 ~]#docker pull busybox:1.29
1.29: Pulling from library/busybox
b4a6e23922dd: Pull complete 
Digest: sha256:8ccbac733d19c0dd4d70b4f0c1e12245b5fa3ad24758a11035ee505c629c0796
Status: Downloaded newer image for busybox:1.29
docker.io/library/busybox:1.29
```



### 查看镜像列表

```bash
[root@docker-01 ~]#docker images
REPOSITORY   TAG        IMAGE ID       CREATED         SIZE
alpine       latest     c059bfaa849c   8 months ago    5.59MB
centos       7.9.2009   eeb6ee3f44bd   11 months ago   204MB
centos       latest     5d0da3dc9764   11 months ago   231MB

[root@docker-01 ~]#docker image ls
REPOSITORY   TAG        IMAGE ID       CREATED         SIZE
alpine       latest     c059bfaa849c   8 months ago    5.59MB
centos       7.9.2009   eeb6ee3f44bd   11 months ago   204MB
centos       latest     5d0da3dc9764   11 months ago   231MB
```



### 运行镜像，生成容器

```bash
1. 下载nginx镜像
curl -s https://registry.hub.docker.com/v1/repositories/nginx/tags | jq

[root@docker-01 ~]#curl -I tabao.com
HTTP/1.1 404 Not Found
Server: nginx/1.17.9
Date: Sun, 21 Aug 2022 09:00:57 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 18
Connection: keep-alive

[root@docker-01 ~]#docker pull nginx:1.17.9
1.17.9: Pulling from library/nginx
123275d6e508: Pull complete 
9a5d769f04f8: Pull complete 
faad4f49180d: Pull complete 
Digest: sha256:88ea86df324b03b3205cbf4ca0d999143656d0a3394675630e55e49044d38b50
Status: Downloaded newer image for nginx:1.17.9
docker.io/library/nginx:1.17.9


2.运行镜像，生成容器
# 备注，若镜像不存在，docker也会自动pull下载镜像
[root@docker-01 ~]#docker run -d -p 80:80 nginx:1.17.9
132aa58824a603e307b618a8b0c3da52054b05a54c22f550e355eac143393e8a

# 参数解释
docker run 参数     镜像
-d 后台运行
-p 端口映射
nginx:1.17.9  镜像名
# 默认我们机器上是没有docker镜像的，docker run是在运行时，寻找且自动下载镜像


3.查看镜像列表
[root@docker-01 ~]#docker images
REPOSITORY   TAG        IMAGE ID       CREATED         SIZE
alpine       latest     c059bfaa849c   8 months ago    5.59MB
centos       7.9.2009   eeb6ee3f44bd   11 months ago   204MB
centos       latest     5d0da3dc9764   11 months ago   231MB
nginx        1.17.9     5a8dfb2ca731   2 years ago     127MB
busybox      1.29       758ec7f3a1ee   3 years ago     1.15MB


4.查看运行中的容器列表
[root@docker-01 ~]#docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                               NAMES
132aa58824a6   nginx:1.17.9   "nginx -g 'daemon of…"   56 seconds ago   Up 55 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   pensive_lewin
```

5.访问容器中的nginx 1.17.9

![image-20220821170534211](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20220821170534211.png)

> 看到这个页面，就说明你正确安装了docker，且运行了第一个nginx容器服务。



# 6.一张图玩懂docker操作

![image-20220821171918134](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20220821171918134.png)



# 7.docker镜像详解

一个完成的Docker镜像可以支撑容器的运行，镜像提供文件系统

```
# 下载ubuntu系统镜像
[root@docker-01 ~]#docker pull ubuntu
Using default tag: latest
latest: Pulling from library/ubuntu
7b1a6ab2e44d: Pull complete 
Digest: sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest
```

## 内核与发行版

传统的虚拟机安装操作系统所提供的系统镜像，包含两部分：

Linux内核部分

```
[root@docker-01 ~]#uname -r
3.10.0-862.el7.x86_64
```

系统发行版部分

```
[root@docker-01 ~]#cat /etc/os-release 
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```

而docker镜像是不包含内核的，只是下载了某个发行版部分。

```
# 获取centos7.5发行版本
[root@docker01 ~]# docker search centos:7.5

# 获取mysql5.6
[root@docker01 ~]# docker search mysql:5.6

# 查看刚才运行的nginx容器，它是什么发行版
[root@docker-01 ~]#docker exec -it 132 bash
root@132aa58824a6:/# cat /etc/os-release 
PRETTY_NAME="Debian GNU/Linux 10 (buster)"
NAME="Debian GNU/Linux"
VERSION_ID="10"
VERSION="10 (buster)"
VERSION_CODENAME=buster
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
root@132aa58824a6:/# 


# 容器内核，和宿主机内核一样
root@132aa58824a6:/# cat /proc/version 
Linux version 3.10.0-862.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-28) (GCC) ) #1 SMP Fri Apr 20 16:44:24 UTC 2018
root@132aa58824a6:/# 
root@132aa58824a6:/# exit
exit
[root@docker-01 ~]#
[root@docker-01 ~]#cat /proc/version 
Linux version 3.10.0-862.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-28) (GCC) ) #1 SMP Fri Apr 20 16:44:24 UTC 2018


# /proc文件系统，它不是普通的文件系统，而是系统内核的映像，也就是说，该目录中的文件是存放在系统内存之中的，它以文件系统的方式为访问系统内核数据的操作提供接口。而我们使用命令“uname -a"的信息就是从该文件获取的，当然用方法二的命令直接查看它的内容也可以达到同等效果.另外，加上参数"a"是获得详细信息，如果不加参数为查看系统名称。
```



## docker镜像定义

我们如果自定义镜像，刚才超哥已经和大家说了，docker镜像不包含linux内核，和宿主机共用。

我们如果想要定义一个mysql5.6镜像，我们会这么做

- 获取基础镜像，选择一个发行版平台（ubutu，centos）
- 在centos镜像中安装mysql5.6软件

导出镜像，可以命名为mysql:5.6镜像文件。

从这个过程，我们可以感觉出这是一层一层的添加的，docker镜像的层级概念就出来了，底层是centos镜像，上层是mysql镜像，centos镜像层属于父镜像。

![image-20220821175036741](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20220821175036741.png)



### 为什么要有docker镜像

其实就是将业务代码运行的环境，整体打包为单个的文件，就是docker镜像。



### 如何创建docker镜像

现在docker官方共有仓库里面有大量的镜像，所以最基础的镜像，我们可以在公有仓库直接拉取，因为这些镜像都是原厂维护，可以得到即使的更新和修护。



### dockerfile

我们如果想去定制这些镜像，我们可以去编写Dockerfile，然后重新bulid，最后把它打包成一个镜像

这种方式是最为推荐的方式包括我们以后去企业当中去实践应用的时候也是推荐这种方式。

<img src="02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20220821173006481.png" alt="image-20220821173006481" style="zoom:50%;" />



### docker commit

当然还有另外一种方式，就是通过镜像启动一个容器，然后进行操作，最终通过commit这个命令commit一个镜像。



## docker镜像分层结构

```
可以基于 docker history 查看镜像每一层信息
[root@docker-01 ~]#docker history nginx:1.17.9 
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
5a8dfb2ca731   2 years ago   /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B        
<missing>      2 years ago   /bin/sh -c #(nop)  STOPSIGNAL SIGTERM           0B        
<missing>      2 years ago   /bin/sh -c #(nop)  EXPOSE 80                    0B        
<missing>      2 years ago   /bin/sh -c ln -sf /dev/stdout /var/log/nginx…   22B       
<missing>      2 years ago   /bin/sh -c set -x     && addgroup --system -…   57.6MB    
<missing>      2 years ago   /bin/sh -c #(nop)  ENV PKG_RELEASE=1~buster     0B        
<missing>      2 years ago   /bin/sh -c #(nop)  ENV NJS_VERSION=0.3.9        0B        
<missing>      2 years ago   /bin/sh -c #(nop)  ENV NGINX_VERSION=1.17.9     0B        
<missing>      2 years ago   /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B        
<missing>      2 years ago   /bin/sh -c #(nop)  CMD ["bash"]                 0B        
<missing>      2 years ago   /bin/sh -c #(nop) ADD file:865f9041e12eb341f…   69.2MB
```

命令查看镜像的分层关系

![image-20220821175353123](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20220821175353123.png)



<img src="02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20240903183649594.png" alt="image-20240903183649594" style="zoom:200%;" />



## base镜像

base镜像是指如各种Linux发行版，如centos，ubuntu，Debian，alpine等。

```
[root@docker-01 ~]#docker run -it alpine /bin/sh
/ # cat /etc/os-release 
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.15.0
PRETTY_NAME="Alpine Linux v3.15"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://bugs.alpinelinux.org/"

提示

1. alpine镜像是用于docker镜像体积优化里的一个重要基础镜像，只有几兆，而其他主流base image至少都在百兆。
2. alpine采用busybox套件，而centos等基础镜像安装的依赖较多。

对于base镜像来说，只需要提供基本的/dev/ /proc /bin 等系统运行必须得文件，因此很小，提供基本命令，底层直接用宿主机的内核kernel。
```



## docker为什么分层镜像

镜像分享一大好处就是共享资源，例如有多个镜像都来自于同一个base镜像，那么在docker 主机只需要存储一份base镜像。

内存里也只需要加载一份base镜像，即可为多个容器服务。

> 即使多个容器共享一个base镜像，某个容器修改了base镜像的内容，例如修改/etc/下配置文件，其他容器的/etc/下内容是不会被修改的，修改动作只限制在单个容器内，这就是容器的写入时复制特性（Copy-on-write）。

![image-20220821181523881](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20220821181523881.png)



## 联合文件系统UnionFS

```
[root@docker-01 ~]#docker pull mysql:5.7
5.7: Pulling from library/mysql
72a69066d2fe: Pull complete 
93619dbc5b36: Pull complete 
99da31dd6142: Pull complete 
626033c43d70: Pull complete 
37d5d7efb64e: Pull complete 
ac563158d721: Pull complete 
d2ba16033dad: Pull complete 
0ceb82207cd7: Pull complete 
37f2405cae96: Pull complete 
e2482e017e53: Pull complete 
70deed891d42: Pull complete 
Digest: sha256:f2ad209efe9c67104167fc609cca6973c8422939491c9345270175a300419f94
Status: Downloaded newer image for mysql:5.7
docker.io/library/mysql:5.7
```

![image-20220821182726794](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20220821182726794.png)



# 8.docker镜像实践操作

## 镜像详细命令

```bash
1.获取镜像，docker hub里有大量高质量的镜像
[root@docker01 ~]# docker pull centos  # 默认标签tag是centos:latest

[root@docker01 ~]# docker pull  centos:7.2.1511  # 指定版本下载

2.查看所有镜像
[root@docker01 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              4bb46517cac3        3 weeks ago         133MB
centos              latest              0d120b6ccaa8        3 weeks ago         215MB
centos              7.2.1511            9aec5c5fe4ba        17 months ago       195MB


3.docker本地镜像存储在宿主机的目录查看

# 基于docker info 查看 docker数据目录

[root@docker-01 ~]#docker images
REPOSITORY   TAG        IMAGE ID       CREATED         SIZE
mysql        5.7        c20987f18b13   8 months ago    448MB
alpine       latest     c059bfaa849c   8 months ago    5.59MB
ubuntu       latest     ba6acccedd29   10 months ago   72.8MB
centos       7.9.2009   eeb6ee3f44bd   11 months ago   204MB
centos       latest     5d0da3dc9764   11 months ago   231MB
nginx        1.17.9     5a8dfb2ca731   2 years ago     127MB
busybox      1.29       758ec7f3a1ee   3 years ago     1.15MB
[root@docker-01 ~]#

[root@docker-01 ~]#docker info |grep -i 'root dir'
 Docker Root Dir: /var/lib/docker

# 查看数据目录下有啥

[root@docker-01 ~]#ll /var/lib/docker/
total 12
drwx--x--x  4 root root  120 Aug 21 20:46 buildkit
drwx--x--- 10 root root 4096 Aug 22 02:23 containers
drwx------  3 root root   22 Aug 21 20:46 image
drwxr-x---  3 root root   19 Aug 21 20:46 network
drwx--x--- 38 root root 4096 Aug 22 02:23 overlay2
drwx------  4 root root   32 Aug 21 20:46 plugins
drwx------  2 root root    6 Aug 21 20:52 runtimes
drwx------  2 root root    6 Aug 21 20:46 swarm
drwx------  2 root root    6 Aug 22 02:17 tmp
drwx------  2 root root    6 Aug 21 20:46 trust
drwx-----x  6 root root 4096 Aug 22 02:23 volumes


# 镜像数据存储在
[root@docker-01 ~]#docker info |grep -i 'storage'
 Storage Driver: overlay2

# 查看大小
[root@docker-01 ~]#du -sh /var/lib/docker/overlay2/
1.3G    /var/lib/docker/overlay2/


# imagesdb该目录存放镜像，容器的相关信息，是一个json文件

[root@docker-01 ~]#ls /var/lib/docker/image/overlay2/imagedb/content/sha256/
5a8dfb2ca7312ee39433331b11d92f45bb19d7809f7c0ff19e1d01a2c131e959  ba6acccedd2923aee4c2acc6a23780b14ed4b8a5fa4e14e252a23b846df9b6c1  eeb6ee3f44bd0b5103bb561b4c16bcb82328cfe5809ab675bb17ab3a16c517c9
5d0da3dc976460b72c77d94c8a1ad043720b0416bfc16c52c45d4847e53fadb6  c059bfaa849c4d8e4aecaeb3a10c2d9b3d85f5165c66ad3a4d937758128c4d18
758ec7f3a1ee85f8f08399b55641bfb13e8c1109287ddc5e22b68c3d653152ee  c20987f18b130f9d144c9828df630417e2a9523148930dc3963e9d0dab302a76

# 以基础镜像运行一个容器，添加参数
-i  Keep STDIN open even if not attached
-t  Allocate a pseudo-TTY 
--rm  Automatically remove the container when it exits
--name   Assign a name to the container
bash 指定解释器

[root@docker-01 ~]#docker run -it --name www.yuchaoit.cn_centos7 centos:7.9.2009 bash
[root@8a6490e3d5a2 /]# 
[root@8a6490e3d5a2 /]# cat /etc/redhat-release 
CentOS Linux release 7.9.2009 (Core)

[root@8a6490e3d5a2 /]# ps -ef
UID         PID   PPID  C STIME TTY          TIME CMD
root          1      0  0 18:41 pts/0    00:00:00 bash
root         17      1  0 18:42 pts/0    00:00:00 ps -ef


# 运行centos8容器
[root@docker-01 ~]#docker run -it --name www.yuchaoit.cn_centos8 centos bash
[root@252c828de18d /]# cat /etc/redhat-release 
CentOS Linux release 8.4.2105
```



## 镜像增删改查管理

```bash
[root@docker-01 ~]#docker images
REPOSITORY   TAG        IMAGE ID       CREATED         SIZE
mysql        5.7        c20987f18b13   8 months ago    448MB
alpine       latest     c059bfaa849c   8 months ago    5.59MB
ubuntu       latest     ba6acccedd29   10 months ago   72.8MB
centos       7.9.2009   eeb6ee3f44bd   11 months ago   204MB
centos       latest     5d0da3dc9764   11 months ago   231MB
nginx        1.17.9     5a8dfb2ca731   2 years ago     127MB
busybox      1.29       758ec7f3a1ee   3 years ago     1.15MB
[root@docker-01 ~]#

列表展示了仓库名，标签，镜像ID，创建时间，占用空间

2.docker镜像体积
docker镜像是多层存储结构，可以继承，复用。因此不同的镜像也会使用相同的base images，从而使用些共同的层layer。
因此docker使用联合文件系统，相同的层只需要保存一份，实际镜像占用硬盘空间要比列表镜像小的多。
```

### none镜像

none镜像（dangline image 虚悬镜像）

出现none镜像的原因：

- 在docker hub上镜像如果更新后，名称变化，用户再docker pull就会出现该情况
- docker build时候，镜像名重复，也会导致新旧镜像同名，旧镜像名称被取消，出现none

```
一般用docker tag解决即可
```



### 列出镜像

```
1.根据名字列出镜像
[root@docker-01 ~]#docker images  cent*
REPOSITORY   TAG        IMAGE ID       CREATED         SIZE
centos       7.9.2009   eeb6ee3f44bd   11 months ago   204MB
centos       latest     5d0da3dc9764   11 months ago   231MB
centos       7.7.1908   08d05d1d5859   2 years ago     204MB
centos       7.6.1810   f1cb7c7d58b7   3 years ago     202MB

2.查看指定镜像
[root@docker-01 ~]#docker images nginx
nginx         nginx:1.17.9  
[root@docker-01 ~]#docker images nginx:1.17.9 
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        1.17.9    5a8dfb2ca731   2 years ago   127MB
[root@docker-01 ~]#


3.只查看镜像id，id就代表该镜像了
[root@docker-01 ~]#docker images -q
c20987f18b13
c059bfaa849c
ba6acccedd29
eeb6ee3f44bd
5d0da3dc9764
5a8dfb2ca731
08d05d1d5859
f1cb7c7d58b7
758ec7f3a1ee
[root@docker-01 ~]#


4.格式化输出docker信息
[root@docker-01 ~]#docker images --format "{{.ID}}:{{.Repository}}"
c20987f18b13:mysql
c059bfaa849c:alpine
ba6acccedd29:ubuntu
eeb6ee3f44bd:centos
5d0da3dc9764:centos
5a8dfb2ca731:nginx
08d05d1d5859:centos
f1cb7c7d58b7:centos
758ec7f3a1ee:busybox


5.更丰富的格式化
[root@docker-01 ~]#docker images --format "table {{.ID}} {{.Repository}} {{.Tag}}"
IMAGE ID       REPOSITORY   TAG
c20987f18b13   mysql        5.7
c059bfaa849c   alpine       latest
ba6acccedd29   ubuntu       latest
eeb6ee3f44bd   centos       7.9.2009
5d0da3dc9764   centos       latest
5a8dfb2ca731   nginx        1.17.9
08d05d1d5859   centos       7.7.1908
f1cb7c7d58b7   centos       7.6.1810
758ec7f3a1ee   busybox      1.29

# 也可以是
docker images --format "table {{.ID}} {{.Repository}} {{.Tag}}" | column -t


6.格式化是docker信息提取的高级语法，需要学习下go的template语法
# 基于--format="{{json .}}" 拿到详细字段，即可格式化

[root@docker-01 ~]#docker images --format="{{json .}}"

[root@docker-01 ~]#docker images --format="{{.CreatedAt}} {{.ID}}  {{.Repository}} {{.Size}} {{.Tag}}" |column -t
2021-12-21  10:56:51  +0800  CST  c20987f18b13  mysql    448MB   5.7
2021-11-25  04:19:40  +0800  CST  c059bfaa849c  alpine   5.59MB  latest
2021-10-16  08:37:47  +0800  CST  ba6acccedd29  ubuntu   72.8MB  latest
2021-09-16  02:20:23  +0800  CST  eeb6ee3f44bd  centos   204MB   7.9.2009
2021-09-16  02:20:05  +0800  CST  5d0da3dc9764  centos   231MB   latest
2020-04-16  18:09:38  +0800  CST  5a8dfb2ca731  nginx    127MB   1.17.9
2019-11-12  08:21:02  +0800  CST  08d05d1d5859  centos   204MB   7.7.1908
2019-03-15  05:20:29  +0800  CST  f1cb7c7d58b7  centos   202MB   7.6.1810
2018-12-26  16:20:42  +0800  CST  758ec7f3a1ee  busybox  1.15MB  1.29
```



### 删除本地镜像

```perl
# docker stop 容器id前三位/CONTAINER ID/NAMES只能暂停容器运行，这个容器名以及id还是存在的，会导致新容器无法使用这个名字
[root@server01 ~]# docker stop 7fb
7fb
[root@server01 ~]# 
[root@server01 ~]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED             STATUS                     PORTS     NAMES
7fb29ae65826   nginx:latest   "/docker-entrypoint.…"   52 minutes ago      Exited (0) 5 seconds ago             nginx01
04af9fbf9dcd   nginx:1.19.7   "/docker-entrypoint.…"   About an hour ago   Up About an hour           80/tcp    sharp_villani


# 因此想要彻底干掉一个容器，需要先停止服务，再删除
[root@server01 ~]# docker rm nginx01
nginx01
[root@server01 ~]# 
[root@server01 ~]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED             STATUS             PORTS     NAMES
04af9fbf9dcd   nginx:1.19.7   "/docker-entrypoint.…"   About an hour ago   Up About an hour   80/tcp    sharp_villani


# 删除镜像，可以用 ID，名字，摘要等
[root@docker-01 ~]#docker rmi centos:7.7.1908 
Untagged: centos:7.7.1908
Untagged: centos@sha256:50752af5182c6cd5518e3e91d48f7ff0cba93d5d760a67ac140e2d63c4dd9efc
Deleted: sha256:08d05d1d5859ebcfb3312d246e2082e46cb307f0e896c9ac097185f0b0b19e56
Deleted: sha256:034f282942cd6c3abf9384601a57f080f8f75cc7f58527db8e07573d9d14ab46


# 删除镜像，要先干掉使用该镜像的容器（无论是否存活）
# 不加tag版本的话，默认latest
[root@docker-01 ~]#docker rmi centos
Error response from daemon: conflict: unable to remove repository reference "centos" (must force) - container 252c828de18d is using its referenced image 5d0da3dc9764
[root@docker-01 ~]#


# 清理挂掉的容器实例记录
docker rm `docker ps -aq`


# 根据id删除（最短3位）
[root@docker-01 ~]#docker rmi f1c
Untagged: centos:7.6.1810
Untagged: centos@sha256:62d9e1c2daa91166139b51577fe4f4f6b4cc41a3a2c7fc36bd895e2a17a3e4e6
Deleted: sha256:f1cb7c7d58b73eac859c395882eec49d50651244e342cd6c68a5c7809785f427
Deleted: sha256:89169d87dbe2b72ba42bfbb3579c957322baca28e03a1e558076542a1c1b2b4a


# 清理所有镜像（危险命令）
# 删除命令，包括了删除，以及取消tag两个步骤，删除所有镜像，可能会导致依赖这些镜像的容器出现问题。
docker rmi `docker images -aq`
# 只删除容器，不会影响镜像库。
docker rm -f $(docker ps -aq)


# 于超老师这里未彻底删除，因为有一个nginx容器在运行
# 得 停止删除容器 > 删除镜像
[root@docker-01 ~]#docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        1.17.9    5a8dfb2ca731   2 years ago   127MB
[root@docker-01 ~]#
[root@docker-01 ~]#docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED       STATUS       PORTS                               NAMES
132aa58824a6   nginx:1.17.9   "nginx -g 'daemon of…"   2 hours ago   Up 2 hours   0.0.0.0:80->80/tcp, :::80->80/tcp   pensive_lewin


# 提示，不要随便用 docker rmi -f 强制参数
```



### 导出、导入镜像

常用于公司的离线环境使用镜像

默认导出的是tar归档文件

```
# 导出镜像
[root@docker-01 ~]#docker save nginx:1.17.9  > /images_save_all/www.yuchaoit.cn_nginx:1.17.9.tar
[root@docker-01 ~]#du -h /images_save_all/www.yuchaoit.cn_nginx\:1.17.9.tar 
125M    /images_save_all/www.yuchaoit.cn_nginx:1.17.9.tar


# 导入镜像
# 环境清理
[root@docker-01 ~]#docker stop `docker ps -aq`
132aa58824a6
[root@docker-01 ~]#
[root@docker-01 ~]#docker rm `docker ps -aq`
132aa58824a6
[root@docker-01 ~]#
[root@docker-01 ~]#docker rmi nginx:1.17.9 
Untagged: nginx:1.17.9
Untagged: nginx@sha256:88ea86df324b03b3205cbf4ca0d999143656d0a3394675630e55e49044d38b50
Deleted: sha256:5a8dfb2ca7312ee39433331b11d92f45bb19d7809f7c0ff19e1d01a2c131e959
Deleted: sha256:eede83f79a434879440e1f6f6f98a135b38057a35ddcdace715ae1bddcd7a884
Deleted: sha256:fa994cfd7aeedcd46b70cf30fea0ccf9f59f990bbb86bfa9b7c02d7ff2a833eb
Deleted: sha256:b60e5c3bcef2f42ec42648b3acf7baf6de1fa780ca16d9180f3b4a3f266fe7bc

[root@docker-01 ~]#docker images
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE


# 导入本地镜像
[root@docker-01 ~]#docker load < /images_save_all/www.yuchaoit.cn_nginx\:1.17.9.tar 
b60e5c3bcef2: Loading layer [==================================================>]  72.49MB/72.49MB
0e07021aa61a: Loading layer [==================================================>]  58.11MB/58.11MB
351816b95c49: Loading layer [==================================================>]  3.584kB/3.584kB
Loaded image: nginx:1.17.9
[root@docker-01 ~]#
[root@docker-01 ~]#du -sh /var/lib/docker/overlay2/
131M    /var/lib/docker/overlay2/
[root@docker-01 ~]#
[root@docker-01 ~]#docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
nginx        1.17.9    5a8dfb2ca731   2 years ago   127MB

[root@docker-01 ~]#file /images_save_all/www.yuchaoit.cn_nginx\:1.17.9.tar 
/images_save_all/www.yuchaoit.cn_nginx:1.17.9.tar: POSIX tar archive
```



### 查看镜像详细信息

```
[root@docker-01 /images_save_all]#docker inspect nginx:1.17.9  | jq 

# 查看无论是镜像，还是容器的详细信息，都是维护容器的重要手段
```



# 9.docker容器管理实践

## 启动容器

`docker run`等于创建+启动

**注意：容器内的进程必须处于前台运行状态，否则容器就会直接退出**

我们运行如centos基础镜像，没有运行任何程序，因此容器直接挂掉

```
[root@docker-01 /images_save_all]#docker run centos:7
Unable to find image 'centos:7' locally
7: Pulling from library/centos
2d473b07cdd5: Pull complete 
Digest: sha256:9d4bcbbb213dfd745b58be38b13b996ebb5ac315fe75711bd618426a630e0987
Status: Downloaded newer image for centos:7

# 交互式的运行，可以进入容器空间
[root@docker-01 /images_save_all]#docker run -it centos:7 bash

[root@94963614be20 /]# cat /etc/redhat-release 
CentOS Linux release 7.9.2009 (Core)


# 非交互式运行，容器会直接挂掉
[root@docker-01 /images_save_all]#docker run  centos:7 
[root@docker-01 /images_save_all]#
[root@docker-01 /images_save_all]#docker run  centos:7 


# 查看容器历史记录
[root@docker-01 /images_save_all]#docker ps -a
CONTAINER ID   IMAGE      COMMAND       CREATED              STATUS                          PORTS     NAMES
baca62de6cbe   centos:7   "/bin/bash"   2 seconds ago        Exited (0) 1 second ago                   kind_jepsen
fe2e571a5d6e   centos:7   "/bin/bash"   4 seconds ago        Exited (0) 4 seconds ago                  objective_allen
94963614be20   centos:7   "bash"        About a minute ago   Exited (0) 10 seconds ago                 determined_meitner
900c51bd3677   centos:7   "/bin/bash"   About a minute ago   Exited (0) About a minute ago             youthful_blackburn
```



## 运行可以活着的容器

```
-d 对于宿主机，后台运行容器
-p 端口映射
[root@docker-01 /images_save_all]#docker run -d -p 80:80 nginx:1.17.9 
4bc32e061a6ebd5b347f80b33957d6ccd73f80d07030d17ef163570fe610a2db


# 直接访问宿主机即可
[root@docker-01 /images_save_all]#curl 10.0.0.200 -I
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Sun, 21 Aug 2022 23:15:27 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: "5e5e6a8f-264"
Accept-Ranges: bytes

# 宿主机上访问容器ip也可以
[root@docker-01 /images_save_all]#docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED              STATUS              PORTS                               NAMES
4bc32e061a6e   nginx:1.17.9   "nginx -g 'daemon of…"   About a minute ago   Up About a minute   0.0.0.0:80->80/tcp, :::80->80/tcp   awesome_wilbur
[root@docker-01 /images_save_all]#

# 提取容器ip
[root@docker-01 /images_save_all]#docker inspect 4bc  --format "{{.NetworkSettings.IPAddress}}"
172.17.0.2

# 访问容器
[root@docker-01 /images_save_all]#curl 172.17.0.2 -I
HTTP/1.1 200 OK
Server: nginx/1.17.9
Date: Sun, 21 Aug 2022 23:20:18 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 03 Mar 2020 14:32:47 GMT
Connection: keep-alive
ETag: "5e5e6a8f-264"
Accept-Ranges: bytes
```



## 运行容器且指定名字

```
# 开发要求提供一个redis容器，进行测试使用

[root@docker-01 /images_save_all]#docker run --name www.yuchaoit.cn_redis5 -d -p 16379:6379 redis:5 
Unable to find image 'redis:5' locally
5: Pulling from library/redis
a2abf6c4d29d: Pull complete 
c7a4e4382001: Pull complete 
4044b9ba67c9: Pull complete 
106f2419edf3: Pull complete 
9772114922b9: Pull complete 
63031aedd0c4: Pull complete 
Digest: sha256:a30e893aa92ea4b57baf51e5602f1657ec5553b65e62ba4581a71e161e82868a
Status: Downloaded newer image for redis:5
87a1f13bf622511591d049628608c5506ea824cdb92ce64cc3a012a3399036eb

检查容器记录

[root@docker-01 /images_save_all]#docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                         NAMES
87a1f13bf622   redis:5        "docker-entrypoint.s…"   7 seconds ago   Up 7 seconds   0.0.0.0:16379->6379/tcp, :::16379->6379/tcp   www.yuchaoit.cn_redis5
4bc32e061a6e   nginx:1.17.9   "nginx -g 'daemon of…"   8 minutes ago   Up 8 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp             awesome_wilbur
```



## 停止容器（并非删除）

```
[root@docker-01 /images_save_all]#docker stop www.yuchaoit.cn_redis5 
www.yuchaoit.cn_redis5
[root@docker-01 /images_save_all]#
[root@docker-01 /images_save_all]#docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                               NAMES
4bc32e061a6e   nginx:1.17.9   "nginx -g 'daemon of…"   9 minutes ago   Up 9 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp   awesome_wilbur
[root@docker-01 /images_save_all]#


启动容器
[root@docker-01 /images_save_all]#docker start www.yuchaoit.cn_redis5 
www.yuchaoit.cn_redis5
[root@docker-01 /images_save_all]#docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED              STATUS          PORTS                                         NAMES
87a1f13bf622   redis:5        "docker-entrypoint.s…"   About a minute ago   Up 1 second     0.0.0.0:16379->6379/tcp, :::16379->6379/tcp   www.yuchaoit.cn_redis5
4bc32e061a6e   nginx:1.17.9   "nginx -g 'daemon of…"   10 minutes ago       Up 10 minutes   0.0.0.0:80->80/tcp, :::80->80/tcp             awesome_wilbur
[root@docker-01 /images_save_all]#
```



## 监控容器资源状态

```
[root@docker-01 /images_save_all]#docker stats www.yuchaoit.cn_redis5
```



## 进入容器空间

```
exec命令
要结合-t -i参数，开启终端，交互式访问容器空间

[root@docker-01 /images_save_all]#docker exec -it www.yuchaoit.cn_redis5 bash
root@87a1f13bf622:/data# 
root@87a1f13bf622:/data# 
root@87a1f13bf622:/data# cat /etc/os-release 
PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"
NAME="Debian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
root@87a1f13bf622:/data#
```



## 访问容器应用(redis)

由于我们做了端口映射，可以基于宿主机的端口访问

```
[root@docker-01 /images_save_all]#redis-cli -h 10.0.0.200 -p 16379
10.0.0.200:16379> ping
PONG
10.0.0.200:16379> set name chaoge666
OK
10.0.0.200:16379> exit


[root@docker-01 /images_save_all]#redis-cli -h 172.17.0.3 -p 6379 
172.17.0.3:6379> 
172.17.0.3:6379> dbsize
(integer) 1
172.17.0.3:6379> get name
"chaoge666"
172.17.0.3:6379>
```



## 查看容器内日志

```
[root@docker-01 /images_save_all]#docker logs  www.yuchaoit.cn_redis5
```



## 删除容器

```
[root@docker-01 /images_save_all]#docker stop www.yuchaoit.cn_redis5 
www.yuchaoit.cn_redis5
[root@docker-01 /images_save_all]#docker rm www.yuchaoit.cn_redis5 
www.yuchaoit.cn_redis5
[root@docker-01 /images_save_all]#docker ps -a
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS                      PORTS                               NAMES
4bc32e061a6e   nginx:1.17.9   "nginx -g 'daemon of…"   24 minutes ago   Up 24 minutes               0.0.0.0:80->80/tcp, :::80->80/tcp   awesome_wilbur
baca62de6cbe   centos:7       "/bin/bash"              25 minutes ago   Exited (0) 25 minutes ago                                       kind_jepsen
fe2e571a5d6e   centos:7       "/bin/bash"              25 minutes ago   Exited (0) 25 minutes ago                                       objective_allen
94963614be20   centos:7       "bash"                   26 minutes ago   Exited (0) 25 minutes ago                                       determined_meitner
900c51bd3677   centos:7       "/bin/bash"              27 minutes ago   Exited (0) 27 minutes ago                                       youthful_blackburn
[root@docker-01 /images_save_all]#
```



## 查看容器记录（挂掉，运行中）

```
[root@docker-01 /images_save_all]#docker ps

[root@docker-01 /images_save_all]#docker ps -a
```



## 批量干掉容器进程

-q 只显示id

-a 显示所有记录

```
[root@docker-01 /images_save_all]#docker stop `docker ps -q`
4bc32e061a6e
[root@docker-01 /images_save_all]#
[root@docker-01 /images_save_all]#
[root@docker-01 /images_save_all]#docker rm `docker ps -aq`
4bc32e061a6e
baca62de6cbe
fe2e571a5d6e
94963614be20
900c51bd3677
```



# 练习

```
1. 下载mysql5.7.38镜像，确保可以远程访问，创建库表
2. 下载redis最新镜像，确保可以远程访问，读写key
3. 下载nginx 1.21.5 ，确保可以远程访问，修改首页内容为，"云原生！我来了！"
4. 下载wordpress最新镜像，运行，确保可以访问，发表文章。
```



# 实践：新建一个自定义镜像并推送到个人仓库

### 1.创建自定义镜像

```bash
# 1.拉取镜像，作为模板
docker pull centos:7.4.1708 


# 2.以bash进程创建一个容器
docker run -it image_name/id bash

# 进入容器
docker exec -it container_id bash


# 3.清空原有yum源
rm -f /etc/yum.repos.d/*


# 4.配置好yum源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo


# 5.安装软件
yum install vim net-tools nginx -y


# 6.清空yum缓存
yum clean all


# 7.退出到宿主机，把处理完毕的容器另存为镜像
docker commit  container_id  tonydu789/nginx_vim_net-tools_centos7.9


# 对比新老镜像的分层信息
# 老镜像
[root@server01 ~]# docker history centos:7.9.2009 
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
eeb6ee3f44bd   3 years ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B        		   # ro层
<missing>      3 years ago   /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B        		   # ro层
<missing>      3 years ago   /bin/sh -c #(nop) ADD file:b3ebbe8bd304723d4…   204MB     		   # ro层

# 新镜像
[root@server01 ~]# docker history tonydu789/nginx_vim_net-tools_centos7.9:latest 
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
0a11d895067e   10 days ago   bash                                            114MB     		   # 对于容器来说的rw层
<missing>      3 years ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B        		   # ro层
<missing>      3 years ago   /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B        		   # ro层
<missing>      3 years ago   /bin/sh -c #(nop) ADD file:b3ebbe8bd304723d4…   204MB     		   # ro层
```



### 2.创建私有仓库

![image-20240923151955112](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20240923151955112.png)



![image-20240923150652515](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20240923150652515.png)



### 3.push镜像到远程仓库

这里我的账号为tonydu789，创建了一个名称为test_repository的仓库，创建好仓库后我们就可以推送本地镜像到这个仓库里了。下面我通过一个实例来演示一下如何推送镜像到自己的仓库中。

在推送镜像仓库前，我们需要使用`docker login`命令先登录一下镜像服务器，因为只有已经登录的用户才可以推送镜像到仓库。

```bash
[root@iZ7xvb2aw7a3ekhvnen6u1Z ~]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: tonydu789
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

使用`docker login`命令登录镜像服务器，这时 Docker 会要求我们输入用户名和密码，输入我们刚才注册的账号和密码，看到`Login Succeeded`表示登录成功。登录成功后就可以推送镜像到自己创建的仓库了。

使用`docker image`命令查看当前服务器存在的镜像。

```bash
[root@iZ7xvb2aw7a3ekhvnen6u1Z ~]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
fishexam     latest    bbc5ded42e1a   40 hours ago   631MB
mysql        5.7.31    42cdba9f1b08   2 years ago    448MB
```

在本地镜像推送到自定义仓库前，我们需要先把镜像“重命名”一下，才能正确推送到自己创建的镜像仓库中，使用`docker tag`命令修改镜像标签

```bash
# 注意新镜像名必须是'仓库用户名作为前缀/自定义新名作为后缀'
[root@iZ7xvb2aw7a3ekhvnen6u1Z ~]# docker tag centos:7.9.2009  tonydu789/nginx_vim_net-tools_centos7.9
# 修改镜像名不写标签默认按 lateset 处理
```

使用`docker push`远程推送镜像

```bash
[root@iZ7xvb2aw7a3ekhvnen6u1Z ~]# docker push tonydu789/nginx_vim_net-tools_centos7.9
```

此时我们查看远程仓库的镜像是否存在

![notion image](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20230610145551258.png)

### 3.使用另外一台服务器查看可行性

使用`docker pull`拉取刚刚上传到远程仓库的镜像

```bash
[root@racknerd-e0c617 ~]# docker pull tonydu789/nginx_vim_net-tools_centos7.9
```

使用`docker run`运行镜像

```bash
docker run -d -p 8888:8888 tonydu789/nginx_vim_net-tools_centos7.9 nginx -g "daemon off;"
```



### 4.浏览器访问测试

发现是403界面

![image-20240929003852852](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20240929003852852.png)

修改容器内nginx的html文件

```bash
[root@8743ecb1451c html]# pwd
/usr/share/nginx/html

[root@8743ecb1451c html]# vim index.html

[root@8743ecb1451c html]# cat index.html 
hello world
```

再次访问

![image-20240929004055815](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/image-20240929004055815.png)



### 5.关于容器日志的问题

```bash
# 发现对tonydu789/nginx_vim_net-tools_centos7.9这个镜像使用"docker logs -f 容器id"命令不会返回任何日志
# 原因：
在Docker中，每个容器的日志通常包括两个部分：
1.容器的标准输出（STDOUT）和标准错误输出（STDERR）：这是容器内部运行的进程产生的日志，通常包括你在终端中执行的命令输出和任何错误信息。
2.应用程序日志：这是容器内部应用程序（如 Nginx）生成的日志，通常写入到文件系统中的特定日志文件中。

	对于 CentOS 镜像，如果没有特别的配置来重定向应用程序日志到 STDOUT 或 STDERR，那么使用 docker logs -f 命令通常只会显示你在容器终端中执行的命令和它们的输出。这是因为 CentOS 镜像通常用于运行各种不同的应用程序，而这些应用程序可能将日志写入到文件系统中的任何位置，而不是 STDOUT 或 STDERR。
	当你使用 docker logs -f 命令时，Docker 默认显示容器的 STDOUT 和 STDERR。这是因为 Docker 守护进程默认会将容器的 STDOUT 和 STDERR 重定向到一个特殊的日志文件中，以便可以通过 docker logs 命令来访问。
	对于 Nginx 镜像，Nginx 服务的日志默认是写入到文件系统中的 /var/log/nginx/ 目录下的。但为了让这些日志能够通过 docker logs 命令查看，Nginx 镜像通常会在启动脚本中将 Nginx 的日志文件链接到 Docker 的 STDOUT 和 STDERR。


# 解决方案（两种）
# 方法一：将 Nginx 的日志文件 /var/log/nginx/access.log 和 /var/log/nginx/error.log 链接到 Docker 的 STDOUT 和 STDERR 通过创建日志文件到/dev/stdout和/dev/stderr的符号链接：
# 强制创建软连接
[root@7b7430aede26 nginx]# ln -sf /dev/stdout /var/log/nginx/access.log

[root@7b7430aede26 nginx]# ln -sf /dev/stderr /var/log/nginx/error.log
# 这样，Nginx 的访问日志和错误日志就会被重定向到 Docker 的日志系统，你就可以使用 docker logs -f 命令来查看它们。


# 方法二：使用docker run -v参数来解决这个问题，把容器内的nginx日志文件所在目录挂载到宿主机的目录上
[root@server01 ~]# docker run -d  -p 83:80  -v /var/log:/var/log/nginx  tonydu789/nginx_vim_net-tools_centos7.9  nginx -g "daemon off;"
d4687069559059f33e40a66e70b4fb28ad12d115132dde131dd87941ac5732c5

# 把容器内的nginx日志文件所在目录/var/log/nginx/挂载到宿主机的/var/log（任意）目录上
[root@server01 /var/log]# ls
access.log         cron                maillog-20240922   spooler
anaconda           cron-20240907       maillog-20240930   spooler-20240907
audit              cron-20240918       messages           spooler-20240918
boot.log           cron-20240922       messages-20240907  spooler-20240922
boot.log-20240918  cron-20240930       messages-20240918  spooler-20240930
boot.log-20240922  dmesg               messages-20240922  tallylog
boot.log-20240923  dmesg.old           messages-20240930  tuned
boot.log-20240930  error.log           rhsm               vmware-vgauthsvc.log.0
boot.log-20241001  firewalld           sa                 vmware-vmsvc.log
boot.log-20241004  grubby_prune_debug  secure             wtmp
boot.log-20241005  lastlog             secure-20240907    yum.log
btmp               maillog             secure-20240918
btmp-20241001      maillog-20240907    secure-20240922
chrony             maillog-20240918    secure-20240930

# 日志这下就能正常输出了
[root@server01 /var/log]# tail -f access.log 
192.168.1.8 - - [05/Oct/2024:11:06:17 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
192.168.1.8 - - [05/Oct/2024:11:06:17 +0000] "GET /favicon.ico HTTP/1.1" 404 3650 "http://192.168.1.200:83/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
192.168.1.8 - - [05/Oct/2024:11:07:06 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
192.168.1.8 - - [05/Oct/2024:11:07:17 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
```



#### 拓展：容器日志stdout和stderr

```bash
# 这个容器一共有这些日志，里面包括 stdout 和 stderr
[root@server01 ~]# docker logs -f ea6
10.0.0.1 - - [10/Oct/2024:16:30:29 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:30:29 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:30:30 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:30:30 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:41:19 +0000] "GET / HTTP/1.1" 200 616 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:41:20 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:41:20 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:08 +0000] "GET / HTTP/1.1" 200 616 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
2024/10/11 16:05:08 [error] 9#9: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "10.0.0.230:81", referrer: "http://10.0.0.230:81/"
10.0.0.1 - - [11/Oct/2024:16:05:08 +0000] "GET /favicon.ico HTTP/1.1" 404 3650 "http://10.0.0.230:81/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
2024/10/11 16:05:13 [error] 9#9: *1 open() "/usr/share/nginx/html/hhh" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /hhh HTTP/1.1", host: "10.0.0.230:81"
10.0.0.1 - - [11/Oct/2024:16:05:13 +0000] "GET /hhh HTTP/1.1" 404 3650 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:13 +0000] "GET /nginx-logo.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:13 +0000] "GET /poweredby.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
2024/10/11 16:05:23 [error] 9#9: *1 open() "/usr/share/nginx/html/hhh" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /hhh HTTP/1.1", host: "10.0.0.230:81"
10.0.0.1 - - [11/Oct/2024:16:05:23 +0000] "GET /hhh HTTP/1.1" 404 3650 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:23 +0000] "GET /nginx-logo.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:23 +0000] "GET /poweredby.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
2024/10/11 16:06:56 [error] 10#10: *3 open() "/usr/share/nginx/html/hhh" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /hhh HTTP/1.1", host: "10.0.0.230:81"
10.0.0.1 - - [11/Oct/2024:16:06:56 +0000] "GET /hhh HTTP/1.1" 404 3650 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:06:56 +0000] "GET /nginx-logo.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:06:56 +0000] "GET /poweredby.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"


# 如果想把日志信息重定向到文件中，不指定参数（默认情况下）则只会把stdout输入到文件中，剩下的srderr会直接输出
[root@server01 ~]# docker logs ea6 > nginx-access.log
2024/10/11 16:05:08 [error] 9#9: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "10.0.0.230:81", referrer: "http://10.0.0.230:81/"
2024/10/11 16:05:13 [error] 9#9: *1 open() "/usr/share/nginx/html/hhh" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /hhh HTTP/1.1", host: "10.0.0.230:81"
2024/10/11 16:05:23 [error] 9#9: *1 open() "/usr/share/nginx/html/hhh" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /hhh HTTP/1.1", host: "10.0.0.230:81"
2024/10/11 16:06:56 [error] 10#10: *3 open() "/usr/share/nginx/html/hhh" failed (2: No such file or directory), client: 10.0.0.1, server: _, request: "GET /hhh HTTP/1.1", host: "10.0.0.230:81"



# 只导出stderr
[root@server01 ~]# docker logs ea6 2> nginx-error.log 
10.0.0.1 - - [10/Oct/2024:16:30:29 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:30:29 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:30:30 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:30:30 +0000] "GET / HTTP/1.1" 403 555 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:41:19 +0000] "GET / HTTP/1.1" 200 616 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:41:20 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [10/Oct/2024:16:41:20 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:08 +0000] "GET / HTTP/1.1" 200 616 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:08 +0000] "GET /favicon.ico HTTP/1.1" 404 3650 "http://10.0.0.230:81/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:13 +0000] "GET /hhh HTTP/1.1" 404 3650 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:13 +0000] "GET /nginx-logo.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:13 +0000] "GET /poweredby.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:23 +0000] "GET /hhh HTTP/1.1" 404 3650 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:23 +0000] "GET /nginx-logo.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:05:23 +0000] "GET /poweredby.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:06:56 +0000] "GET /hhh HTTP/1.1" 404 3650 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:06:56 +0000] "GET /nginx-logo.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"
10.0.0.1 - - [11/Oct/2024:16:06:56 +0000] "GET /poweredby.png HTTP/1.1" 200 368 "http://10.0.0.230:81/hhh" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36 SLBrowser/9.0.3.5211 SLBChan/105" "-"



# 把所有的日志都导出到文件中
[root@server01 ~]# docker logs ea6 > nginx-all.log 2>&1
# 没有输出内容
```



## 导出镜像、导入镜像

```bash
# 导出镜像，docker save命令
[root@server01 ~]# docker images
REPOSITORY                                TAG        IMAGE ID       CREATED       SIZE
tonydu789/nginx_vim_net-tools_centos7.9   latest     0a11d895067e   4 days ago    318MB
nginx                                     latest     5ef79149e0ec   6 weeks ago   188MB
centos                                    7.9.2009   eeb6ee3f44bd   3 years ago   204MB
nginx                                     1.19.7     35c43ace9216   3 years ago   133MB
centos                                    7.4.1708   9f266d35e02c   5 years ago   197MB
[root@server01 ~]# 
[root@server01 ~]# docker save tonydu789/nginx_vim_net-tools_centos7.9 > /opt/nginx_vim_net-tools_centos7.9.tar
[root@server01 ~]# 
[root@server01 ~]# ls /opt
containerd  docker-compose.yml  nginx_vim_net-tools_centos7.9.tar  python_project

# 导入镜像，docker load命令
[root@server01 /opt]# docker images 
REPOSITORY   TAG        IMAGE ID       CREATED       SIZE
nginx        latest     5ef79149e0ec   6 weeks ago   188MB
centos       7.9.2009   eeb6ee3f44bd   3 years ago   204MB
nginx        1.19.7     35c43ace9216   3 years ago   133MB
centos       7.4.1708   9f266d35e02c   5 years ago   197MB
[root@server01 /opt]# 
[root@server01 /opt]# docker load -i nginx_vim_net-tools_centos7.9.tar 
b01bcc31801f: Loading layer  117.8MB/117.8MB
Loaded image: tonydu789/nginx_vim_net-tools_centos7.9:latest
[root@server01 /opt]# 
[root@server01 /opt]# docker images
REPOSITORY                                TAG        IMAGE ID       CREATED       SIZE
tonydu789/nginx_vim_net-tools_centos7.9   latest     0a11d895067e   5 days ago    318MB
nginx                                     latest     5ef79149e0ec   6 weeks ago   188MB
centos                                    7.9.2009   eeb6ee3f44bd   3 years ago   204MB
nginx                                     1.19.7     35c43ace9216   3 years ago   133MB
centos                                    7.4.1708   9f266d35e02c   5 years ago   197MB
```



# 关于docker.sock

![1661997269648](02_docker%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.assets/1661997269648.png)



# docker命令总结

## 增

```bash
# 拉取镜像
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
# OPTIONS：可以是一些拉取镜像时的选项，如 --all-tags 表示拉取所有标签的镜像，--platform 用于指定平台架构等
# [:TAG|@DIGEST]：TAG：是镜像的标签，用于区分不同版本或不同构建的镜像，通常以 :tag 的形式跟在镜像名称后面，如 :5.7.25
# @DIGEST：可以使用镜像的 sha256 摘要来精确拉取镜像，例如：@sha256:98455b9624a96e32b353297bb312913b6bbd62ac195cea2c7dd477209ba572d6

# 完整的镜像名称可以包含三部分：[registry/][username/]imagename[:tag]
# registry：是镜像仓库的地址，例如 docker.io 是 Docker Hub 的默认仓库地址，通常可以省略
# username：是 Docker 账户名，如果是公共仓库的公共镜像，这个部分也可以省略
# imagename：是镜像的名称，如 mysql
# tag：是镜像的标签，如 :5.7.25

# 如果想拉取官方的mysql镜像，但是官方只给了docker pull mysql:5.7.25命令，如果别的仓库也有镜像叫这个名，万一拉取错了怎么办？
# 指定仓库地址或者使用摘要
docker pull docker.io/mysql@sha256:98455b9624a96e32b353297bb312913b6bbd62ac195cea2c7dd477209ba572d6



# 基于镜像，生成一个新的容器进程
docker run 镜像id

# docker run常用参数 
-i -t -d -p -P -v



# 尝试启动一个容器
docker start 挂掉的容器id
```



## 删

 ```bash
 # 删除一个容器记录
 docker rm  容器id
 
 
 
 # 先docker stop，再docker rm
 docker rm -f  容器id
 
 
 
 # 删除指定镜像
 docker rmi 镜像id
 ```



## 改

```bash
# 提交容器记录为新的镜像
docker commit 容器id



# 重命名容器（注意'docker tag 旧镜像仓库:标签/镜像id 新镜像仓库:标签'是修改镜像名）
docker rename  旧容器名  新容器名
[root@docker-200 ~]# docker rename cranky_wilbur  bingcheng_docker
[root@docker-200 ~]# 
[root@docker-200 ~]# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS     NAMES
e5cb987f3e72   nginx:1.19.7   "/docker-entrypoint.…"   24 minutes ago   Up 22 minutes   80/tcp    bingcheng_docker



# 停止一个运行中的容器
docker stop 容器id
[root@docker-200 ~]#docker stop bingcheng_docker 
bingcheng_docker



# 重启一个容器进程
docker restart 容器id



# 和 docket stop 一样，可以停止单个、多个容器
docker kill



# 导出容器为一个tar包，相当于 docker commit + docker save
docker export bingcheng_docker > /opt/bingcheng.tar
# docker export 和 docker commit + docker save的区别
# docker export 导出的镜像用 docker history 看不出来镜像层，而 docker commit + docker save 导出的镜像是可以查看镜像层的
# docker export 导出的镜像没有名字和标签
[root@docker-201 /opt]#docker images
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
<none>       <none>    c80569235212   8 seconds ago   131MB
# 由于docker export 导出的镜像无法查看镜像层，所以这个镜像暴露的端口也就无从查起，docker run -P 也无法使用



# 不如 docker load 好用，这个命令在新版docker已经弃用了
docker import /opt/bingcheng.tar



# 修改镜像的repository和tag
docker tag c80569235212  yuchao163/nginx:1.19.7
[root@docker-201 /opt]#docker images
REPOSITORY        TAG       IMAGE ID       CREATED              SIZE
yuchao163/nginx   1.19.7    c80569235212   About a minute ago   131MB



# 把容器中的文件复制到宿主机，反之亦然，宿主机的地址可以为非本地（常用命令）
docker cp 容器id:目标文件或目标目录  宿主机目录
[root@server01 ~]# docker cp ea6:/var/log/nginx/ ./
Successfully copied 2.56kB to /root/./
[root@server01 ~]# 
[root@server01 ~]# ls
anaconda-ks.cfg  nginx

[root@server01 ~]# docker cp ./nginx/ ea6:/root
Successfully copied 2.56kB to ea6:/root
[root@server01 ~]# 
[root@server01 ~]# docker exec -it ea6 bash
[root@ea645a946b58 /]# 
[root@ea645a946b58 ~]# ls /root
anaconda-ks.cfg  nginx
```



## 查

```bash
# 查询运行中的容器
docker ps



# 查询容器的历史记录，运行的，挂掉的
docker ps -a



# 查询所有容器的记录，只显示id号
docker ps -aq



# 查看一个运行中的容器进程信息
docker top  容器id



# 查看容器的端口映射情况
docker port 容器id



# 输出容器的详细json数据信息
docker inspect  容器id



# 输出容器内的stdout，stderr的日志（需要你自己主动设置，前面的实践有具体操作）
docker logs 容器id



# 查看一个镜像的镜像层
docker hitory 镜像id



# 在一个运行中的容器内，执行一个命令
docker exec 容器id 命令
# 交互式，进入容器空间内，执行命令，touch /opt/heiheihei.log 
[root@docker-200 ~]#docker exec    -i -t  10b646bb3e09   bash 
root@10b646bb3e09:/# touch /opt/heiheihei.log
root@10b646bb3e09:/# cd /opt
root@10b646bb3e09:/opt# ls
heiheihei.log

# 非交互式，再给容器的日志写入一句话“xixi” 
[root@docker-200 ~]#docker exec 10b bash -c "echo 'xixi' > /opt/heiheihei.log"
# 试试能查出来，强写入的这个数据吗（非交互式）
[root@docker-200 ~]#docker exec 10b cat /opt/heiheihei.log
xixi
[root@docker-200 ~]#docker exec 10b bash -c "cat /opt/heiheihei.log"
xixi



# 查看运行中的容器，使用资源情况 cpu 内存 网络 磁盘 进程等
docket stats 容器id
CONTAINER ID   NAME          CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O     PIDS
69540a0f019c   busy_carson   0.00%     1.621MiB / 1.936GiB   0.08%     1.09kB / 0B   10.6MB / 0B   1



# 显示docker服务端信息
docker info



# 显示容器端口映射关系
docker port 容器id
[root@docker-200 ~]#docker run -d -p 80:80 nginx:1.19.7 
10b646bb3e0980c08100c6ab4c352501a47944b14167b11fef6b37e0e9908ce6
[root@docker-200 ~]#
[root@docker-200 ~]#docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                               NAMES
10b646bb3e09   nginx:1.19.7   "/docker-entrypoint.…"   8 seconds ago   Up 7 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   vigilant_bhabha
[root@docker-200 ~]#
[root@docker-200 ~]#docker port 10b
80/tcp -> 0.0.0.0:80
80/tcp -> :::80
```

