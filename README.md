# Install

1. 安装的minikube版本是1.7.3
2. 安装的VirtualBox版本是Oracle VM VirtualBox Manager 5.2.34_Ubuntu
3. 安装的kubectl客户端版本是v1.18.2

启动minikube使用下面的命令：

```shell
minikube start --image-mirror-country cn --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.7.3.iso --registry-mirror=https://registry.docker-cn.com
```
测试使用的平台是Ubuntu18.04

minikube start --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers' --registry-mirror=https://registry.docker-cn.com
