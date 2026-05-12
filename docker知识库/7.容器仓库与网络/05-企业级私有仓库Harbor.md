# 05-企业级私有仓库Harbor

仓库的概念也就是用于存储，docker仓库用于存储镜像。

镜像构建完成后，很容易可以在宿主机上运行，但是如果要在其他服务器上运行，则需要考虑镜像的分发，存储的问题。

## 公有/私有/仓库

Docker Registry有两种形式

- 公开，开放给所有用户，提供给所有用户搜索，拉取，提交，更新镜像，还免费保管用户镜像数据。
  - 此类服务受限于网络限制，无法及时立即获取所需镜像，简单说就是需要用什么得现下载，得看网速
  - 优点是可以获取绝大部分公开的镜像，方便使用
- 私有范围的Registry服务，用在学校，企业内网的研发环境
  - 局域网环境，保证了镜像拉取速度
  - 保证核心镜像数据安全
  - 存在镜像不丰富问题



## 公开服务仓库

最常见的Registry是Docker Hub，也是docker默认允许用户管理镜像的Registry服务，拥有大量高质量的官方镜像。

由于网络地域原因，公开服务在国内访问较慢，也就出现了针对镜像服务的加速器。

- [阿里云加速器](http://book.bikongge.com/sre/10-云原生容器编排/docker-all/www.yuchaoit.cn)
- DaoCloud加速器
- 灵雀云加速器
- 等等

使用加速器，可以从国内的地址获取Docker Hub的镜像，速度会快很多。



## 私有服务仓库

除了私用公开服务外，还可以在自己本地搭建私有Docker Registry。

开源的Docker Registry镜像只提供了Docker Registry API的功能，没有图形化功能。



# 1.部署docker-harbor

## 步骤

```
1. 安装docker-compose工具
2. 下载harbor安装包
3. 修改harbor配置
4. 自动化安装
```



## 安装docker-compose

```bash
yum install docker-compose -y

[root@docker-200 /www.yuchaoit.cn/test_dockerfile/tomcat_web/web_base]#docker-compose version
docker-compose version 1.18.0, build 8dd22a9
docker-py version: 2.6.1
CPython version: 3.6.8
OpenSSL version: OpenSSL 1.0.2k-fips  26 Jan 2017
```



## 上传harbor安装包

```bash
[root@docker-200 /www.yuchaoit.cn]#ll
total 633852
-rw-r--r-- 1 root root  32749651 Aug 25 05:02 all-game.tgz
-rw-r--r-- 1 root root 616312579 Aug 27 20:03 harbor-offline-installer-v1.9.0-rc1.tgz
drwxr-xr-x 6 root root       114 Aug 23 00:08 jd
drwxr-xr-x 5 root root        59 Aug 27 20:03 test_dockerfile
drwxr-xr-x 3 root root        98 Aug 22 23:40 xiaoniao
drwxr-xr-x 7 root root       153 Aug 23 20:42 yunpan


# 解压
[root@docker-200 /www.yuchaoit.cn]#tar -xf harbor-offline-installer-v1.9.0-rc1.tgz 

[root@docker-200 /www.yuchaoit.cn/harbor]#ls
harbor.v1.9.0.tar.gz  harbor.yml  install.sh  LICENSE  prepare



# 修改harbor.yaml配置文件
[root@docker-200 /www.yuchaoit.cn/harbor]#grep -E '10.0.0.200|yuchao' harbor.yml 
hostname: 10.0.0.200    # 改成宿主机ip
harbor_admin_password: www.yuchaoit.cn    # 修改默认登录密码



# 安装，一键自动化安装的过程
[root@docker-200 /www.yuchaoit.cn/harbor]#./install.sh 

[Step 0]: checking installation environment ...

Note: docker version: 20.10.17

Note: docker-compose version: 1.18.0

[Step 1]: loading Harbor images ...



# 安装结果
✔ ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at http://10.0.0.200. 
For more details, please visit https://github.com/goharbor/harbor .
```



## 访问harbor

检查harbor进程

![image-20250106105408528](05-%E4%BC%81%E4%B8%9A%E7%BA%A7%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93Harbor.assets/image-20250106105408528.png)

登录信息

```
admin 
www.yuchaoit.cn
```

![image-20220827122047517](05-%E4%BC%81%E4%B8%9A%E7%BA%A7%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93Harbor.assets/image-20220827122047517.png)



## 创建项目

![image-20220827122228236](05-%E4%BC%81%E4%B8%9A%E7%BA%A7%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93Harbor.assets/image-20220827122228236.png)



## 修改docker配置，信任自建仓库（支持非https镜像仓库）

```bash
[root@docker-200 /www.yuchaoit.cn/harbor]#cat  /etc/docker/daemon.json 
{
  "registry-mirrors" : ["https://ms9glx6x.mirror.aliyuncs.com"],
  # 只加这段
  "insecure-registries":["http://10.0.0.200"]
}

[root@docker-200 /www.yuchaoit.cn/harbor]#systemctl daemon-reload

[root@docker-200 /www.yuchaoit.cn/harbor]#systemctl restart docker

[root@docker-200 /www.yuchaoit.cn/harbor]#docker restart $(docker ps -aq)
27dd0afa18db
211d14ec1c62
b6e66bb15157
302c4a1a8caa
e31af30cc7f8
336fbfca3a39
00f12d9e9222
fb2f61f376c2
31ca2addee34
```



## 推送本地镜像（修改镜像tag）

```bash
# 镜像要推送到仓库，名字上有要求：harbor地址/项目名/镜像名:版本

# 把镜像改名
[root@docker-200 /www.yuchaoit.cn/harbor]#docker tag nginx:1.17.9    10.0.0.200/my_harbor01/nginx:1.17.9

[root@docker-200 /www.yuchaoit.cn/harbor]#docker images |grep nginx
centos7_nginx                   1.20.1                     f20ce96eb5be   22 hours ago        262MB
10.0.0.200/my_harbor01/nginx    1.17.9                     5a8dfb2ca731   2 years ago         127MB
nginx                           1.17.9                     5a8dfb2ca731   2 years ago         127MB
goharbor/nginx-photon           v1.9.0                     492e9528214c   2 years ago         43.2MB



# 登录harbor
[root@docker-200 /www.yuchaoit.cn/harbor]#docker login 10.0.0.200
Username: admin
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
[root@docker-200 /www.yuchaoit.cn/harbor]#


# 推送镜像,docker自动根据名字，知道往哪推送
[root@docker-200 /www.yuchaoit.cn/harbor]#docker push 10.0.0.200/my_harbor01/nginx:1.17.9 
The push refers to repository [10.0.0.200/my_harbor01/nginx]
351816b95c49: Pushed 
0e07021aa61a: Pushed 
b60e5c3bcef2: Pushed 
1.17.9: digest: sha256:30d9dde0c4cb5ab4989a92bc2c235b995dfa88ff86c09232f309b6ad27f1c7cd size: 948
```



### 检查harbor镜像

![image-20220827122758564](05-%E4%BC%81%E4%B8%9A%E7%BA%A7%E7%A7%81%E6%9C%89%E4%BB%93%E5%BA%93Harbor.assets/image-20220827122758564.png)



### 下载harbor镜像

```bash
# 其他机器即可使用该镜像地址，获取私有镜像仓库内容

docker pull 10.0.0.200/my_harbor01/nginx:1.17.9
```



## 停止harbor

```bash
[root@docker-200 /www.yuchaoit.cn/harbor]#docker-compose stop 
Stopping nginx             ... done
Stopping harbor-jobservice ... done
Stopping harbor-core       ... done
Stopping redis             ... done
Stopping registryctl       ... done
Stopping harbor-portal     ... done
Stopping registry          ... done
Stopping harbor-db         ... done
Stopping harbor-log        ... done
```



## 还有一个docker registry私有仓库

纯API形式的私有仓库，不做讲解了，比较简单

```bash
# 获取registry镜像，加上参数，确保每次重启
docker run -d \
    --name chaoge_registry \
    --restart=always \
    -p 5000:5000 \
    -v /opt/data/registry:/var/lib/registry \
    registry



# 确保接口可以访问
[root@docker-200 ~]#curl http://10.0.0.200:5000/ -I
HTTP/1.1 200 OK
Cache-Control: no-cache
Date: Wed, 07 Sep 2022 08:24:47 GMT

# 私有仓库接口 /v2 path
http://10.0.0.200:5000/v2/



# 推送本地镜像到私有registry
# 修改镜像tag，制定为本地私有镜像仓库api
# 私有registery没有账户密码认证功能
docker tag ubuntu:latest  10.0.0.200:5000/ubuntu:latest

docker  push 10.0.0.200:5000/ubuntu:latest



# 修改docker配置文件，加入私有registry，改为 非安全模式的 镜像地址
[root@docker-200 /docker-harbor/harbor]#cat  /etc/docker/daemon.json 
{
  "registry-mirrors": ["https://ms9glx6x.mirror.aliyuncs.com"],
  "insecure-registries":["http://10.0.0.200","http://10.0.0.200:5000"]

}



# 重启docker
[root@docker-200 /docker-harbor/harbor]#systemctl daemon-reload

[root@docker-200 /docker-harbor/harbor]#systemctl restart docker



# 查询私有仓库api接口
[root@docker-200 /docker-harbor/harbor]#curl 10.0.0.200:5000/v2/_catalog
{"repositories":["linux0224/t1-flask-redis","ubuntu"]}



# 拉取ubuntu镜像
[root@docker-200 /docker-harbor/harbor]#docker pull 10.0.0.200:5000/ubuntu
```



### 官方提供registry仓库的API接口规则文档

https://docs.docker.com/registry/spec/api/#listing-repositories

```bash
# 列出所有的镜像地址
[root@docker-200 /docker-harbor/harbor]#curl 10.0.0.200:5000/v2/_catalog
{"repositories":["linux0224/t1-flask-redis"]}
```



## 还有一个docker hub

可以自行注册账号，玩一玩公开仓库。