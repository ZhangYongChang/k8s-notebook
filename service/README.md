# Service

# 基本概念

Kubernete Service是一个定义了一组Pod的策略的抽象，被服务标记的Pod一般通过label Selector决定的。

为什么要设计Service对象，因为Kubernetes Pod它门会被创建，也会死掉，并且他们是不可复活的，具有自身的生命周期。 ReplicationControllers动态的创建和销毁Pods(比如规模扩大或者缩小，或者执行动态更新)。每个pod都有自己的IP，这些IP也随着时间的变化也不能持续依赖。

这样就引发了一个问题：如果一些Pods提供了一些功能供其它的Pod使用，在kubernete集群中要如何实现让这些使用者能够持续的追踪到这些Pod提供的服务？所以服务的概念就产生了。

# 定义Service

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


1. 首先我们可以通过 service 的虚拟 IP 去访问，比如说刚创建的 my-service 这个服务，通过 kubectl get svc 或者 kubectl discribe service 都可以看到它的虚拟 IP 地址是 172.29.3.27，端口是 80，然后就可以通过这个虚拟 IP 及端口在 pod 里面直接访问到这个 service 的地址。

2. 第二种方式直接访问服务名，依靠 DNS 解析，就是同一个 namespace 里 pod 可以直接通过 service 的名字去访问到刚才所声明的这个 service。不同的 namespace 里面，我们可以通过 service 名字加“.”，然后加 service 所在的哪个 namespace 去访问这个 service，例如我们直接用 curl 去访问，就是 my-service:80 就可以访问到这个 service。

3. 第三种是通过环境变量访问，在同一个 namespace 里的 pod 启动时，K8s 会把 service 的一些 IP 地址、端口，以及一些简单的配置，通过环境变量的方式放到 K8s 的 pod 里面。在 K8s pod 的容器启动之后，通过读取系统的环境变量比读取到 namespace 里面其他 service 配置的一个地址，或者是它的端口号等等。比如在集群的某一个 pod 里面，可以直接通过 curl $ 取到一个环境变量的值，比如取到 MY_SERVICE_SERVICE_HOST 就是它的一个 IP 地址，MY_SERVICE 就是刚才我们声明的 MY_SERVICE，SERVICE_PORT 就是它的端口号，这样也可以请求到集群里面的 MY_SERVICE 这个 service。


## 实验

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

## 集群外访问Service

前面介绍的都是在集群里面 node 或者 pod 去访问 service，service 怎么去向外暴露呢？怎么把应用实际暴露给公网去访问呢？这里 service 也有两种类型去解决这个问题，一个是NodePort，一个是LoadBalancer。

1. NodePort的方式就是在集群的 node 上面（即集群的节点的宿主机上面）去暴露节点上的一个端口，这样相当于在节点的一个端口上面访问到之后就会再去做一层转发，转发到虚拟的 IP 地址上面，就是刚刚宿主机上面 service 虚拟 IP 地址。


2. LoadBalancer类型就是在 NodePort 上面又做了一层转换，刚才所说的 NodePort其实是集群里面每个节点上面一个端口，LoadBalancer是在所有的节点前又挂一个负载均衡。比如在阿里云上挂一个 SLB，这个负载均衡会提供一个统一的入口，并把所有它接触到的流量负载均衡到每一个集群节点的 node pod 上面去。然后 node pod 再转化成 ClusterIP，去访问到实际的 pod 上面。


## 实验

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

## Ingress

将集群的服务提供给外部访问的另外一种方式是Ingress，Ingress是授权入站连接到达集群服务的规则集合，有点类似nginx的功能。你可以给Ingress配置提供外部可访问的URL、负载均衡、SSL、基于名称的虚拟主机等。用户通过POST Ingress资源到API server的方式来请求ingress。 Ingress controller负责实现Ingress，通常使用负载平衡器，它还可以配置边界路由和其他前端，这有助于以HA方式处理流量。

## 实验

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
