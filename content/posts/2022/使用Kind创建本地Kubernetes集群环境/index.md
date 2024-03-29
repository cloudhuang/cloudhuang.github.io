---
title: 使用Kind创建本地Kubernetes集群环境
date: 2022-07-25
lastmod: 2022-07-25
tags: [publish, Kubernetes]
---

## Kind 介绍
[Kind][kind] (Kubernetes IN Docker) 让你能够快速在本地计算机上运行 Kubernetes集群。 kind 要求你安装并配置好 Docker。

下面是使用kind创建一个多worker的Kubernetes集群，将下面的内容保存为`kind-config.yaml`:

```
kind: Cluster 
apiVersion: kind.x-k8s.io/v1alpha4 
nodes: 
- role: control-plane 
- role: worker 
- role: worker 
- role: worker 
networking: 
  disableDefaultCNI: false
```

这里不使用默认的CNI，然后通过`kind`的命令行程序创建Kubernetes集群:
```
kind create cluster --name k8s --config kind-config.yaml
```

[kind][kind] 会自动的pull需要的docker镜像(镜像比较大)，并创建由一个控制节点，3个工作节点组成的Kubernetes集群。

```shell
$ kind create cluster --name k8s --config kind-config.yaml
Creating cluster "k8s" ...
 • Ensuring node image (kindest/node:v1.20.2) �  ...
 ✓ Ensuring node image (kindest/node:v1.20.2) �
 • Preparing nodes � � � �   ...
 ✓ Preparing nodes � � � �
 • Writing configuration �  ...
 ✓ Writing configuration �
 • Starting control-plane �️  ...
 ✓ Starting control-plane �️
 • Installing CNI �  ...
 ✓ Installing CNI �
 • Installing StorageClass �  ...
 ✓ Installing StorageClass �
 • Joining worker nodes �  ...
 ✓ Joining worker nodes �
Set kubectl context to "kind-k8s"
You can now use your cluster with:

kubectl cluster-info --context kind-k8s

Have a nice day! �               
```

完毕之后，可以通过`kubectl`命令来查看集群的信息:

```shell
$ kubectl cluster-info --context kind-k8s
Kubernetes control plane is running at https://127.0.0.1:51472
KubeDNS is running at https://127.0.0.1:51472/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes
NAME                STATUS     ROLES                  AGE     VERSION
k8s-control-plane   NotReady   control-plane,master   2m20s   v1.20.2
k8s-worker          NotReady   <none>                 102s    v1.20.2
k8s-worker2         NotReady   <none>                 103s    v1.20.2
k8s-worker3         NotReady   <none>                 102s    v1.20.2
```

由于还么有安装CNI插件，所以现在集群的状态是NotReady的。

这里可以通过`docker ps`命令来检查一个运行的Docker容器:

```shell
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS         PORTS                       NAMES
7f9fb033c531   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes   127.0.0.1:55289->6443/tcp   bigdata-control-plane
4ac75ba79030   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes                               bigdata-worker2
5a27977d042a   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes                               bigdata-worker
8589f4d65b40   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes                               bigdata-worker3
```

然后需要将Master的kubeconfig备份出来，通过kubeconfig配置拷贝到相应管理节点，就可以通过`kubectl`命令来管理Kubernetes集群了。

```
$ kind get kubeconfig --name=k8s > kubeconfig
$ cp kubeconfig ~/.kube/config
```

## Helm3

Helm 是Kubernetes的包管理器，可以方便的查找、分享和使用软件构建 Kubernetes 应用包。
基于Helm，可以大大的优化基础运维建设及业务应用运维： 

- 基础设施，更方便地部署与升级基础设施，如 gitlab，prometheus，grafana，ES 等
- 业务应用，更方便地部署，管理与升级公司内部应用，为公司内部的项目配置 Chart，使用 helm 结合 CI，在 k8s 中部署应用如一行命令般简单方便

### 安装Helm3
这里参考官方文档 [安装 helm](https://helm.sh/docs/intro/install/)

## MetalLB

在Kubernetes中，Kubernets为一组提供相同功能的Pods提供了`Service`抽象，借助Service，应用可以方便的实现服务发现与负载均衡，并实现应用的零宕机升级。

Service有三种类型：

- ClusterIP：默认类型，自动分配一个仅cluster内部可以访问的虚拟IP
- NodePort：在ClusterIP基础上为Service在每台机器上绑定一个端口，这样就可以通过`<NodeIP>:NodePort`来访问该服务
- LoadBalancer：在NodePort的基础上，借助cloud provider创建一个外部的负载均衡器，并将请求转发到`<NodeIP>:NodePort`

然而kubernetes本身并没有实现LoadBalancer，所以Kubernetes的LoadBalancer类型的Service依赖于外部的云提供的Load Balancer。但是当我们把K8S部署在裸机上面，或者是测试环境时，仍然需要简单的LoadBalancer来验证工作，这个时候，开源的MetalLB就是一个不错的选择。

### 安装 MetalLB 

```shell
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
```

下面需要为MetalLB创建一个IP的区段，用来给Service分配External IP,  这里采用了Kind的方式来创建Kubernetes集群，所以需要查看Kind创建的docker network的可用地址段，Metallb 将会在其中选取地址分配给服务:

```shell
$ docker network inspect -f '{{.IPAM.Config}}' kind
[{172.24.0.0/16  172.24.0.1 map[]} {fc00:f853:ccd:e793::/64   map[]}]
```

通过上面的命令，可以看到我们的IP地址池为`172.22.26.210/20`，所以我们可以为MetalLB设该地址池里的某个IP段:

```shell
# ./helm/metallb/values.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.24.0.200 - 172.24.0.250
```

将上面的内容保存到文件，然后执行

```shell
$ kubectl apply -f ./values.yaml
```

然后创建一个类型为 `LoadBalancer` 的Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 4
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
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```
当Pod成功运行之后，可以通过`kubectl get svc`命令查看分配的 `EXTERNAL-IP` 地址:
```shell
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP        25m
nginx        LoadBalancer   10.96.133.110   172.24.0.200   80:30905/TCP   22s
```

### 通过Helm安装 MetalLB

下面通过helm来安装MetalLB

```shell
$ helm repo add metallb https://metallb.github.io/metallb
$ helm install metallb metallb/metallb
```

用户需要在配置中提供一个地址池，Metallb 将会在其中选取地址分配给服务:

```shell
$ docker network inspect -f '{{.IPAM.Config}}' kind
[{172.24.0.0/16  172.24.0.1 map[]} {fc00:f853:ccd:e793::/64   map[]}]
```

通过上面的命令，可以看到我们的IP地址池为`172.22.26.210/20`，所以我们可以为MetalLB设该地址池里的某个IP段:

```shell
# ./helm/metallb/values.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.24.0.200 - 172.24.0.250
```

将上面文件保存到`./helm/metallb/values.yaml`文件中，然后通过helm命令安装:

```shell
$ helm install metallb metallb/metallb -f ./helm/metallb/values.yaml -n metallb-system --create-namespace
```

安装成功之后，可以通过 `kubectl` 查看运行状态

```shell
$ kubectl get all -n metallb-system
NAME                                      READY   STATUS    RESTARTS   AGE
pod/metallb-controller-5488f4c94b-8zmpn   1/1     Running   0          39s
pod/metallb-speaker-7w4tg                 1/1     Running   0          39s
pod/metallb-speaker-jbjj9                 1/1     Running   0          39s
pod/metallb-speaker-knnvj                 1/1     Running   0          39s
pod/metallb-speaker-z76gd                 1/1     Running   0          39s

NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/metallb-speaker   4         4         4       4            4           kubernetes.io/os=linux   39s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/metallb-controller   1/1     1            1           39s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/metallb-controller-5488f4c94b   1         1         1       39s
```

## 参考

- [k8s-install-helm](https://docs.cilium.io/en/v1.10/gettingstarted/k8s-install-helm/)

[kind]: https://kind.sigs.k8s.io/
[flannel]: https://github.com/coreos/flannel
[metallb]: https://metallb.universe.tf/
[cilium]: https://cilium.io/

