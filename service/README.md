# Service

## 动机

Kubernetes Pods 是有生命周期的。他们可以被创建，而且销毁不会再启动。 如果您使用 Deployment 来运行您的应用程序，则它可以动态创建和销毁 Pod。

每个 Pod 都有自己的 IP 地址，但是在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同。

这样就导致了一个问题： 如果一组 Pod（称为“后端”）为群集内的其他 Pod（称为“前端”）提供功能，那么前端如何找出并跟踪要连接的 IP 地址，以便前端可以使用后端功能？

## 基本概念

Kubernetes Service 定义了这样一种抽象：逻辑上的一组 Pod，一种可以访问它们的策略 —— 通常称为微服务。 这一组 Pod 能够被 Service 访问到，通常是通过 selector 实现的。举个例子，考虑一个图片处理 backend，它运行了3个副本。这些副本是可互换的 —— frontend 不需要关心它们调用了哪个 backend 副本。 然而组成这一组 backend 程序的 Pod 实际上可能会发生变化，frontend 客户端不应该也没必要知道，而且也不需要跟踪这一组 backend 的状态。 Service 定义的抽象能够解耦这种关联。

## VIP和Service 代理

Kubernetes集群中，每个Node运行一个kube-proxy进程。kube-proxy负责为Service实现了一种VIP（虚拟IP）的形式。

Kubernetes v1.0开始，使用 用户空间代理模式。 Kubernetes v1.1添加了 iptables 模式代理，在 Kubernetes v1.2 中，kube-proxy 的 iptables 模式成为默认设置。 Kubernetes v1.8添加了 ipvs 代理模式。

### userspace 代理模式

userspace模式下，kube-proxy 会监视 Kubernetes master 对 Service 对象和 Endpoints 对象的添加和移除。 对每个 Service，它会在本地 Node 上打开一个端口（随机选择）。 任何连接到“代理端口”的请求，都会被代理到 Service 的backend Pods 中的某个上面（如 Endpoints 所报告的一样）。 使用哪个 backend Pod，是 kube-proxy 基于 SessionAffinity 来确定的。

最后，它安装 iptables 规则，捕获到达该 Service 的 clusterIP（是虚拟 IP）和 Port 的请求，并重定向到代理端口，代理端口再代理请求到 backend Pod。

默认情况下，用户空间模式下的kube-proxy通过循环算法选择后端。

默认的策略是，通过 round-robin 算法来选择 backend Pod。

![services-userspace-overview](./services-userspace-overview.svg)




### iptables 代理模式

iptables模式下，kube-proxy 会监视 Kubernetes 控制节点对 Service 对象和 Endpoints 对象的添加和移除。 对每个 Service，它会安装 iptables 规则，从而捕获到达该 Service 的 clusterIP 和端口的请求，进而将请求重定向到 Service 的一组 backend 中的某个上面。 对于每个 Endpoints 对象，它也会安装 iptables 规则，这个规则会选择一个 backend 组合。

默认的策略是，kube-proxy 在 iptables 模式下随机选择一个 backend。

使用 iptables 处理流量具有较低的系统开销，因为流量由 Linux netfilter 处理，而无需在用户空间和内核空间之间切换。 这种方法也可能更可靠。

如果 kube-proxy 在 iptable s模式下运行，并且所选的第一个 Pod 没有响应，则连接失败。 这与用户空间模式不同：在这种情况下，kube-proxy 将检测到与第一个 Pod 的连接已失败，并会自动使用其他后端 Pod 重试。

您可以使用 Pod readiness 探测器 验证后端 Pod 可以正常工作，以便 iptables 模式下的 kube-proxy 仅看到测试正常的后端。 这样做意味着您避免将流量通过 kube-proxy 发送到已知已失败的Pod。

![services-iptables-overview](./services-iptables-overview.svg)

### IPVS 代理模式

ipvs 模式下，kube-proxy监视Kubernetes服务和端点，调用 netlink 接口相应地创建 IPVS 规则， 并定期将 IPVS 规则与 Kubernetes 服务和端点同步。 该控制循环可确保　IPVS　状态与所需状态匹配。 访问服务时，IPVS　将流量定向到后端Pod之一。

IPVS代理模式基于类似于 iptables 模式的 netfilter 挂钩函数，但是使用哈希表作为基础数据结构，并且在内核空间中工作。 这意味着，与 iptables 模式下的 kube-proxy 相比，IPVS 模式下的 kube-proxy 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。与其他代理模式相比，IPVS 模式还支持更高的网络流量吞吐量。

![services-ipvs-overview](./services-ipvs-overview.svg)

IPVS提供了更多选项来平衡后端Pod的流量。这些有：

- rr: round-robin
- lc: least connection (smallest number of open connections)
- dh: destination hashing
- sh: source hashing
- sed: shortest expected delay
- nq: never queue


## Service案例

server.yaml定义一组Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
        imagePullPolicy: IfNotPresent
```

service.yaml定义服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name:  nginx
  labels:
    run:  nginx
spec:
  selector:
    run:  nginx
  ports:
  - port:  80
    protocol: TCP
    targetPort:  80
```

client.yaml定义客户端

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: client
spec:
  containers:
  - name: client
    image: nginx:alpine
```



# 服务发现

## 集群内部访问服务

在集群里面，其他 pod 要怎么访问到我们所创建的这个 service 呢？有三种方式：

1. 第1种方式直接访问服务名，依靠 DNS 解析，就是同一个 namespace 里 pod 可以直接通过 service 的名字去访问到刚才所声明的这个 service。不同的 namespace 里面，我们可以通过 service 名字加“.”，然后加 service 所在的哪个 namespace 去访问这个 service，例如我们直接用 curl 去访问，就是 my-service:80 就可以访问到这个 service。

2. 第2种是通过环境变量访问，在同一个 namespace 里的 pod 启动时，K8s 会把 service 的一些 IP 地址、端口，以及一些简单的配置，通过环境变量的方式放到 K8s 的 pod 里面。在 K8s pod 的容器启动之后，通过读取系统的环境变量比读取到 namespace 里面其他 service 配置的一个地址，或者是它的端口号等等。比如在集群的某一个 pod 里面，可以直接通过 curl $ 取到一个环境变量的值，比如取到 MY_SERVICE_SERVICE_HOST 就是它的一个 IP 地址，MY_SERVICE 就是刚才我们声明的 MY_SERVICE，SERVICE_PORT 就是它的端口号，这样也可以请求到集群里面的 MY_SERVICE 这个 service。

3. 第3种方式可以通过 service 的虚拟 IP 去访问，比如说刚创建的 my-service 这个服务，通过 kubectl get svc 或者 kubectl discribe service 都可以看到它的虚拟 IP 地址是 172.29.3.27，端口是 80，然后就可以通过这个虚拟 IP 及端口在 pod 里面直接访问到这个 service 的地址。这种方式对访问服务的应用来说比较死板，已经被遗弃。

## 服务发现案例

下面的命令演示如何在namespace内部访问上述服务定义的功能:
```shell
yczhang@yczhang:~/workspace/service$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   11h
nginx        ClusterIP   10.97.244.221   <none>        80/TCP    11h
yczhang@yczhang:~/workspace/service$ kubectl describe service nginx
Name:              nginx
Namespace:         default
Labels:            run=nginx
Annotations:       <none>
Selector:          run=nginx
Type:              ClusterIP
IP:                10.97.244.221
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         172.17.0.3:80,172.17.0.6:80
Session Affinity:  None
Events:            <none>
yczhang@yczhang:~/workspace/service$ kubectl exec -it client /bin/sh
/ # wget -q -O - 10.97.244.221
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
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ # wget -q -O - nginx
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
/ # wget -q -O - nginx
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
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ # env
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=client
SHLVL=1
HOME=/root
NGINX_PORT_80_TCP=tcp://10.97.244.221:80
PKG_RELEASE=1
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
NGINX_VERSION=1.17.10
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
NGINX_SERVICE_HOST=10.97.244.221
KUBERNETES_PORT_443_TCP_PORT=443
NJS_VERSION=0.3.9
KUBERNETES_PORT_443_TCP_PROTO=tcp
NGINX_SERVICE_PORT=80
NGINX_PORT=tcp://10.97.244.221:80
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
NGINX_PORT_80_TCP_ADDR=10.97.244.221
NGINX_PORT_80_TCP_PORT=80
NGINX_PORT_80_TCP_PROTO=tcp
/ # wget -q -O - $NGINX_SERVICE_HOST
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
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ #
```

# 服务发布

## 集群外访问服务

对一些应用（如 Frontend）的某些部分，可能希望通过外部Kubernetes 集群外部IP 地址暴露 Service。

Kubernetes `ServiceTypes` 允许指定一个需要的类型的 Service，默认是 `ClusterIP` 类型。

`Type` 的取值以及行为如下：

- `ClusterIP`：通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的 `ServiceType`。
- `NodePort`：通过每个 Node 上的 IP 和静态端口（`NodePort`）暴露服务。`NodePort` 服务会路由到 `ClusterIP` 服务，这个 `ClusterIP` 服务会自动创建。通过请求 `:`，可以从集群的外部访问一个 `NodePort` 服务。
- `LoadBalancer`：使用云提供商的负载局衡器，可以向外部暴露服务。外部的负载均衡器可以路由到 `NodePort` 服务和 `ClusterIP` 服务。
- `ExternalName`：通过返回 `CNAME` 和它的值，可以将服务映射到 `externalName` 字段的内容（例如， `foo.bar.example.com`）。 没有任何类型代理被创建。


## 服务发布案例

下面的命令演示集群外部如何访问服务：

```shell
yczhang@yczhang:~/workspace/service$ kubectl get service
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP        13h
nginx        LoadBalancer   10.97.244.221   <pending>     80:32074/TCP   12h
yczhang@yczhang:~/workspace/service$ minikube ip
192.168.99.100
yczhang@yczhang:~/workspace/service$ wget -O - 192.168.99.100:32074
--2020-04-20 11:38:30--  http://192.168.99.100:32074/
正在连接 192.168.99.100:32074... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度： 612 [text/html]
正在保存至: “STDOUT”

-                                              0%[                                                                                            ]       0  --.-KB/s               <!DOCTYPE html>
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
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                                            100%[===========================================================================================>]     612  --.-KB/s    用时 0s

2020-04-20 11:38:30 (37.8 MB/s) - 已写入至标准输出 [612/612]

yczhang@yczhang:~/workspace/service$ cat service.yaml
apiVersion: v1
kind: Service
metadata:
  name:  nginx
  labels:
    run:  nginx
spec:
  selector:
    run:  nginx
  ports:
  - port:  80
    protocol: TCP
    targetPort:  80
  type: LoadBalancer

yczhang@yczhang:~/workspace/service$
```
上面的pending在官方文档上介绍是由于定制版的minikube是没有定制LoadBalancer功能，所以上面的访问方式实际使用的是NodePort方式在访问。

# Ingress

## 基本概念

将集群的服务提供给外部访问的另外一种方式是Ingress，Ingress是授权入站连接到达集群服务的规则集合，有点类似nginx的功能。你可以给Ingress配置提供外部可访问的URL、负载均衡、SSL、基于名称的虚拟主机等。

以Service的NodePort为入口，Service在转发到后端对应的Pod上提供服务
```
  Internet
------------
[ Services ]
--|------|--
[   Pod    ]
```
以Ingress为入口，Ingress动态检查对应Service的Pod(通过Endpoints),然后直接将请求转发到Pod
```
 Internet
    |
[ Ingress  ]
--|-----|--
[ Services ]
--|-----|--
[   Pod    ]
```
比较：
- Service NodePort: 后期维护困难，不支持虚拟路径

- Service LoadBlancer: 需要云厂商支持，有局限性

- Ingress: 灵活，无依赖



## Ingress案例

下面是创建Ingress的例子：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress
spec:
  rules:
  - host: yczhang.k8s
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
```

```shell
yczhang@yczhang:~/workspace/service$ kubectl get ingress
NAME      HOSTS         ADDRESS          PORTS   AGE
ingress   yczhang.k8s   192.168.99.100   80      10m
yczhang@yczhang:~/workspace/service$ echo "$(minikube ip) yczhang.k8s" | sudo tee -a /etc/hosts
yczhang@yczhang:~/workspace/service$ kubectl get pods -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-7f9c544f75-gvtkb                    1/1     Running   2          16h
coredns-7f9c544f75-rwb2g                    1/1     Running   2          16h
etcd-minikube                               1/1     Running   2          16h
kube-apiserver-minikube                     1/1     Running   2          16h
kube-controller-manager-minikube            1/1     Running   2          16h
kube-proxy-dxzqn                            1/1     Running   2          16h
kube-scheduler-minikube                     1/1     Running   2          16h
nginx-ingress-controller-6fc5bcc8c9-6tcld   1/1     Running   0          13m
storage-provisioner                         1/1     Running   3          16h
yczhang@yczhang:~/workspace/service$ wget -O - http://yczhang.k8s
--2020-04-20 14:55:06--  http://yczhang.k8s/
正在解析主机 yczhang.k8s (yczhang.k8s)... 192.168.99.100
正在连接 yczhang.k8s (yczhang.k8s)|192.168.99.100|:80... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度： 612 [text/html]
正在保存至: “STDOUT”

-                                    0%[                                                               ]       0  --.-KB/s               <!DOCTYPE html>
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
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                                  100%[==============================================================>]     612  --.-KB/s    用时 0s

2020-04-20 14:55:06 (66.1 MB/s) - 已写入至标准输出 [612/612]

yczhang@yczhang:~/workspace/service$
```

notice:minikube addons enable ingress
