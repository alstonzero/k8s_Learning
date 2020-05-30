## Kubernetes入门实战

### 1、部署一个应用

Kubernetes 部署

- 在 k8s 上进行部署前，首先需要了解一个基本概念 **Deployment**
- **Deployment** （ **部署**）。在k8s中，通过发布 Deployment，可以创建应用程序 (docker image) 的实例 (docker container)，这个实例会被包含在称为 **Pod** 的概念中，**Pod** 是 k8s 中最小可管理单元。
- 在 k8s 集群中发布 Deployment 后，Deployment 将指示 k8s 如何创建和更新应用程序的实例，master 节点将应用程序实例调度到集群中的具体的节点上。
- 创建应用程序实例后，Kubernetes Deployment Controller 会持续监控这些实例。如果运行实例的 worker 节点关机或被删除，则 Kubernetes Deployment Controller 将在群集中资源最优的另一个 worker 节点上重新创建一个新的实例。**这提供了一种自我修复机制来解决机器故障或维护问题。**
- 在容器编排之前的时代，各种安装脚本通常用于启动应用程序，但是不能够使应用程序从机器故障中恢复。通过创建应用程序实例并确保它们在集群节点中的运行实例个数，Kubernetes Deployment 提供了一种完全不同的方式来管理应用程序。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524131640264.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

Deployment、Pod和Container。

Deployment 处于 master 节点上，通过发布 Deployment，master 节点会选择合适的 worker 节点创建 Container（即图中的正方体），Container 会被包含在 Pod （即蓝色圆圈）里。

#### 实战：部署 nginx Deployment

**1、创建 YAML 文件**

创建文件 nginx-deployment.yaml，内容如下：

```shell
apiVersion: apps/v1	#与k8s集群版本有关，使用 kubectl api-versions 即可查看当前集群支持的版本
kind: Deployment	#该配置的类型，我们使用的是 Deployment
metadata:	        #译名为元数据，即 Deployment 的一些基本属性和信息
  name: nginx-deployment	#Deployment 的名称
  labels:	    #标签，可以灵活定位一个或多个资源，其中key和value均可自定义，可以定义多组，目前不需要理解
    app: nginx	#为该Deployment设置key为app，value为nginx的标签
spec:	        #这是关于该Deployment的描述，可以理解为你期待该Deployment在k8s中如何使用
  replicas: 1	#使用该Deployment创建一个应用程序实例
  selector:	    #标签选择器，与上面的标签共同作用，目前不需要理解
    matchLabels: #选择包含标签app:nginx的资源
      app: nginx
  template:	    #这是选择或创建的Pod的模板
    metadata:	#Pod的元数据
      labels:	#Pod的标签，上面的selector即选择包含标签app:nginx的Pod
        app: nginx
    spec:	    #期望Pod实现的功能（即在pod中部署）
      containers:	#生成container，与docker中的container是同一种
      - name: nginx	#container的名称
        image: nginx:1.7.9	#使用镜像nginx:1.7.9创建container，该container默认80端口可访问
```

**2、应用 YAML 文件**

```sh
kubectl apply -f nginx-deployment.yaml
```

**3、查看部署结果**

```powershell
# 查看 Deployment
kubectl get deployments

# 查看 Pod
kubectl get pods

#可分别查看到一个名为 nginx-deployment 的 Deployment 和一个名为 nginx-deployment-xxxxxxx 的 Pod
```

### 2、查看Pods/Nodes

#### Kubernetes Pods

在 第一步 中创建 Deployment 后，k8s创建了一个 **Pod（容器组）** 来放置应用程序实例（container 容器）

#### Pods概述

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524131711226.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

**Pod 容器组** 是一个k8s中一个抽象的概念，用于存放一组 container（可包含一个或多个 container 容器，即图上正方体)，以及这些 container （容器）的一些共享资源。这些资源包括：

- 共享存储，称为卷(Volumes)，即图上紫色圆柱
- 网络，每个 Pod（容器组）在集群中有个唯一的 IP，pod（容器组）中的 container（容器）共享该IP地址
- container（容器）的基本信息，例如容器的镜像版本，对外暴露的端口等

> 例如，Pod可能既包含带有Node.js应用程序的 container 容器，也包含另一个非 Node.js 的 container 容器，用于提供 Node.js webserver 要发布的数据。Pod中的容器共享 IP 地址和端口空间（同一 Pod 中的不同 container 端口不能相互冲突），始终位于同一位置并共同调度，并在同一节点上的共享上下文中运行。（同一个Pod内的容器可以使用 localhost + 端口号互相访问）

**Pod（容器组）是 k8s 集群上的最基本的单元**。当我们在 k8s 上创建 Deployment 时，会在集群上创建包含容器的 Pod (而不是直接创建容器)。每个Pod都与运行它的 worker 节点（Node）绑定，并保持在那里直到终止或被删除。如果节点（Node）发生故障，则会在群集中的其他可用节点（Node）上运行相同的 Pod（从同样的镜像创建 Container，使用同样的配置，IP 地址不同，Pod 名字不同）。

> TIP
>
> 重要：
>
> - Pod 是一组容器（可包含一个或多个应用程序容器），以及共享存储（卷 Volumes）、IP 地址和有关如何运行容器的信息。
> - 如果多个容器紧密耦合并且需要共享磁盘等资源，则他们应该被部署在同一个Pod（容器组）中。

#### Node（节点）

下图显示一个 Node（节点）上含有4个 Pod（容器组）

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052413180868.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

Pod（容器组）总是在 **Node（节点）** 上运行。Node（节点）是 kubernetes 集群中的计算机，可以是虚拟机或物理机。每个 Node（节点）都由 master 管理。一个 Node（节点）可以有多个Pod（容器组），kubernetes master 会根据每个 Node（节点）上可用资源的情况，自动调度 Pod（容器组）到最佳的 Node（节点）上。

每个 Kubernetes Node（节点）至少运行：

- Kubelet，负责 master 节点和 worker 节点之间通信的进程；管理 Pod（容器组）和 Pod（容器组）内运行的 Container（容器）。
- 容器运行环境（如Docker）负责下载镜像、创建和运行容器等。

#### 实战：故障排除

我们使用了 kubectl 命令行界面部署了 nginx 并且查看了 Deployment 和 Pod。kubectl 还有如下四个常用命令，在我们排查问题时可以提供帮助

- **kubectl get** - 显示资源列表

```sh
# kubectl get 资源类型

#获取类型为Deployment的资源列表
kubectl get deployments

#获取类型为Pod的资源列表
kubectl get pods

#获取类型为Node的资源列表
kubectl get nodes
```

> 名称空间
>
> 在命令后增加 `-A` 或 `--all-namespaces` 可查看所有 名称空间中 的对象，使用参数 `-n` 可查看指定名称空间的对象，例如
>
> ```sh
> # 查看所有名称空间的 Deployment
> kubectl get deployments -A
> kubectl get deployments --all-namespaces
> # 查看 kube-system 名称空间的 Deployment
> kubectl get deployments -n kube-system
> ```
>
> 并非所有对象都在名称空间里
>
> 大部分的 Kubernetes 对象（例如，Pod、Service、Deployment、StatefulSet等）都必须在名称空间里。但是某些更低层级的对象，是不在任何名称空间中的，例如 nodes、persistentVolumes、storageClass 等
>
> ```sh
> 执行一下命令可查看哪些 Kubernetes 对象在名称空间里，哪些不在
> # 在名称空间里
> kubectl api-resources --namespaced=true
> 
> # 不在名称空间里
> kubectl api-resources --namespaced=false
> ```

- **kubectl describe** - 显示有关资源的详细信息

```sh
# kubectl describe 资源类型 资源名称

#查看名称为nginx-XXXXXX的Pod的信息
kubectl describe pod nginx-XXXXXX	

#查看名称为nginx的Deployment的信息
kubectl describe deployment nginx
```

- **kubectl logs** - 查看pod中的容器的打印日志（和命令docker logs 类似）

```sh
# kubectl logs Pod名称

#查看名称为nginx-pod-XXXXXXX的Pod内的容器打印的日志
#本案例中的 nginx-pod 没有输出日志，所以您看到的结果是空的
kubectl logs -f nginx-pod-XXXXXXX
```

- **kubectl exec** - 在pod中的容器环境内执行命令(和命令docker exec 类似)

```sh
# kubectl exec Pod名称 操作命令

# 在名称为nginx-pod-xxxxxx的Pod中运行bash
kubectl exec -it nginx-pod-xxxxxx /bin/bash
```

> TIP
>
> Worker节点是k8s中的工作计算机，可能是VM或物理计算机，具体取决于群集。多个Pod可以在一个节点上运行。

### 3、公布应用程序

- 了解 Kubernetes 的 Service（服务）
- 了解 Labels（标签）和 LabelSelector（标签选择器）与 Service（服务）的关系
- 在 kubernetes 集群中，通过 Service（服务）向外公布应用程序

#### Kubernetes Service（服务）概述

事实上，**Pod（容器组）有自己的 生命周期**。当 worker node（节点）故障时，节点上运行的 Pod（容器组）也会消失。然后，**Deployment 可以通过创建新的 Pod（容器组）来动态地将群集调整回原来的状态**，以使应用程序保持运行。

举个例子，假设有一个图像处理后端程序，具有 3 个运行时副本。这 3 个副本是可以替换的（无状态应用），即使 Pod（容器组）消失并被重新创建，或者副本数由 3 增加到 5，前端系统也无需关注后端副本的变化。由于 Kubernetes 集群中每个 Pod（容器组）都有一个唯一的 IP 地址（即使是同一个 Node 上的不同 Pod），我们需要一种机制，**为前端系统屏蔽后端系统的 Pod（容器组）在销毁、创建过程中所带来的 IP 地址的变化。**

Kubernetes 中的 **Service（服务）** 提供了这样的一个抽象层，它选择具备某些特征的 Pod（容器组）并为它们定义一个访问方式。**Service（服务）使 Pod（容器组）之间的相互依赖解耦**（原本从一个 Pod 中访问另外一个 Pod，需要知道对方的 IP 地址）。一个 Service（服务）选定哪些 **Pod（容器组）** 通常由 **LabelSelector(标签选择器)** 来决定。

在创建Service的时候，通过设置配置文件中的 spec.type 字段的值，可以以不同方式向外部暴露应用程序：

- **ClusterIP**（默认）

  在群集中的内部IP上公布服务，这种方式的 Service（服务）只在集群内部可以访问到

- **NodePort**

  使用 NAT 在集群中每个的同一端口上公布服务。这种方式下，可以通过访问集群中任意节点+端口号的方式访问服务 `:`。此时 ClusterIP 的访问方式仍然可用。

- **LoadBalancer**

  在云环境中（需要云供应商可以支持）创建一个集群外部的负载均衡器，并为使用该负载均衡器的 IP 地址作为服务的访问地址。此时 ClusterIP 和 NodePort 的访问方式仍然可用。

> TIP
>
> Service是一个抽象层，它通过 LabelSelector 选择了一组 Pod（容器组），把这些 Pod 的指定端口公布到到集群外部，并支持负载均衡和服务发现。
>
> - 公布 Pod 的端口以使其可访问
> - 在多个 Pod 间实现负载均衡
> - 使用 Label 和 LabelSelecto

#### 服务和标签

下图中有两个服务Service A(黄色虚线)和Service B(蓝色虚线) Service A 将请求转发到 IP 为 10.10.10.1 的Pod上， Service B 将请求转发到 IP 为 10.10.10.2、10.10.10.3、10.10.10.4 的Pod上。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524131904680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

Service 将外部请求路由到一组 Pod 中，它提供了一个抽象层，使得 Kubernetes 可以在不影响服务调用者的情况下，动态调度容器组（在容器组失效后重新创建容器组，增加或者减少同一个 Deployment 对应容器组的数量等）。

Service使用 Labels、LabelSelector(标签和选择器) 匹配一组 Pod。Labels（标签）是附加到 Kubernetes 对象的键/值对，其用途有多种：

- 将 Kubernetes 对象（Node、Deployment、Pod、Service等）指派用于开发环境、测试环境或生产环境
- 嵌入版本标签，使用标签区别不同应用软件版本
- 使用标签对 Kubernetes 对象进行分类

下图体现了 Labels（标签）和 LabelSelector（标签选择器）之间的关联关系

- Deployment B 含有 LabelSelector 为 app=B 通过此方式声明含有 app=B 标签的 Pod 与之关联
- 通过 Deployment B 创建的 Pod 包含标签为 app=B
- Service B 通过标签选择器 app=B 选择可以路由的 Pod

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524132058517.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

Labels（标签）可以在创建 Kubernetes 对象时附加上去，也可以在创建之后再附加上去。任何时候都可以修改一个 Kubernetes 对象的 Labels（标签）

#### 实战：为您的 nginx Deployment 创建一个 Service

创建nginx的Deployment中定义了Labels，如下：

```sh
metadata:	#译名为元数据，即Deployment的一些基本属性和信息
  name: nginx-deployment	#Deployment的名称
  labels:	#标签，可以灵活定位一个或多个资源，其中key和value均可自定义，可以定义多组
    app: nginx	#为该Deployment设置key为app，value为nginx的标签
```

**创建文件 nginx-service.yaml**

```sh
vim nginx-service.yaml
```

**文件内容如下：**

```sh
apiVersion: v1
kind: Service
metadata:
  name: nginx-service	#Service 的名称
  labels:     	#Service 自己的标签
    app: nginx	#为该 Service 设置 key 为 app，value 为 nginx 的标签
spec:	    #这是关于该 Service 的定义，描述了 Service 如何选择 Pod，如何被访问
  selector:	    #标签选择器
    app: nginx	#选择包含标签 app:nginx 的 Pod
  ports:
  - name: nginx-port	#端口的名字
    protocol: TCP	    #协议类型 TCP/UDP
    port: 80	        #集群内的其他容器组可通过 80 端口访问 Service
    nodePort: 32600   #通过任意节点的 32600 端口访问 Service
    targetPort: 80	#将请求转发到匹配 Pod 的 80 端口
  type: NodePort	#Serive的类型，ClusterIP/NodePort/LoaderBalancer
```

**执行命令**

```sh
kubectl apply -f nginx-service.yaml
```

**检查结果**

```sh
kubectl get services -o wide
```

**访问服务**

```sh
curl <任意节点的 IP>:32600
```

### 4、伸缩应用程序

#### Scaling（伸缩）应用程序

在之前的文章中，我们创建了一个 Deployment，然后通过 服务 提供访问 Pod 的方式。我们发布的 Deployment 只创建了一个 Pod 来运行我们的应用程序。当流量增加时，我们需要对应用程序进行伸缩操作以满足系统性能需求。

**伸缩** 的实现可以通过更改 nginx-deployment.yaml 文件中部署的 replicas（副本数）来完成

```sh
spec:
  replicas: 2    #使用该Deployment创建两个应用程序实例
```

#### Scaling（伸缩）概述

下图中，Service A 只将访问流量转发到 IP 为 10.0.0.5 的Pod上

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524132134276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

修改了 Deployment 的 replicas 为 4 后，Kubernetes 又为该 Deployment 创建了 3 新的 Pod，这 4 个 Pod 有相同的标签。因此Service A通过标签选择器与新的 Pod建立了对应关系，将访问流量通过负载均衡在 4 个 Pod 之间进行转发。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524132151516.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

> TIP
>
> 通过更改部署中的 replicas（副本数）来完成扩展

#### 实战：将 nginx Deployment 扩容到 4 个副本

**修改 nginx-deployment.yaml 文件**

```sh
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
        image: nginx:1.7.9
        ports:
        - containerPort: 80
  
  
#执行命令
kubectl apply -f nginx-deployment.yaml
#查看结果
watch kubectl get pods -o wide
```

### 5、执行滚动更新

#### 更新应用程序

用户期望应用程序始终可用，为此开发者/运维者在更新应用程序时要分多次完成。在 Kubernetes 中，这是通过 Rolling Update 滚动更新完成的。**Rolling Update滚动更新** 通过使用新版本的 Pod 逐步替代旧版本的 Pod 来实现 Deployment 的更新，从而实现零停机。新的 Pod 将在具有可用资源的 Node（节点）上进行调度。

> Kubernetes 更新多副本的 Deployment 的版本时，会逐步的创建新版本的 Pod，逐步的停止旧版本的 Pod，以便使应用一直处于可用状态。这个过程中，Service 能够监视 Pod 的状态，将流量始终转发到可用的 Pod 上。

在上一个模块中，我们学习了将应用程序 Scale Up（扩容）为多个实例，这是执行更新而不影响应用程序可用性的前提（如果只有 1 个实例那还玩啥）。默认情况下，**Rolling Update 滚动更新** 过程中，Kubernetes 逐个使用新版本 Pod 替换旧版本 Pod（最大不可用 Pod 数为 1、最大新建 Pod 数也为 1）。这两个参数可以配置为数字或百分比。在Kubernetes 中，更新是版本化的，任何部署更新都可以恢复为以前的（稳定）版本。

#### 滚动更新概述

1、 原本 Service A 将流量负载均衡到 4 个旧版本的 Pod （当中的容器为 绿色）上

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052413222055.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

2、更新完 Deployment 部署文件中的镜像版本后，master 节点选择了一个 worker 节点，并根据新的镜像版本创建 Pod（紫色容器）。新 Pod 拥有唯一的新的 IP。同时，master 节点选择一个旧版本的 Pod 将其移除。

此时，Service A 将新 Pod 纳入到负载均衡中，将旧Pod移除

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524132237858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

3、同步骤2，再创建一个新的 Pod 替换一个原有的 Pod

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524132253541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)
4、 如此 Rolling Update 滚动更新，直到所有旧版本 Pod 均移除，新版本 Pod 也达到 Deployment 部署文件中定义的副本数，则滚动更新完成

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524132310480.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NjAzNjE2NQ==,size_16,color_FFFFFF,t_70)

滚动更新允许以下操作：

- 将应用程序从准上线环境升级到生产环境（通过更新容器镜像）
- 回滚到以前的版本
- 持续集成和持续交付应用程序，无需停机

#### 实战：更新 nginx Deployment

**修改 nginx-deployment.yaml 文件**

修改文件中 image 镜像的标签，如下所示

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
        image: nginx:1.8   #使用镜像nginx:1.8替换原来的nginx:1.7.9
        ports:
        - containerPort: 80
kubectl apply -f nginx-deployment.yaml

#查看过程及结果
#执行命令，可观察到 pod 逐个被替换的过程。
watch kubectl get pods -l app=nginx
```

### 6、也可以都使用命令来完成

```sh
#1、部署一个 nginx
kubectl create deployment mynginx --image=nginx 

#2、暴露 nginx 访问 
kubectl expose deployment mynginx --port=80 --type=NodePort
随便一歌机器的ip+端口都可以

#3、扩容
kubectl scale --replicas=3 deployment/nginx

#4、更多
https://kubernetes.io/docs/reference/kubectl/overview/
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands
```

### 7、搭建可视化界面

```shell
#下载dashboard.yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

#暴露端口，修改位置
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  type: NodePort
  selector:
    k8s-app: kubernetes-dashboard


#创建集群用户,按照如下说明
https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

#访问测试
每次访问都需要令牌
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
eyJhbGciOiJSUzI1NiIsImtpZCI6InVrOWJVTXpoY1ZraThKSExJYl96eFFTMi1zWTE1TFhMaW15NG9MNzAyVmsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWY5eDg2Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI3NDk2ODc4ZS02YWQzLTQ1NmYtYWQwZS00YjRkMGUwMGRhYTQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.O6rtH_70s-UilYBpKEmRObBBayBM5i4mRhnsPUhONZ7Nc5O-bsoF6JhgP5iZLEXRU5_QA8wDm4V1831JBR0kPjjMSRrbzofLHUh61PPgHaq0kMe7LAyfr0njnnu24UexYllaPCV5BuDVRsyV1zwISyJNEkbiUmJ73-Rh0vOkYxE3bB0mDa6pniIkr9GiiarDthmm5tBvGx7EGEapSS9xl5RXMsPdRSIMGb7aFepst5OXh76CICLl_4JBiqCDlcjKIsks2ghOwyAd_2s5m1h5lvVSKIGOEqQQc8FN1uA_zuC2anBSk-XJCWGmzx6vmxMhwOQJN-hQSJh_JgS66UabpA

```