# Docker快速入门

## 1. 了解Docker

### 什么是Docker

那么什么是Docker呢？你可以查到很多与它相关的技术资料，但对于刚刚入门的朋友来说，可能会显得有些晦涩难懂。

简单来说，Docker是一种虚拟化技术，也就是在当前的操作系统下创建一个相对独立的特定操作系统环境，用来运行我们所需要的软件。Docker的虚拟化介于虚拟机和沙盒之间，是一种非常高效的虚拟化方式。

![image-20241114205857125](%E5%9F%BA%E4%BA%8Ewsl%E7%9A%84docker_desktop%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.assets/image-20241114205857125.png)



### Docker的优势

每个运行的Docker容器都捆绑了这个容器所运行的软件以及最低程度所需求的系统环境。举个例子，如果我需要运行一个Docker中的文本编辑器，那么Docker不会给它提供超出文本编辑器之外的系统组件，例如处理音频、视频、网络通信等这一类的组件。因此，我们的Docker容器通常非常精简，运行效率极高。

如果你在Windows上运行一个Linux的虚拟机，则必须附带Linux系统的所有系统组件。即便我只想在这个虚拟机中运行一个简单的文本编辑器，它同样会加载音频、视频、网络通信等系统核心组件。

![image-20241114210035917](%E5%9F%BA%E4%BA%8Ewsl%E7%9A%84docker_desktop%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.assets/image-20241114210035917.png)



### Docker与Windows适配

Docker在Windows上的适配经历了三个阶段：第一个阶段是完全不适配Windows，因为Docker本身是构建在Linux系统上的一个虚拟化技术；第二阶段是有限的基于虚拟机的适配；第三阶段是基于WSL2（Windows Subsystem for Linux 2）虚拟化技术的适配。

![image-20241114210056675](%E5%9F%BA%E4%BA%8Ewsl%E7%9A%84docker_desktop%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.assets/image-20241114210056675.png)

目前WSL2已经可以支持以非常高的效率在Windows系统上虚拟Linux系统。配合Docker Desktop，可以使得我们非常方便地在Windows上运行Linux程序。



### Docker与LLM应用

目前很多大型语言模型的应用工具，例如RAG（Retrieval-Augmented Generation）、工作流、Agent等，都是在Linux环境下开发的，它们都不支持直接在Windows上安装和部署。因此，我们就需要采用Docker环境来运行这些程序。今天，我们就来带领大家学习如何安装Docker，并进行简单的管理操作。



## 2. 安装Docker

### 2.0 前置条件

首先要注意，想要运行 Docker Desktop，你的windows系统必须高于1904版本。

你可以进入 Windows设置→系统→关于，来查看当前的windows版本。例如我所使用的windows版本号为22h2。

![image-20241114210116216](%E5%9F%BA%E4%BA%8Ewsl%E7%9A%84docker_desktop%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.assets/image-20241114210116216.png)

不过请放心，一般来说只有超过5年以上从未更新过的电脑才会停留在1904那个比较古老的版本。

通常只有Windows专业版、企业版或工作站版才能够安装Hyper-V和WSL2。而这两个组件是在Windows上运行Docker Desktop的必备条件。

但是今天我们介绍的安装方法，同时支持Windows家庭版和专业版的安装。我们将会帮助大家绕过一部分Windows系统的限制。



### 2.1 安装 Hyper-V

首先我们需要运行一个批处理程序来启动Hyper-V组件。

通过将下方的代码复制到记事本中，并另存为`enable_hyper_v.cmd`，之后直接右击该脚本，通过“管理员模式”运行。

```powershell
pushd "%~dp0"

dir /b %SystemRoot%\\servicing\\Packages\\*Hyper-V*.mum > hyper-v.txt

for /f %%i in ('findstr /i . hyper-v.txt 2^>nul') do dism /online /norestart /add-package:"%SystemRoot%\\servicing\\Packages\\%%i"

del hyper-v.txt

Dism /online /enable-feature /featurename:Microsoft-Hyper-V-All /LimitAccess /ALL
```

你也可以直接点击这里 https://www.123pan.com/s/5cWiVv-wj8C.html（提取码:leai）下载这个脚本。

如果你的电脑没有Hyper-V组件，系统将会从Windows Update服务器那里更新Windows的相关组件，这可能会花费一些时间。安装完成后，系统需要重启。



### 2.2 安装WSL升级包（非必要步骤）

然后你需要前往微软的官方网站下载WSL的离线升级包。

https://learn.microsoft.com/zh-cn/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package

或者直接点击这个链接 WSL离线升级包： https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi 下载升级包。

当然，你也可以前往我们提供的网盘进行下载。点击这里 https://www.123pan.com/s/5cWiVv-wj8C.html（提取码:leai）下载WSL的离线升级包。

双击安装。这里我们可能需要再次重启电脑。

更多关于WSL安装的帮助信息可以在这里找到：https://learn.microsoft.com/zh-cn/windows/wsl/install



### 2.3 安装Docker Desktop

完成以上两步后，无论你使用的是Windows家庭版还是专业版，你就基本上可以无痛安装Docker。

Docker的官方下载地址是：

https://www.docker.com/products/docker-desktop/

如果网速不佳，你也可以前往我们提供的网盘进行下载。

下面我们来演示Docker的安装过程。安装时默认选中使用基于WSL2的虚拟化，点击“下一步”就可以轻松完成Docker的安装。

![image-20241114210450545](%E5%9F%BA%E4%BA%8Ewsl%E7%9A%84docker_desktop%E5%AE%89%E8%A3%85%E6%95%99%E7%A8%8B.assets/image-20241114210450545.png)

如果你只是偶尔使用基于Docker的某些应用程序，那么你可以关闭Docker的自动启动功能，选择在必要的时候手动启动。