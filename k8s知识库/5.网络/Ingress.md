# Ingress

前面我们学习了在 Kubernetes 集群内部使用 kube-dns 实现服务发现的功能，那么我们部署在 Kubernetes 集群中的应用如何暴露给外部的用户使用呢？我们知道可以使用 `NodePort` 和 `LoadBlancer` 类型的 Service 可以把应用暴露给外部用户使用，除此之外，Kubernetes 还为我们提供了一个非常重要的资源对象可以用来暴露服务给外部用户，那就是 `Ingress`。对于小规模的应用我们使用 NodePort 或许能够满足我们的需求，但是当你的应用越来越多的时候，你就会发现对于 NodePort 的管理就非常麻烦了，这个时候使用 Ingress 就非常方便了，可以避免管理大量的端口。



## 资源对象

`Ingress` 资源对象是 Kubernetes 内置定义的一个对象，是从 Kuberenets 集群外部访问集群的一个入口，将外部的请求转发到集群内不同的 Service 上，其实就相当于 nginx、haproxy 等负载均衡代理服务器，可能你会觉得我们直接使用 nginx 就实现了，但是只使用 nginx 这种方式有很大缺陷，每次有新服务加入的时候怎么改 Nginx 配置？不可能让我们去手动更改或者滚动更新前端的 Nginx Pod 吧？那我们再加上一个服务发现的工具比如 consul 如何？貌似是可以，对吧？Ingress 实际上就是这样实现的，只是服务发现的功能自己实现了，不需要使用第三方的服务了，然后再加上一个域名规则定义，路由信息的刷新依靠 Ingress Controller 来提供。

![ingress flow](Ingress.assets/ingress-flow.png)

Ingress Controller 可以理解为一个监听器，通过不断地监听 kube-apiserver，实时的感知后端 Service、Pod 的变化，当得到这些信息变化后，Ingress Controller 再结合 Ingress 的配置，更新反向代理负载均衡器，达到服务发现的作用。其实这点和服务发现工具 consul、 consul-template 非常类似。



## 定义

一个常见的 Ingress 资源清单如下所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
          - path: /testpath
            pathType: Prefix
            backend:
              service:
                name: test
                port:
                  number: 80
```

上面这个 Ingress 资源的定义，配置了一个路径为 `/testpath` 的路由，所有 `/testpath/**` 的入站请求，会被 Ingress 转发至名为 test 的服务的 80 端口的 `/` 路径下。可以将 Ingress 狭义的理解为 Nginx 中的配置文件 `nginx.conf`。

此外 Ingress 经常使用注解 `annotations` 来配置一些选项，当然这具体取决于 Ingress 控制器的实现方式，不同的 Ingress 控制器支持不同的注解。

另外需要注意的是当前集群版本是 `v1.22`，这里使用的 apiVersion 是 `networking.k8s.io/v1`，所以如果是之前版本的 Ingress 资源对象需要进行迁移。 Ingress 资源清单的描述我们可以使用 `kubectl explain` 命令来了解：

```shell
➜ kubectl explain ingress.spec
KIND:     Ingress
VERSION:  networking.k8s.io/v1

RESOURCE: spec <Object>

DESCRIPTION:
     Spec is the desired state of the Ingress. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

     IngressSpec describes the Ingress the user wishes to exist.

FIELDS:
   defaultBackend       <Object>
     DefaultBackend is the backend that should handle requests that don't match
     any rule. If Rules are not specified, DefaultBackend must be specified. If
     DefaultBackend is not set, the handling of requests that do not match any
     of the rules will be up to the Ingress controller.

   ingressClassName     <string>
     IngressClassName is the name of the IngressClass cluster resource. The
     associated IngressClass defines which controller will implement the
     resource. This replaces the deprecated `kubernetes.io/ingress.class`
     annotation. For backwards compatibility, when that annotation is set, it
     must be given precedence over this field. The controller may emit a warning
     if the field and annotation have different values. Implementations of this
     API should ignore Ingresses without a class specified. An IngressClass
     resource may be marked as default, which can be used to set a default value
     for this field. For more information, refer to the IngressClass
     documentation.

   rules        <[]Object>
     A list of host rules used to configure the Ingress. If unspecified, or no
     rule matches, all traffic is sent to the default backend.

   tls  <[]Object>
     TLS configuration. Currently the Ingress only supports a single TLS port,
     443. If multiple members of this list specify different hosts, they will be
     multiplexed on the same port according to the hostname specified through
     the SNI TLS extension, if the ingress controller fulfilling the ingress
     supports SNI.
```

从上面描述可以看出 Ingress 资源对象中有几个重要的属性：`defaultBackend`、`ingressClassName`、`rules`、`tls`。



### rules

其中核心部分是 `rules` 属性的配置，每个路由规则都在下面进行配置：

- `host`：可选字段，上面我们没有指定 host 属性，所以该规则适用于通过指定 IP 地址的所有入站 HTTP 通信，如果提供了 host 域名，则 `rules` 则会匹配该域名的相关请求，此外 `host` 主机名可以是精确匹配（例如 `foo.bar.com`）或者使用通配符来匹配（例如 `*.foo.com`）。
- `http.paths`：定义访问的路径列表，比如上面定义的 `/testpath`，每个路径都有一个由 `backend.service.name` 和 `backend.service.port.number` 定义关联的 Service 后端，在控制器将流量路由到引用的服务之前，`host` 和 `path` 都必须匹配传入的请求才行。
- `backend`：该字段其实就是用来定义后端的 Service 服务的，与路由规则中 `host` 和 `path` 匹配的流量会将发送到对应的 backend 后端去。

> 此外一般情况下在 Ingress 控制器中会配置一个 `defaultBackend` 默认后端，当请求不匹配任何 Ingress 中的路由规则的时候会使用该后端。`defaultBackend` 通常是 Ingress 控制器的配置选项，而非在 Ingress 资源中指定。



### Resource

`backend` 后端除了可以引用一个 Service 服务之外，还可以通过一个 `resource` 资源进行关联，`Resource` 是当前 Ingress 对象命名空间下引用的另外一个 Kubernetes 资源对象，但是需要注意的是 `Resource` 与 `Service` 配置是互斥的，只能配置一个，`Resource` 后端的一种常见用法是将所有入站数据导向带有静态资产的对象存储后端，如下所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  rules:
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```

该 Ingress 资源对象描述了所有的 `/icons` 请求会被路由到同命名空间下的名为 `icon-assets` 的 `StorageBucket` 资源中去进行处理。



### pathType

上面的示例中在定义路径规则的时候都指定了一个 `pathType` 的字段，事实上每个路径都需要有对应的路径类型，当前支持的路径类型有三种：

- `ImplementationSpecific`：该路径类型的匹配方法取决于 `IngressClass`，具体实现可以将其作为单独的 pathType 处理或者与 `Prefix` 或 `Exact` 类型作相同处理。
- `Exact`：精确匹配 URL 路径，且区分大小写。
- `Prefix`：基于以 `/` 分隔的 URL 路径前缀匹配，匹配区分大小写，并且对路径中的元素逐个完成，路径元素指的是由 `/` 分隔符分隔的路径中的标签列表。

`Exact` 比较简单，就是需要精确匹配 URL 路径，对于 `Prefix` 前缀匹配，需要注意如果路径的最后一个元素是请求路径中最后一个元素的子字符串，则不会匹配，例如 `/foo/bar` 可以匹配 `/foo/bar/baz`, 但不匹配 `/foo/barbaz`，可以查看下表了解更多的匹配场景（来自官网）：

![示例](Ingress.assets/20211214171445.png)

> 在某些情况下，Ingress 中的多条路径会匹配同一个请求，这种情况下最长的匹配路径优先，如果仍然有两条同等的匹配路径，则精确路径类型优先于前缀路径类型。



### IngressClass

Kubernetes 1.18 起，正式提供了一个 `IngressClass` 资源，作用与 `kubernetes.io/ingress.class` 注解类似，因为可能在集群中有多个 Ingress 控制器，可以通过该对象来定义我们的控制器，例如：

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: nginx-ingress-internal-controller
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
```

其中重要的属性是 `metadata.name` 和 `spec.controller`，前者是这个 `IngressClass` 的名称，需要设置在 Ingress 中，后者是 Ingress 控制器的名称。

Ingress 中的 `spec.ingressClassName` 属性就可以用来指定对应的 IngressClass，并进而由 IngressClass 关联到对应的 Ingress 控制器，如：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: external-lb # 上面定义的 IngressClass 对象名称
  defaultBackend:
    service:
      name: myapp
      port:
        number: 80
```

不过需要注意的是 `spec.ingressClassName` 与老版本的 `kubernetes.io/ingress.class` 注解的作用并不完全相同，因为 `ingressClassName` 字段引用的是 `IngressClass` 资源的名称，`IngressClass` 资源中除了指定了 Ingress 控制器的名称之外，还可能会通过 `spec.parameters` 属性定义一些额外的配置。

比如 `parameters` 字段有一个 `scope` 和 `namespace` 字段，可用来引用特定于命名空间的资源，对 Ingress 类进行配置。 `scope` 字段默认为 `Cluster`，表示默认是集群作用域的资源。将 `scope` 设置为 `Namespace` 并设置 `namespace` 字段就可以引用某特定命名空间中的参数资源，比如：

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: nginx-ingress-internal-controller
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
    namespace: external-configuration
    scope: Namespace
```

由于一个集群中可能有多个 Ingress 控制器，所以我们还可以将一个特定的 `IngressClass` 对象标记为集群默认是 Ingress 类。只需要将一个 IngressClass 资源的 `ingressclass.kubernetes.io/is-default-class` 注解设置为 true 即可，这样未指定 `ingressClassName` 字段的 Ingress 就会使用这个默认的 IngressClass。

> 如果集群中有多个 `IngressClass` 被标记为默认，准入控制器将阻止创建新的未指定 `ingressClassName` 的 Ingress 对象。最好的方式还是确保集群中最多只能有一个 `IngressClass` 被标记为默认。



### TLS

Ingress 资源对象还可以用来配置 Https 的服务，可以通过设定包含 TLS 私钥和证书的 Secret 来保护 Ingress。 Ingress 只支持单个 TLS 端口 443，如果 Ingress 中的 TLS 配置部分指定了不同的主机，那么它们将根据通过 SNI TLS 扩展指定的主机名 （如果 Ingress 控制器支持 SNI）在同一端口上进行复用。需要注意 TLS Secret 必须包含名为 `tls.crt` 和 `tls.key` 的键名，例如：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 编码的 cert
  tls.key: base64 编码的 key
type: kubernetes.io/tls
```

在 Ingress 中引用此 Secret 将会告诉 Ingress 控制器使用 TLS 加密从客户端到负载均衡器的通道，我们需要确保创建的 TLS Secret 创建自包含 `https-example.foo.com` 的公用名称的证书，如下所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
    - hosts:
        - https-example.foo.com
      secretName: testsecret-tls
  rules:
    - host: https-example.foo.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service1
                port:
                  number: 80
```

现在我们了解了如何定义 Ingress 资源对象了，但是仅创建 Ingress 资源本身没有任何效果。还需要部署 Ingress 控制器，例如 `ingress-nginx`，现在可以供大家使用的 Ingress 控制器有很多，比如 traefik、nginx-controller、Kubernetes Ingress Controller for Kong、HAProxy Ingress controller，当然你也可以自己实现一个 Ingress Controller，现在普遍用得较多的是 traefik 和 ingress-nginx，traefik 的性能比 ingress-nginx 差，但是配置使用要简单许多，我们这里会重点给大家介绍 ingress-nginx、traefik 以及 apisix 的使用。

> 实际上社区目前还在开发一组高配置能力的 API，被称为 [Service API](https://gateway-api.sigs.k8s.io/)，新 API 会提供一种 Ingress 的替代方案，它的存在目的不是替代 Ingress，而是提供一种更具配置能力的新方案。



# ingress-nginx

我们已经了解了 Ingress 资源对象只是一个路由请求描述配置文件，要让其真正生效还需要对应的 Ingress 控制器才行，Ingress 控制器有很多，这里我们先介绍使用最多的 [ingress-nginx](https://kubernetes.github.io/ingress-nginx/)，它是基于 Nginx 的 Ingress 控制器。



## 运行原理

`ingress-nginx` 控制器主要是用来组装一个 `nginx.conf` 的配置文件，当配置文件发生任何变动的时候就需要重新加载 Nginx 来生效，但是并不会只在影响 `upstream` 配置的变更后就重新加载 Nginx，控制器内部会使用一个 `lua-nginx-module` 来实现该功能。

我们知道 Kubernetes 控制器使用控制循环模式来检查控制器中所需的状态是否已更新或是否需要变更，所以 `ingress-nginx` 需要使用集群中的不同对象来构建模型，比如 Ingress、Service、Endpoints、Secret、ConfigMap 等可以生成反映集群状态的配置文件的对象，控制器需要一直 Watch 这些资源对象的变化，但是并没有办法知道特定的更改是否会影响到最终生成的 `nginx.conf` 配置文件，所以一旦 Watch 到了任何变化控制器都必须根据集群的状态重建一个新的模型，并将其与当前的模型进行比较，如果模型相同则就可以避免生成新的 Nginx 配置并触发重新加载，否则还需要检查模型的差异是否只和端点有关，如果是这样，则需要使用 HTTP POST 请求将新的端点列表发送到在 Nginx 内运行的 Lua 处理程序，并再次避免生成新的 Nginx 配置并触发重新加载，如果运行和新模型之间的差异不仅仅是端点，那么就会基于新模型创建一个新的 Nginx 配置了，这样构建模型最大的一个好处就是在状态没有变化时避免不必要的重新加载，可以节省大量 Nginx 重新加载。

下面简单描述了需要重新加载的一些场景：

- 创建了新的 Ingress 资源
- TLS 添加到现有 Ingress
- 从 Ingress 中添加或删除 path 路径
- Ingress、Service、Secret 被删除了
- Ingress 的一些缺失引用对象变可用了，例如 Service 或 Secret
- 更新了一个 Secret

对于集群规模较大的场景下频繁的对 Nginx 进行重新加载显然会造成大量的性能消耗，所以要尽可能减少出现重新加载的场景。



## 安装

由于 `ingress-nginx` 所在的节点需要能够访问外网，这样域名可以解析到这些节点上直接使用，所以需要让 `ingress-nginx` 绑定节点的 80 和 443 端口，所以可以使用 hostPort 来进行访问，当然对于线上环境来说为了保证高可用，一般是需要运行多个 ingress-nginx 实例的，然后可以用一个 nginx/haproxy 作为入口，通过 keepalived 来访问边缘节点的 vip 地址。

> 所谓的边缘节点即集群内部用来向集群外暴露服务能力的节点，集群外部的服务通过该节点来调用集群内部的服务，边缘节点是集群内外交流的一个 Endpoint。

安装 ingress-nginx 有多种方式，我们这里直接使用下面的命令进行一键安装：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
```

上面的命令执行后会自动创建一个名为 ingress-nginx 的命名空间，会生成如下几个 Pod：

```bash
[root@k8s-master01 ~]# kubectl get pods -n ingress-nginx 
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-hqfwp        0/1     Completed   0          26s
ingress-nginx-admission-patch-42td2         0/1     Completed   0          26s
ingress-nginx-controller-75f8595755-zwrbw   1/1     Running     0          26s
```

此外还会创建如下两个 Service 对象：

```bash
[root@k8s-master01 ~]# kubectl get svc -n ingress-nginx 
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.109.214.76   <pending>     80:32430/TCP,443:32753/TCP   46s
ingress-nginx-controller-admission   ClusterIP      10.106.69.205   <none>        443/TCP                      46s
```

其中 ingress-nginx-controller-admission 是为准入控制器提供服务的，我们也是强烈推荐开启该准入控制器，这样当我们创建不合要求的 Ingress 对象后就会直接被拒绝了，另外一个 ingress-nginx-controller 就是ingress 控制器对外暴露的服务，我们可以看到默认是一个 LoadBalancer 类型的 Service，我们知道该类型是用于云服务商的，我们这里在本地环境，暂时不能使用，但是可以通过他的 NodePort 来对外暴露，后面我们会提供在本地测试环境提供 LoadBalancer 的方式。

到这里 ingress-nginx 就部署成功了，安装完成后还会创建一个名为 nginx 的 IngressClass 对象：

```bash
[root@k8s-master01 ~]# kubectl get ingressclass
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       75s

➜ kubectl get ingressclass nginx -o yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
 labels:
 app.kubernetes.io/component: controller
 app.kubernetes.io/instance: ingress-nginx
 app.kubernetes.io/name: ingress-nginx
 app.kubernetes.io/part-of: ingress-nginx
 app.kubernetes.io/version: 1.5.1
 name: nginx
spec:
 controller: k8s.io/ingress-nginx
```

这里我们只提供了一个 controller 属性，对应的值和 ingress-nginx 的启动参数中的 controller-class 一致的。

```yaml
# 官方文件deploy.yaml中的ingress-nginx-controller部分
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.5.1
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  minReadySeconds: 0
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/name: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: ingress-nginx
        app.kubernetes.io/name: ingress-nginx
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx #
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
```



## 示例

安装成功后，现在我们来为一个 nginx 应用创建一个 Ingress 资源，如下所示：

```yaml
# my-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
        - name: my-nginx
          image: nginx
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  ports:
    - port: 80
      protocol: TCP
      name: http
  selector:
    app: my-nginx
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-nginx
  namespace: default
spec:
  ingressClassName: nginx # 使用名为 nginx 的 IngressClass（关联的 ingress-nginx 控制器）
  rules:
    - host: first-ingress.172.18.0.2.nip.io # 将域名映射到名为 my-nginx 的svc
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: # 将所有请求发送到名为 my-nginx 的svc的 80 端口
                name: my-nginx
                port:
                  number: 80
# 不过需要注意大部分Ingress控制器都不是直接转发到Service
# 而是只是通过Service来获取后端的Endpoints列表，直接转发到Pod，这样可以减少网络跳转，提高性能
```

注意我们这里配置的域名是 first-ingress.172.18.0.2.nip.io，该地址其实会直接映射到 172.18.0.2 上面，该 IP 地址就是我的 Node 节点地址（必须是ingress控制器所在的节点），因为我们这里 ingress 控制器是通过 NodePort 对外进行暴露的，所以可以通过 域名:nodePort 来访问服务。nip.io 是由 PowerDNS 提供支持的开源服务，允许我们可以直接通过使用以下格式将任何 IP 地址映射到主机名，这样我们就不需要在 etc/hosts 文件中配置映射了，对于 Ingress 测试非常方便。

这里直接创建上面的资源对象即可：

```bash
➜ kubectl apply -f my-nginx.yaml
deployment.apps/my-nginx created
service/my-nginx created
ingress.networking.k8s.io/my-nginx created

➜ kubectl get ingress
NAME       CLASS   HOSTS                             ADDRESS   PORTS   AGE
my-nginx   nginx   first-ingress.172.18.0.2.nip.io             80      6s
```

在上面的 Ingress 资源对象中我们使用配置 ingressClassName: nginx 指定让我们安装的 ingress-nginx 这个控制器来处理我们的 Ingress 资源，配置的匹配路径类型为前缀的方式去匹配 / ，将来自域名 first-ingress.172.18.0.2.nip.io 的所有请求转发到 my-nginx 服务的后端 Endpoints 中去，注意访问的时候需要带上 NodePort 端口。

```bash
# 端口要和上面查看的loadbalancer的service的一致
# curl first-ingress.172.18.0.2.nip.io:30877
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

下图显示了客户端是如何通过 Ingress 控制器连接到其中一个 Pod 的流程，客户端首先对 `ngdemo.qikqiak.com` 执行 DNS 解析，得到 Ingress 控制器所在节点的 IP，然后客户端向 Ingress 控制器发送 HTTP 请求，然后根据 Ingress 对象里面的描述匹配域名，找到对应的 Service 对象，并获取关联的 Endpoints 列表，将客户端的请求转发给其中一个 Pod。

![ingress controller workflow](Ingress.assets/ingress-controller-workflow.png)

前面我们也提到了 ingress-nginx 控制器的核心原理就是将我们的 Ingress 这些资源对象映射翻译成 Nginx 配置文件 nginx.conf ，我们可以通过查看控制器中的配置文件来验证这点：

```bash
[root@k8s-master01 ~]# kubectl exec -it ingress-nginx-controller-75f8595755-l52qp -n ingress-nginx -- cat /etc/nginx/nginx.conf
......
	upstream upstream_balancer {
		### Attention!!!
		#
		# We no longer create "upstream" section for every backend.
		# Backends are handled dynamically using Lua. If you would like to debug
		# and see what backends ingress-nginx has in its memory you can
		# install our kubectl plugin https://kubernetes.github.io/ingress-nginx/kubectl-plugin.
		# Once you have the plugin you can use "kubectl ingress-nginx backends" command to
		# inspect current backends.
		#
		###
		
		server 0.0.0.1; # placeholder
		
		balancer_by_lua_block {
			balancer.balance()
		}
		
		keepalive 320;
		keepalive_time 1h;
		keepalive_timeout  60s;
		keepalive_requests 10000;
		
	}
	......
	## start server first-ingress.10.0.0.121.nip.io
	server {
		server_name first-ingress.10.0.0.121.nip.io ;
		
		listen 80  ;
		listen [::]:80  ;
		listen 443  ssl http2 ;
		listen [::]:443  ssl http2 ;
		
		set $proxy_upstream_name "-";
		
		ssl_certificate_by_lua_block {
			certificate.call()
		}
		
		location / {
			
			set $namespace      "default";
			set $ingress_name   "my-nginx";
			set $service_name   "my-nginx";
			set $service_port   "80";
			set $location_path  "/";
			set $global_rate_limit_exceeding n;
			
			rewrite_by_lua_block {
				lua_ingress.rewrite({
					force_ssl_redirect = false,
					ssl_redirect = true,
					force_no_ssl_redirect = false,
					preserve_trailing_slash = false,
					use_port_in_redirects = false,
					global_throttle = { namespace = "", limit = 0, window_size = 0, key = { }, ignored_cidrs = { } },
				})
				balancer.rewrite()
				plugins.run()
			}
			
			# be careful with `access_by_lua_block` and `satisfy any` directives as satisfy any
			# will always succeed when there's `access_by_lua_block` that does not have any lua code doing `ngx.exit(ngx.DECLINED)`
			# other authentication method such as basic auth or external auth useless - all requests will be allowed.
			#access_by_lua_block {
			#}
			
			header_filter_by_lua_block {
				lua_ingress.header()
				plugins.run()
			}
			
			body_filter_by_lua_block {
				plugins.run()
			}
			
			log_by_lua_block {
				balancer.log()
				
				monitor.call()
				
				plugins.run()
			}
			
			port_in_redirect off;
			
			set $balancer_ewma_score -1;
			set $proxy_upstream_name "default-my-nginx-80";
			set $proxy_host          $proxy_upstream_name;
			set $pass_access_scheme  $scheme;
			
			set $pass_server_port    $server_port;
			
			set $best_http_host      $http_host;
			set $pass_port           $pass_server_port;
			
			set $proxy_alternative_upstream_name "";
			
			client_max_body_size                    1m;
			
			proxy_set_header Host                   $best_http_host;
			
			# Pass the extracted client certificate to the backend
			
			# Allow websocket connections
			proxy_set_header                        Upgrade           $http_upgrade;
			
			proxy_set_header                        Connection        $connection_upgrade;
			
			proxy_set_header X-Request-ID           $req_id;
			proxy_set_header X-Real-IP              $remote_addr;
			
			proxy_set_header X-Forwarded-For        $remote_addr;
			
			proxy_set_header X-Forwarded-Host       $best_http_host;
			proxy_set_header X-Forwarded-Port       $pass_port;
			proxy_set_header X-Forwarded-Proto      $pass_access_scheme;
			proxy_set_header X-Forwarded-Scheme     $pass_access_scheme;
			
			proxy_set_header X-Scheme               $pass_access_scheme;
			
			# Pass the original X-Forwarded-For
			proxy_set_header X-Original-Forwarded-For $http_x_forwarded_for;
			
			# mitigate HTTPoxy Vulnerability
			# https://www.nginx.com/blog/mitigating-the-httpoxy-vulnerability-with-nginx/
			proxy_set_header Proxy                  "";
			
			# Custom headers to proxied server
			
			proxy_connect_timeout                   5s;
			proxy_send_timeout                      60s;
			proxy_read_timeout                      60s;
			
			proxy_buffering                         off;
			proxy_buffer_size                       4k;
			proxy_buffers                           4 4k;
			
			proxy_max_temp_file_size                1024m;
			
			proxy_request_buffering                 on;
			proxy_http_version                      1.1;
			
			proxy_cookie_domain                     off;
			proxy_cookie_path                       off;
			
			# In case of errors try the next upstream server before returning an error
			proxy_next_upstream                     error timeout;
			proxy_next_upstream_timeout             0;
			proxy_next_upstream_tries               3;
			
			proxy_pass http://upstream_balancer;
			
			proxy_redirect                          off;
			
		}
		
	}
	## end server first-ingress.10.0.0.121.nip.io
	......
```

我们可以在 nginx.conf 配置文件中看到上面我们新增的 Ingress 资源对象的相关配置信息，不过需要注意的是现在并不会为每个 backend 后端都创建一个 upstream 配置块，现在是使用 Lua 程序进行动态处理的，所以我们没有直接看到后端的 Endpoints 相关配置数据。

此外我们也可以安装一个 kubectl 插件 https://kubernetes.github.io/ingress-nginx/kubectl-plugin 来辅助使用 ingress-nginx，要安装该插件的前提需要先安装 krew，然后执行下面的命令即可：

```bash
➜ kubectl krew install ingress-nginx
```



## Nginx 配置

如果我们还想进行一些自定义配置，则有几种方式可以实现：使用 Configmap 在 Nginx 中设置全局配置、通过 Ingress 的 Annotations 设置特定 Ingress 的规则、自定义模板。接下来我们重点给大家介绍使用注解来对 Ingress 对象进行自定义。

### Basic Auth

我们可以在 Ingress 对象上配置一些基本的 Auth 认证，比如 Basic Auth，可以用 `htpasswd` 生成一个密码文件来验证身份验证。

```shell
➜ htpasswd -c auth foo
New password:
Re-type new password:
Adding password for user foo
```

然后根据上面的 auth 文件创建一个 secret 对象：

```shell
➜ kubectl create secret generic basic-auth --from-file=auth
secret/basic-auth created

➜ kubectl get secret basic-auth -o yaml
apiVersion: v1
data:
  auth: Zm9vOiRhcHIxJFUxYlFZTFVoJHdIZUZQQ1dyZTlGRFZONTQ0dXVQdC4K
kind: Secret
metadata:
  name: basic-auth
  namespace: default
type: Opaque
```

然后对上面的 my-nginx 应用创建一个具有 Basic Auth 的 Ingress 对象：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-auth
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic # 认证类型
    nginx.ingress.kubernetes.io/auth-secret: basic-auth # 包含 user/password 定义的 secret 对象名
    nginx.ingress.kubernetes.io/auth-realm: 'Please Input Your Username and Password' # 要显示的带有适当上下文的消息，说明需要身份验证的原因
spec:
  ingressClassName: nginx # 使用 nginx 的 IngressClass（关联的 ingress-nginx 控制器）
  rules:
    - host: bauth.qikqiak.com # 将域名映射到名为 my-nginx 的svc
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: # 将所有请求发送到名为 my-nginx 的svc的 80 端口
                name: my-nginx
                port:
                  number: 80
```

直接创建上面的资源对象，然后通过下面的命令或者在浏览器中直接打开配置的域名：

```shell
➜ kubectl get ingress
NAME                CLASS    HOSTS                        ADDRESS         PORTS   AGE
ingress-with-auth   nginx    bauth.qikqiak.com            192.168.31.31   80      6m55s

➜ curl -v http://192.168.31.31 -H 'Host: bauth.qikqiak.com'
*   Trying 192.168.31.31...
* TCP_NODELAY set
* Connected to 192.168.31.31 (192.168.31.31) port 80 (#0)
> GET / HTTP/1.1
> Host: bauth.qikqiak.com
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
< Date: Thu, 16 Dec 2021 10:49:03 GMT
< Content-Type: text/html
< Content-Length: 172
< Connection: keep-alive
< WWW-Authenticate: Basic realm="Authentication Required - foo"
<
<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx</center>
</body>
</html>
* Connection #0 to host 192.168.31.31 left intact
* Closing connection 0
```

我们可以看到出现了 401 认证失败错误，然后带上我们配置的用户名和密码进行认证：

```shell
➜ curl -v http://192.168.31.31 -H 'Host: bauth.qikqiak.com' -u 'foo:foo'
*   Trying 192.168.31.31...
* TCP_NODELAY set
* Connected to 192.168.31.31 (192.168.31.31) port 80 (#0)
* Server auth using Basic with user 'foo'
> GET / HTTP/1.1
> Host: bauth.qikqiak.com
> Authorization: Basic Zm9vOmZvbw==
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Thu, 16 Dec 2021 10:49:38 GMT
< Content-Type: text/html
< Content-Length: 615
< Connection: keep-alive
< Last-Modified: Tue, 02 Nov 2021 14:49:22 GMT
< ETag: "61814ff2-267"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host 192.168.31.31 left intact
* Closing connection 0
```

可以看到已经认证成功了。除了可以使用我们自己在本地集群创建的 Auth 信息之外，还可以使用外部的 Basic Auth 认证信息，比如我们使用 `https://httpbin.org` 的外部 Basic Auth 认证，创建如下所示的 Ingress 资源对象：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # 配置外部认证服务地址
    nginx.ingress.kubernetes.io/auth-url: https://httpbin.org/basic-auth/user/passwd
  name: external-auth
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: external-bauth.qikqiak.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-nginx
                port:
                  number: 80
```

上面的资源对象创建完成后，再进行简单的测试：

```shell
➜ kubectl get ingress
NAME                CLASS    HOSTS                        ADDRESS         PORTS   AGE
external-auth       <none>   external-bauth.qikqiak.com                   80      72s

➜ curl -k http://192.168.31.31 -v -H 'Host: external-bauth.qikqiak.com'
*   Trying 192.168.31.31...
* TCP_NODELAY set
* Connected to 192.168.31.31 (192.168.31.31) port 80 (#0)
> GET / HTTP/1.1
> Host: external-bauth.qikqiak.com
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
< Date: Thu, 16 Dec 2021 10:57:25 GMT
< Content-Type: text/html
< Content-Length: 172
< Connection: keep-alive
< WWW-Authenticate: Basic realm="Fake Realm"
<
<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx</center>
</body>
</html>
* Connection #0 to host 192.168.31.31 left intact
* Closing connection 0
```

然后使用正确的用户名和密码测试：

```shell
➜ curl -k http://192.168.31.31 -v -H 'Host: external-bauth.qikqiak.com' -u 'user:passwd'
*   Trying 192.168.31.31...
* TCP_NODELAY set
* Connected to 192.168.31.31 (192.168.31.31) port 80 (#0)
* Server auth using Basic with user 'user'
> GET / HTTP/1.1
> Host: external-bauth.qikqiak.com
> Authorization: Basic dXNlcjpwYXNzd2Q=
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Thu, 16 Dec 2021 10:58:31 GMT
< Content-Type: text/html
< Content-Length: 615
< Connection: keep-alive
< Last-Modified: Tue, 02 Nov 2021 14:49:22 GMT
< ETag: "61814ff2-267"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host 192.168.31.31 left intact
* Closing connection 0
```

如果用户名或者密码错误则同样会出现 401 的状态码：

```shell
➜ curl -k http://192.168.31.31 -v -H 'Host: external-bauth.qikqiak.com' -u 'user:passwd123'
*   Trying 192.168.31.31...
* TCP_NODELAY set
* Connected to 192.168.31.31 (192.168.31.31) port 80 (#0)
* Server auth using Basic with user 'user'
> GET / HTTP/1.1
> Host: external-bauth.qikqiak.com
> Authorization: Basic dXNlcjpwYXNzd2QxMjM=
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
< Date: Thu, 16 Dec 2021 10:59:18 GMT
< Content-Type: text/html
< Content-Length: 172
< Connection: keep-alive
* Authentication problem. Ignoring this.
< WWW-Authenticate: Basic realm="Fake Realm"
<
<html>
<head><title>401 Authorization Required</title></head>
<body>
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx</center>
</body>
</html>
* Connection #0 to host 192.168.31.31 left intact
* Closing connection 0
```

当然除了 Basic Auth 这一种简单的认证方式之外，`ingress-nginx` 还支持一些其他高级的认证，比如我们可以使用 GitHub OAuth 来认证 Kubernetes 的 Dashboard。



### URL Rewrite

`ingress-nginx` 很多高级的用法可以通过 Ingress 对象的 `annotation` 进行配置，比如常用的 URL Rewrite 功能。很多时候我们会将 `ingress-nginx` 当成网关使用，比如对访问的服务加上 `/app` 这样的前缀，在 `nginx` 的配置里面我们知道有一个 `proxy_pass` 指令可以实现：

```nginx
location /app/ {
  proxy_pass http://127.0.0.1/remote/;
}
```

`proxy_pass` 后面加了 `/remote` 这个路径，此时会将匹配到该规则路径中的 `/app` 用 `/remote` 替换掉，相当于截掉路径中的 `/app`。同样的在 Kubernetes 中使用 `ingress-nginx` 又该如何来实现呢？我们可以使用 `rewrite-target` 的注解来实现这个需求，比如现在我们想要通过 `rewrite.qikqiak.com/gateway/` 来访问到 Nginx 服务，则我们需要对访问的 URL 路径做一个 Rewrite，在 PATH 中添加一个 gateway 的前缀，关于 Rewrite 的操作在 [ingress-nginx 官方文档](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)中也给出对应的说明:

![ingress nginx rewrite](Ingress.assets/ingress-nginx-rewrite-list.png)

按照要求我们需要在 `path` 中匹配前缀 `gateway`，然后通过 `rewrite-target` 指定目标，Ingress 对象如下所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2 # 将原始请求路径中第二个捕获组匹配到的内容，拼接到 / 后面，作为转发到后端服务的路径
spec:
  ingressClassName: nginx
  rules:
    - host: rewrite.qikqiak.com
      http:
        paths:
          # 第一个捕获组 (/|$)：匹配 / 或 gateway 结尾（用于处理 /gateway 和 /gateway/ 两种情况），第二个捕获组 (.*)：匹配 /gateway 之后的任意字符（这是核心需要保留的部分）
          - path: /gateway(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: my-nginx
                port:
                  number: 80
```

更新后，我们可以预见到直接访问域名肯定是不行了，因为我们没有匹配 `/` 的 path 路径：

```shell
➜ curl rewrite.qikqiak.com
default backend - 404
```

但是我们带上 `gateway` 的前缀再去访问:

![ingress nginx rewrite](Ingress.assets/20211219180809.png)

我们可以看到已经可以访问到了，这是因为我们在 `path` 中通过正则表达式 `/gateway(/|$)(.*)` 将匹配的路径设置成了 `rewrite-target` 的目标路径了，所以我们访问 `rewite.qikqiak.com/gateway/` 的时候实际上相当于访问的就是后端服务的 `/` 路径。

要解决我们访问主域名出现 404 的问题，我们可以给应用设置一个 `app-root` 的注解，这样当我们访问主域名的时候会自动跳转到我们指定的 `app-root` 目录下面，如下所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  annotations:
    nginx.ingress.kubernetes.io/app-root: /gateway/
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: rewrite.qikqiak.com
      http:
        paths:
          - path: /gateway(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: my-nginx
                port:
                  number: 80
```

这个时候我们更新应用后访问主域名 `rewrite.qikqiak.com` 就会自动跳转到 `rewrite.qikqiak.com/gateway/` 路径下面去了。但是还有一个问题是我们的 path 路径其实也匹配了 `/app` 这样的路径，可能我们更加希望我们的应用在最后添加一个 `/` 这样的 slash，同样我们可以通过 `configuration-snippet` 配置来完成，如下 Ingress 对象：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  annotations:
    nginx.ingress.kubernetes.io/app-root: /gateway/
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^(/gateway)$ $1/ redirect;
spec:
  ingressClassName: nginx
  rules:
    - host: rewrite.qikqiak.com
      http:
        paths:
          - path: /gateway(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: my-nginx
                port:
                  number: 80
```

更新后我们的应用就都会以 `/` 这样的 slash 结尾了。这样就完成了我们的需求，如果你原本对 nginx 的配置就非常熟悉的话应该可以很快就能理解这种配置方式了。



### 灰度发布

在日常工作中我们经常需要对服务进行版本更新升级，所以我们经常会使用到滚动升级、蓝绿发布、灰度发布等不同的发布操作。而 `ingress-nginx` 支持通过 Annotations 配置来实现不同场景下的灰度发布和测试，可以满足金丝雀发布、蓝绿部署与 A/B 测试等业务场景。

[ingress-nginx 的 Annotations](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary) 支持以下 4 种 Canary 规则：

- `nginx.ingress.kubernetes.io/canary-by-header`：基于 Request Header 的流量切分，适用于灰度发布以及 A/B 测试。当 Request Header 设置为 always 时，请求将会被一直发送到 Canary 版本；当 Request Header 设置为 never 时，请求不会被发送到 Canary 入口；对于任何其他 Header 值，将忽略 Header，并通过优先级将请求与其他金丝雀规则进行优先级的比较。
- `nginx.ingress.kubernetes.io/canary-by-header-value`：要匹配的 Request Header 的值，用于通知 Ingress 将请求路由到 Canary Ingress 中指定的服务。当 Request Header 设置为此值时，它将被路由到 Canary 入口。该规则允许用户自定义 Request Header 的值，必须与上一个 annotation (`canary-by-header`) 一起使用。
- `nginx.ingress.kubernetes.io/canary-weight`：基于服务权重的流量切分，适用于蓝绿部署，权重范围 0 - 100 按百分比将请求路由到 Canary Ingress 中指定的服务。权重为 0 意味着该金丝雀规则不会向 Canary 入口的服务发送任何请求，权重为 100 意味着所有请求都将被发送到 Canary 入口。
- `nginx.ingress.kubernetes.io/canary-by-cookie`：基于 cookie 的流量切分，适用于灰度发布与 A/B 测试。用于通知 Ingress 将请求路由到 Canary Ingress 中指定的服务的 cookie。当 cookie 值设置为 always 时，它将被路由到 Canary 入口；当 cookie 值设置为 never 时，请求不会被发送到 Canary 入口；对于任何其他值，将忽略 cookie 并将请求与其他金丝雀规则进行优先级的比较。

> 需要注意的是金丝雀规则按优先顺序进行排序：`canary-by-header - > canary-by-cookie - > canary-weight`

总的来说可以把以上的四个 annotation 规则划分为以下两类：

- 基于权重的 Canary 规则 

![基于权重的 Canary 规则](Ingress.assets/20201213130834.png)

- 基于用户请求的 Canary 规则 

![基于用户请求的 Canary 规则](Ingress.assets/20201213130856.png)

下面我们通过一个示例应用来对灰度发布功能进行说明。

#### 基于权重

##### 1.部署 Production 应用

首先创建一个 production 环境的应用资源清单：

```yaml
# production.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production
  labels:
    app: production
spec:
  selector:
    matchLabels:
      app: production
  template:
    metadata:
      labels:
        app: production
    spec:
      containers:
        - name: production
          image: cnych/echoserver
          ports:
            - containerPort: 8080
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
---
apiVersion: v1
kind: Service
metadata:
  name: production
  labels:
    app: production
spec:
  ports:
    - port: 80
      targetPort: 8080
      name: http
  selector:
    app: production
```

然后创建一个用于 production 环境访问的 Ingress 资源对象：

```yaml
# production-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production
spec:
  ingressClassName: nginx
  rules:
    - host: echo.qikqiak.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: production
                port:
                  number: 80
```

直接创建上面的几个资源对象：

```shell
➜ kubectl apply -f production.yaml

➜ kubectl apply -f production-ingress.yaml

➜ kubectl get pods -l app=production
NAME                         READY   STATUS    RESTARTS   AGE
production-856d5fb99-d6bds   1/1     Running   0          2m50s

➜ kubectl get ingress
NAME         CLASS    HOSTS                ADDRESS        PORTS   AGE
production   <none>   echo.qikqiak.com     10.151.30.11   80      90s
```

应用部署成功后，将域名 `echo.qikqiak.com` 映射到 master1 节点（ingress-nginx 所在的节点）的 IP 即可正常访问应用：

```shell
➜ curl http://echo.qikqiak.com

Hostname: production-856d5fb99-d6bds

Pod Information:
    node name:  node1
    pod name:   production-856d5fb99-d6bds
    pod namespace:  default
    pod IP: 10.244.1.111

Server values:
    server_version=nginx: 1.13.3 - lua: 10008

Request Information:
    client_address=10.244.0.0
    method=GET
    real path=/
    query=
    request_version=1.1
    request_scheme=http
    request_uri=http://echo.qikqiak.com:8080/

Request Headers:
    accept=*/*
    host=echo.qikqiak.com
    user-agent=curl/7.64.1
    x-forwarded-for=171.223.99.184
    x-forwarded-host=echo.qikqiak.com
    x-forwarded-port=80
    x-forwarded-proto=http
    x-real-ip=171.223.99.184
    x-request-id=e680453640169a7ea21afba8eba9e116
    x-scheme=http

Request Body:
    -no body in request-
```



##### 2.创建 Canary 版本

参考将上述 Production 版本的 `production.yaml` 文件，再创建一个 Canary 版本的应用（除了名字和标签不一样，其他没什么区别）。

```yaml
# canary.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: canary
  labels:
    app: canary
spec:
  selector:
    matchLabels:
      app: canary
  template:
    metadata:
      labels:
        app: canary
    spec:
      containers:
        - name: canary
          image: cnych/echoserver
          ports:
            - containerPort: 8080
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
---
apiVersion: v1
kind: Service
metadata:
  name: canary
  labels:
    app: canary
spec:
  ports:
    - port: 80
      targetPort: 8080
      name: http
  selector:
    app: canary
```

接下来就可以通过配置 Annotation 规则进行流量切分了。



##### 3.Annotation 规则配置

**1. 基于权重**：基于权重的流量切分的典型应用场景就是蓝绿部署，可通过将权重设置为 0 或 100 来实现。例如，可将 Green 版本设置为主要部分，并将 Blue 版本的入口配置为 Canary。最初，将权重设置为 0，因此不会将流量代理到 Blue 版本。一旦新版本测试和验证都成功后，即可将 Blue 版本的权重设置为 100，即所有流量从 Green 版本转向 Blue。

创建一个基于权重的 Canary 版本的应用路由 Ingress 对象。

```yaml
# canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    nginx.ingress.kubernetes.io/canary: 'true' # 要开启灰度发布机制，首先需要启用 Canary
    nginx.ingress.kubernetes.io/canary-weight: '30' # 分配30%流量到当前Canary版本
spec:
  ingressClassName: nginx
  rules:
    - host: echo.qikqiak.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: canary
                port:
                  number: 80
```

直接创建上面的资源对象即可：

```shell
➜ kubectl apply -f canary.yaml

➜ kubectl apply -f canary-ingress.yaml

➜ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
canary-66cb497b7f-48zx4      1/1     Running   0          7m48s
production-856d5fb99-d6bds   1/1     Running   0          21m
......

➜ kubectl get svc
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
canary                     ClusterIP   10.106.91.106    <none>        80/TCP                       8m23s
production                 ClusterIP   10.105.182.15    <none>        80/TCP                       22m
......

➜ kubectl get ingress
NAME         CLASS    HOSTS                ADDRESS        PORTS   AGE
canary       <none>   echo.qikqiak.com     10.151.30.11   80      108s
production   <none>   echo.qikqiak.com     10.151.30.11   80      22m
```

Canary 版本应用创建成功后，接下来我们在命令行终端中来不断访问这个应用，观察 Hostname 变化：

```shell
➜ for i in $(seq 1 10); do curl -s echo.qikqiak.com | grep "Hostname"; done
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
```

由于我们给 Canary 版本应用分配了 30% 左右权重的流量，所以上面我们访问 10 次有 3 次访问到了 Canary 版本的应用，符合我们的预期。



#### 基于 Request Header

基于 Request Header 进行流量切分的典型应用场景即灰度发布或 A/B 测试场景。

在上面的 Canary 版本的 Ingress 对象中新增一条 annotation 配置 `nginx.ingress.kubernetes.io/canary-by-header: canary`（这里的 value 可以是任意值），使当前的 Ingress 实现基于 Request Header 进行流量切分，由于 `canary-by-header` 的优先级大于 `canary-weight`，所以会忽略原有的 `canary-weight` 的规则。

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: 'true' # 要开启灰度发布机制，首先需要启用 Canary
  nginx.ingress.kubernetes.io/canary-by-header: canary # 基于header的流量切分
  nginx.ingress.kubernetes.io/canary-weight: '30' # 会被忽略，因为配置了 canary-by-headerCanary版本
```

更新上面的 Ingress 资源对象后，我们在请求中加入不同的 Header 值，再次访问应用的域名。

> 注意：当 Request Header 设置为 never 或 always 时，请求将不会或一直被发送到 Canary 版本，对于任何其他 Header 值，将忽略 Header，并通过优先级将请求与其他 Canary 规则进行优先级的比较。

```shell
➜ for i in $(seq 1 10); do curl -s -H "canary: never" echo.qikqiak.com | grep "Hostname"; done
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
```

这里我们在请求的时候设置了 `canary: never` 这个 Header 值，所以请求没有发送到 Canary 应用中去。如果设置为其他值呢：

```shell
➜ for i in $(seq 1 10); do curl -s -H "canary: other-value" echo.qikqiak.com | grep "Hostname"; done
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
Hostname: production-856d5fb99-d6bds
Hostname: canary-66cb497b7f-48zx4
```

由于我们请求设置的 Header 值为 `canary: other-value`，所以 ingress-nginx 会通过优先级将请求与其他 Canary 规则进行优先级的比较，我们这里也就会进入 `canary-weight: "30"` 这个规则去。

这个时候我们可以在上一个 annotation (即 `canary-by-header`）的基础上添加一条 `nginx.ingress.kubernetes.io/canary-by-header-value: user-value` 这样的规则，就可以将请求路由到 Canary Ingress 中指定的服务了。

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: 'true' # 要开启灰度发布机制，首先需要启用 Canary
  nginx.ingress.kubernetes.io/canary-by-header-value: user-value
  nginx.ingress.kubernetes.io/canary-by-header: canary # 基于header的流量切分
  nginx.ingress.kubernetes.io/canary-weight: '30' # 分配30%流量到当前Canary版本
```

同样更新 Ingress 对象后，重新访问应用，当 Request Header 满足 `canary: user-value`时，所有请求就会被路由到 Canary 版本：

```shell
➜ for i in $(seq 1 10); do curl -s -H "canary: user-value" echo.qikqiak.com | grep "Hostname"; done
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
```



#### 基于 Cookie

与基于 Request Header 的 annotation 用法规则类似。例如在 A/B 测试场景下，需要让地域为北京的用户访问 Canary 版本。那么当 cookie 的 annotation 设置为 `nginx.ingress.kubernetes.io/canary-by-cookie: "users_from_Beijing"`，此时后台可对登录的用户请求进行检查，如果该用户访问源来自北京则设置 cookie `users_from_Beijing` 的值为 `always`，这样就可以确保北京的用户仅访问 Canary 版本。

同样我们更新 Canary 版本的 Ingress 资源对象，采用基于 Cookie 来进行流量切分，

```yaml
annotations:
  nginx.ingress.kubernetes.io/canary: 'true' # 要开启灰度发布机制，首先需要启用 Canary
  nginx.ingress.kubernetes.io/canary-by-cookie: 'users_from_Beijing' # 基于 cookie
  nginx.ingress.kubernetes.io/canary-weight: '30' # 会被忽略，因为配置了 canary-by-cookie
```

更新上面的 Ingress 资源对象后，我们在请求中设置一个 `users_from_Beijing=always` 的 Cookie 值，再次访问应用的域名。

```shell
➜ for i in $(seq 1 10); do curl -s -b "users_from_Beijing=always" echo.qikqiak.com | grep "Hostname"; done
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
Hostname: canary-66cb497b7f-48zx4
```

我们可以看到应用都被路由到了 Canary 版本的应用中去了，如果我们将这个 Cookie 值设置为 never，则不会路由到 Canary 应用中。



### HTTPS

如果我们需要用 HTTPS 来访问我们这个应用的话，就需要监听 443 端口了，同样用 HTTPS 访问应用必然就需要证书，这里我们用 `openssl` 来创建一个自签名的证书：

```shell
➜ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=foo.bar.com"
```

然后通过 Secret 对象来引用证书文件：

```shell
# 要注意证书文件名称必须是 tls.crt 和 tls.key
➜ kubectl create secret tls foo-tls --cert=tls.crt --key=tls.key
secret/who-tls created
```

这个时候我们就可以创建一个 HTTPS 访问应用的：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-with-auth
  annotations:
    # 认证类型
    nginx.ingress.kubernetes.io/auth-type: basic
    # 包含 user/password 定义的 secret 对象名
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    # 要显示的带有适当上下文的消息，说明需要身份验证的原因
    nginx.ingress.kubernetes.io/auth-realm: 'Please Input Your Username and Password' # 要显示的带有适当上下文的消息，说明需要身份验证的原因
spec:
  ingressClassName: nginx
  tls: # 配置 tls 证书
    - hosts:
        - foo.bar.com
      secretName: foo-tls
  rules:
    - host: foo.bar.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-nginx
                port:
                  number: 80
```

除了自签名证书或者购买正规机构的 CA 证书之外，我们还可以通过一些工具来自动生成合法的证书，[cert-manager](https://cert-manager.io/) 是一个云原生证书管理开源项目，可以用于在 Kubernetes 集群中提供 HTTPS 证书并自动续期，支持 `Let's Encrypt/HashiCorp/Vault` 这些免费证书的签发。在 Kubernetes 中，可以通过 Kubernetes Ingress 和 Let's Encrypt 实现外部服务的自动化 HTTPS。



### TCP 与 UDP

由于在 Ingress 资源对象中没有直接对 TCP 或 UDP 服务的支持，要在 `ingress-nginx` 中提供支持，需要在控制器启动参数中添加 `--tcp-services-configmap` 和 `--udp-services-configmap` 标志指向一个 ConfigMap，其中的 key 是要使用的外部端口，value 值是使用格式 `<namespace/service name>:<service port>:[PROXY]:[PROXY]` 暴露的服务，端口可以使用端口号或者端口名称，最后两个字段是可选的，用于配置 PROXY 代理。

比如现在我们要通过 `ingress-nginx` 来暴露一个 MongoDB 服务，首先创建如下的应用：

```yaml
# mongo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
  labels:
    app: mongo
spec:
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      volumes:
        - name: data
          emptyDir: {}
      containers:
        - name: mongo
          image: mongo:4.0
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: data
              mountPath: /data/db
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  selector:
    app: mongo
  ports:
    - port: 27017
```

直接创建上面的资源对象：

```shell
➜ kubectl apply -f mongo.yaml

➜ kubectl get svc
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
mongo           ClusterIP   10.98.117.228    <none>        27017/TCP   2m26s

➜ kubectl get pods -l app=mongo
NAME                     READY   STATUS    RESTARTS   AGE
mongo-84c587f547-gd7pv   1/1     Running   0          2m5s
```

现在我们要通过 `ingress-nginx` 来暴露上面的 MongoDB 服务，我们需要创建一个如下所示的 ConfigMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-tcp
  namespace: ingress-nginx
data:
  '27017': default/mongo:27017
```

然后在 `ingress-nginx` 的启动参数中添加 `--tcp-services-configmap=$(POD_NAMESPACE)/ingress-nginx-tcp` 这样的配置即可

```yaml
containers:
 - args:
 - /nginx-ingress-controller
 - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
 - --election-id=ingress-nginx-leader
 - --controller-class=k8s.io/ingress-nginx
 - --ingress-class=nginx
 - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
 - --tcp-services-configmap=$(POD_NAMESPACE)/ingress-nginx-tcp #
 - --validating-webhook=:8443
 - --validating-webhook-certificate=/usr/local/certificates/cert
 - --validating-webhook-key=/usr/local/certificates/key
 env:
```

重新部署即可。由于我们这里安装的 ingress-nginx 是通过 LoadBalancer 的 Service 暴露出去的，那么自然我们也需要通过 Service 去暴露我们这里的 tcp 端口，所以我们还需要更新 ingress-nginx 的 Service 对象，如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.5.1
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
    protocol.openelb.kubesphere.io/v1alpha1: layer2
    eip.openelb.kubesphere.io/v1alpha2: eip-pool
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  externalTrafficPolicy: Local
  ipFamilies:
    - IPv4
  ipFamilyPolicy: SingleStack
  ports:
    - appProtocol: http
      name: http
      port: 80
      protocol: TCP
      targetPort: http
    - appProtocol: https
      name: https
      port: 443
      protocol: TCP
      targetPort: https
    - name: mongo # 暴露 27017 端口
      port: 27017
      protocol: TCP
      targetPort: 27017
  selector:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
  type: LoadBalancer
```

更新该 Service 对象即可：

```shell
kubectl apply -f deploy.yaml

➜ kubectl get svc ingress-nginx-controller -n ingress-nginx
NAME 					TYPE 		 CLUSTER-IP 	EXTERNAL-IP 	PORT(S)  								  AGE
ingress-nginx-controller  LoadBalancer 10.96.127.133   172.18.0.10     80:30877/TCP,443:30615/TCP,27017:30696/TCP 	3d18h
```

现在我们就可以通过 LB 地址 172.18.0.10 加上暴露的 27017 端口去访问 Mongo 服务了，比如我们这里在节点上安装了 MongoDB 客户端 mongosh，使用命令 mongosh "mongodb://172.18.0.10:27017" 就可以访问到我们的Mongo 服务了：

```bash
➜ mongosh "mongodb://172.18.0.10:27017"
Current Mongosh Log ID: 63f0814f292e3b15b3db5e2d
Connecting to: mongodb://172.18.0.10:27017/?
directConnection=true&appName=mongosh+1.7.1
Using MongoDB: 4.0.28
Using Mongosh: 1.7.1
For mongosh info see: https://docs.mongodb.com/mongodb-shell/
# ......
test> show dbs
admin 32.00 KiB
config 12.00 KiB
local 32.00 KiB
test>
```

同样的我们也可以去查看最终生成的 `nginx.conf` 配置文件：

```shell
➜ kubectl exec -it ingress-nginx-controller-gc582 -n ingress-nginx -- cat /etc/nginx/nginx.conf
......
stream {
    ......
    # TCP services
    server {
            preread_by_lua_block {
                    ngx.var.proxy_upstream_name="tcp-default-mongo-27017";
            }
            listen                  27017;
            listen                  [::]:27017;
            proxy_timeout           600s;
            proxy_next_upstream     on;
            proxy_next_upstream_timeout 600s;
            proxy_next_upstream_tries   3;
            proxy_pass              upstream_balancer;
    }
    # UDP services
}
```

TCP 相关的配置位于 `stream` 配置块下面。从 Nginx 1.9.13 版本开始提供 UDP 负载均衡，同样我们也可以在 `ingress-nginx` 中来代理 UDP 服务，比如我们可以去暴露 `kube-dns` 的服务，同样需要创建一个如下所示的 ConfigMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-udp
  namespace: ingress-nginx
data:
  53: 'kube-system/kube-dns:53'
```

然后需要在 `ingress-nginx` 参数中添加一个 `- --udp-services-configmap=$(POD_NAMESPACE)/udp-services` 这样的配置，然后重新更新即可。



### 全局配置

除了可以通过 `annotations` 对指定的 Ingress 进行定制之外，我们还可以配置 `ingress-nginx` 的全局配置，在控制器启动参数中通过标志 `--configmap` 指定了一个全局的 ConfigMap 对象，我们可以将全局的一些配置直接定义在该对象中即可：

```yaml
containers:
  - args:
    - /nginx-ingress-controller
    - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
    ......
```

比如这里我们用于全局配置的 ConfigMap 名为 `ingress-nginx-controller`：

```shell
➜ kubectl get configmap -n ingress-nginx
NAME                        DATA   AGE
ingress-nginx-controller    1      5d2h
```

比如我们可以添加如下所示的一些常用配置：

```yaml
➜ kubectl edit configmap ingress-nginx-controller -n ingress-nginx
apiVersion: v1
data:
  allow-snippet-annotations: "true"
  client-header-buffer-size: 32k  # 注意不是下划线
  client-max-body-size: 5m
  use-gzip: "true"
  gzip-level: "7"
  large-client-header-buffers: 4 32k
  proxy-connect-timeout: 11s
  proxy-read-timeout: 12s
  keep-alive: "75"   # 启用keep-alive，连接复用，提高QPS
  keep-alive-requests: "100"
  upstream-keepalive-connections: "10000"
  upstream-keepalive-requests: "100"
  upstream-keepalive-timeout: "60"
  disable-ipv6: "true"
  disable-ipv6-dns: "true"
  max-worker-connections: "65535"
  max-worker-open-files: "10240"
kind: ConfigMap
......
```

修改完成后 Nginx 配置会自动重载生效，我们可以查看 `nginx.conf` 配置文件进行验证：

```shell
kubectl exec -it ingress-nginx-controller-gc582 -n ingress-nginx -- cat /etc/nginx/nginx.conf | grep large_client_header_buffers
        large_client_header_buffers     4 32k;
```

此外往往我们还需要对 `ingress-nginx` 部署的节点进行性能优化，修改一些内核参数，使得适配 Nginx 的使用场景，一般我们是直接去修改节点上的内核参数（可以参考官方博客 https://www.nginx.com/blog/tuning-nginx/ 进行调整），为了能够统一管理，我们可以使用 `initContainers` 来进行配置：

```yaml
initContainers:
- command:
  - /bin/sh
  - -c
  - |
    mount -o remount rw /proc/sys
    sysctl -w net.core.somaxconn=65535  # 积压队列设置，具体的配置视具体情况而定
    sysctl -w net.ipv4.tcp_tw_reuse=1
    sysctl -w net.ipv4.ip_local_port_range="1024 65535"
    sysctl -w fs.file-max=1048576
    sysctl -w fs.inotify.max_user_instances=16384
    sysctl -w fs.inotify.max_user_watches=524288
    sysctl -w fs.inotify.max_queued_events=16384
image: busybox
imagePullPolicy: IfNotPresent
name: init-sysctl
securityContext:
  capabilities:
    add:
    - SYS_ADMIN
    drop:
    - ALL
......
```

部署完成后通过 `initContainers` 就可以修改节点内核参数了，生产环境建议对节点内核参数进行相应的优化。性能优化需要有丰富的经验，关于 nginx 的性能优化可以参考文章https://cloud.tencent.com/developer/article/1026833



# Ingress、IngressClass与IngressController

**总结**

- **IngressClass 是全局资源**，无命名空间，可被所有命名空间的 Ingress 引用。
- **Ingress 是命名空间级资源**，但引用 IngressClass 时不受命名空间限制。
- **IngressController 是命名空间级资源**，但通过 IngressClass 的标识关联，与 Ingress、IngressClass 的命名空间无关。

因此，三者完全可以不在同一个命名空间下，这也是 Kubernetes 中常见的配置方式（例如 IngressController 部署在 `ingress-nginx` 命名空间，IngressClass 全局定义，Ingress 分布在各业务命名空间）。

**Ingress通过IngressClass绑定具体的IngressController**



# 让IngressController能被外部访问的三种方案

Ingress Controller 作为外部流量进入 Kubernetes 集群的 “入口网关”，必须被暴露到集群外部才能接收外部请求。你提到的 **NodePort**、**LoadBalancer**、**hostNetwork** 是三种最常用的暴露方式，它们适用于不同的环境和需求，下面详细解析每种方案的原理、配置、优缺点及适用场景：

## 一、NodePort  Service方式：通过节点节点端口暴露

原理

- NodePort 是 Kubernetes 内置的 Service 类型，原理是在集群的 **每个节点上开放一个静态端口（30000-32767 之间）**，外部流量通过 `任意节点 IP:NodePort` 即可访问到 Ingress Controller。
- 当外部请求发送到 `节点 IP:NodePort` 时，节点的 kube-proxy 会将流量转发到 Ingress Controller 的 Pod（通过 Service 关联的标签选择器匹配），再由 Ingress Controller 根据 Ingress 规则转发到后端 Service。

配置示例

**部署 Ingress Controller（以 Nginx 为例）**：

通常以 Deployment 或 DaemonSet 部署，Pod 监听 80（HTTP）和 443（HTTPS）端口。

**创建 NodePort 类型的 Service 关联 Ingress Controller**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-controller
spec:
  selector:
    app: nginx-ingress  # 匹配 Ingress Controller Pod 的标签
  type: NodePort
  ports:
  - name: http
    port: 80           # Service 暴露的端口（集群内部访问用）
    targetPort: 80     # 转发到 Ingress Controller Pod 的端口
    nodePort: 30080    # 节点上开放的静态端口（可选，不指定则随机分配）
  - name: https
    port: 443
    targetPort: 443
    nodePort: 30443
```

**外部访问方式**：

外部客户端通过 `节点 IP:30080`（HTTP）或 `节点 IP:30443`（HTTPS）访问 Ingress Controller，例如 `http://192.168.1.100:30080`（192.168.1.100 是集群任意节点的 IP）。

优缺点

- **优点**：
  - 无需依赖外部组件（纯 Kubernetes 内置功能），所有环境（物理机、虚拟机、云环境）都支持。
  - 配置简单，只需定义 NodePort 类型的 Service 即可。
- **缺点**：
  - 端口范围固定（30000-32767），可能与节点上其他服务的端口冲突。
  - 外部访问需指定 “节点 IP + 端口”，若节点 IP 变化（如节点故障替换），需手动更新客户端配置。
  - 缺乏负载均衡能力：若节点故障，客户端需手动切换到其他节点 IP，高可用性需额外方案（如外部 DNS 轮询）。

适用场景

- 测试环境或小型集群（无需高可用）。
- 没有负载均衡器（如物理机集群、私有云环境），且节点 IP 相对稳定的场景。



## 二、LoadBalancer  Service方式：通过云厂商负载均衡器暴露

原理

- LoadBalancer 是云环境（如 AWS、GCP、阿里云、Azure 等）特有的 Service 类型，原理是：当创建 `type: LoadBalancer` 的 Service 时，云平台会 **自动创建一个公网负载均衡器（LB）**，并将 LB 的公网 IP 绑定到 Service。
- 外部流量通过 LB 的公网 IP:80/443 访问，LB 会自动将流量分发到集群中运行 Ingress Controller 的节点（通过 NodePort 转发到 Ingress Controller Pod），再由 Ingress Controller 路由到后端 Service。

**外部负载均衡器（如阿里云 SLB、AWS ELB）确实可以 “直接” 将流量转发到 Ingress Controller 的 Pod，但在 Kubernetes 场景下，我们仍建议通过 Service（通常是 NodePort 或 LoadBalancer 类型）间接转发**。核心原因是 **Service 解决了 Pod 动态变化带来的 “地址漂移” 问题**

配置示例

**部署 Ingress Controller**：与 NodePort 方式相同，Pod 监听 80/443 端口。

**创建 LoadBalancer 类型的 Service**：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-controller
spec:
  selector:
    app: nginx-ingress
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
```

**外部访问方式**：

创建 Service 后，云平台会为其分配一个公网 IP（可通过 `kubectl get svc nginx-ingress-controller` 查看 `EXTERNAL-IP` 字段），外部客户端直接通过 `http://公网IP` 或 `https://公网IP` 访问。

优缺点

- **优点**：
  - 自带公网 IP 和负载均衡能力：LB 会自动检测节点健康状态，将流量分发到正常节点，实现高可用。
  - 无需手动管理端口：使用标准 80/443 端口，客户端访问无需指定端口（如 `https://www.example.com` 直接映射到 LB 公网 IP）。
  - 与云平台深度集成：支持自动扩缩容、SSL 卸载（部分云 LB 支持）等高级功能。

- **缺点**：
  - 仅支持云环境，物理机或私有云（无 LB 支持）无法使用。
  - 可能产生额外费用：云厂商的负载均衡器通常按使用收费。
  - 灵活性受限：LB 配置依赖云厂商实现，部分高级功能（如路径路由）需 Ingress Controller 而非 LB 处理。

适用场景

- 生产环境的云原生集群（AWS、阿里云等），对高可用性和稳定性要求高的场景。
- 需要通过公网域名（如 `www.example.com`）直接访问服务的场景。



## 三、hostNetwork 方式：直接使用宿主机网络

原理

- hostNetwork 是 Pod 级别的配置，原理是让 Ingress Controller 的 Pod **直接使用宿主机的网络命名空间**（而非独立的 Pod 网络），此时 Pod 监听的端口（80/443）会直接绑定到宿主机的网络接口上。
- 外部流量通过 `宿主机 IP:80/443` 直接访问 Ingress Controller（无需经过 Service 转发），再由 Ingress Controller 路由到后端 Service。

配置示例

**部署 Ingress Controller 时启用 hostNetwork**：

通常以 DaemonSet 部署（确保每个节点都运行一个副本，避免单点故障），并设置 `hostNetwork: true`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
spec:
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
    spec:
      hostNetwork: true  # 启用宿主机网络
      containers:
      - name: nginx-ingress-controller
        image: nginx/ingress-nginx:latest
        ports:
        - containerPort: 80  # 直接绑定到宿主机的 80 端口
        - containerPort: 443 # 直接绑定到宿主机的 443 端口
```

**外部访问方式**：

外部客户端通过 `宿主机 IP:80` 或 `宿主机 IP:443` 访问，例如 `http://192.168.1.100`（192.168.1.100 是运行 Ingress Controller 的宿主机 IP）。

优缺点

- **优点**：
  - 性能高：流量直接从宿主机网络进入 Pod，无需经过 Service 转发（减少网络开销）。
  - 支持标准端口：直接使用 80/443 端口，无需暴露 NodePort 范围的端口。
  - 不依赖 Service：无需创建 Service 即可暴露，适合对网络性能要求高的场景。
- **缺点**：
  - 端口冲突风险高：Pod 直接占用宿主机端口，若宿主机上已有其他服务（如本地 Nginx）使用 80/443 端口，会导致冲突。
  - 安全性降低：Pod 共享宿主机网络命名空间，可能获取宿主机网络的敏感信息。
  - 部署限制：通常需以 DaemonSet 部署（确保每个节点都有副本），否则若运行 Ingress Controller 的节点故障，流量会中断。

适用场景

- 对网络性能要求极高的场景（如高并发 API 网关）。
- 没有负载均衡器，且需要使用标准 80/443 端口的物理机或私有云集群。
- 边缘节点（如物联网设备）场景，需直接暴露宿主机网络的场景。

### 总结：三种方式的核心区别与选择建议

| 维度     | NodePort                     | LoadBalancer               | hostNetwork                 |
| -------- | ---------------------------- | -------------------------- | --------------------------- |
| 依赖环境 | 所有环境（通用）             | 仅云环境（依赖云 LB）      | 所有环境（通用）            |
| 访问方式 | 节点 IP:NodePort（如 30080） | 公网 IP:80/443             | 宿主机 IP:80/443            |
| 高可用性 | 需手动维护（如 DNS 轮询）    | 自动支持（云 LB 负载均衡） | 需 DaemonSet 部署 + 外部 LB |
| 性能     | 一般（经 Service 转发）      | 一般（经 LB + Service）    | 高（直接绑定宿主机端口）    |
| 适用场景 | 测试 / 小型集群              | 生产云环境                 | 高性能 / 边缘节点场景       |

**1. 第一种（NodePort） vs 第二种（LoadBalancer）的区别**

它们都用了 Service。但它们的**流量入口形态**和**依赖关系**完全不同。

**关键点拨：**

- **NodePort 是“穷人版”入口**：Kubernetes 只是在每个房子的侧面开了一个“侧门”（30000+ 端口）。你得自己记住每个房子的地址。
- **LoadBalancer 是“自动集成版”**：当你创建一个 `type: LoadBalancer` 的 Service 时，Kubernetes 会给云厂商发个指令：“帮我买个公网 VIP，并把它对接到我这些 NodePort 上。” **结论：** 第二种方法其实是在第一种（NodePort）的基础上，由云厂商自动在外面套了一个“官方大门”。

**2. 它们都要依赖外部负载均衡器吗？**

- **第一种 (NodePort)**：**不需要。** 它是自给自足的。只要你能访问到集群节点的 IP，就能进去。
- **第二种 (LoadBalancer)**：**需要。** 如果你在自己家里的物理机上创建 `type: LoadBalancer`，它的状态会一直卡在 `<pending>`，因为没人给它分配公网 IP。
  - *注：在私有环境（物理机）想用这种模式，通常要安装像 **MetalLB** 这样的插件来模拟云厂商的行为。*

**3. 第三种 (hostNetwork) 要依赖外部负载均衡器吗？**

**不需要依赖，但它“渴望”一个。**

`hostNetwork` 是最霸道的方式：Ingress Controller 的 Pod 直接抢占宿主机的 80 端口。

- **现象**：此时，宿主机本身就变成了“负载均衡器”。
- **问题**：如果你的集群有 3 台机器，Ingress 跑在 Node-A 上，那么只有 Node-A 的 80 端口能通，Node-B 和 Node-C 是不通的。

**为什么说它“渴望”外部负载均衡？** 因为在生产环境，你不可能让用户手动输入 Node-A 的 IP。通常你还是会在集群外面放一个硬件 LB（比如 F5）或者 DNS 轮询，把流量均匀地分发到这几台宿主机的 IP 上。

**4. 终极总结：流量到底是怎么进去的？**

为了让你一眼看透，我们按“谁是第一个接客的人”来排队：

1. **NodePort**：第一个接客的是**节点的内核 (iptables/kube-proxy)**。它在 30080 端口等着。
2. **LoadBalancer**：第一个接客的是**云厂商的设备**。它接完客，转交给节点的内核，内核再给 Pod。
3. **hostNetwork**：第一个接客的是 **Ingress Pod 进程本身**。因为它直接就在宿主机的 80 端口上“守株待兔”。

**NodePort 和 hostNetwork 这两种方案有一点是一样的，如果只有一个 ingress controller 节点可以不使用外部 LB，而如果有多个节点，则最好使用外部 LB，让流量可以自动并均匀地分摊到所有 ingress controller 上，并且LB还支持健康检查，如果某个节点挂了，LB可以自动屏蔽该节点；否则这两种方案都需要用户手动根据不同的节点输入不同的 ip**

# Ingress与Service的区别

- **Service**：解决 “集群内服务如何稳定访问动态 Pod” 的问题，是 Pod 的 “固定访问点 + 内部负载均衡器”。
- **Ingress**：解决 “外部流量如何智能路由到集群内 Service” 的问题，是外部流量的 “统一入口 + 规则管理器”。





