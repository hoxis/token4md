---
title: kubernetes笔记
tags:  kubernetes
---

# Kubernetes命令行用法

## 启动
```shell
[root@centos7 ~]# systemctl start etcd
[root@centos7 ~]# systemctl start docker
[root@centos7 ~]# systemctl start kube-apiserver
[root@centos7 ~]# systemctl start kube-controller-manager
[root@centos7 ~]# systemctl start kube-scheduler
[root@centos7 ~]# systemctl start kubelet
[root@centos7 ~]# systemctl start kube-proxy
```

## kubectl
语法：`$ kubectl [command] [TYPE] [NAME] [flags]`

1. `command`：子命令，用于操作资源对象，例如create、delete、get、 describe等；
2. `TYPE`：资源对象类型，区分大小写
3. `NAME`：对象名称，区分大小写，若不指定，则返回所有对象的列表；
4. `flags`：可选参数；

![kubectl可操作的资源对象类型][1]

# 示例
Guestbook留言板系统将通过`Pod`、`RC`、`Service`等资源对象搭建完成。其系统架构是一个基于PHP+Redis的分布式web应用，前端PHP Web通过访问后端的Redis来完成用户留言的查询和添加等功能，同时Redis以Master+Slave的模式部署，实现数据的读写分离能力。

![Guestbook留言板系统系统架构][2]

Web层是一个基于PHP页面的Apache服务，启动3个实例组成集群，为客户端访问提供负载均衡。Redis Master启动一个实例用于写操作，Slave启动两个实例用于读操作。

本例用的3个镜像：
- `redis-master`：用于前端web进行写操作；
- `guestbook-redis-slave`：用于前端进行读redis服务，并与Master节点数据保持同步；
- `guestbook-php-fronted`：前端服务，展示留言内容，提供输入框；

## 创建redis-master RC和Service
先定义Service，然后定义一个RC来创建和控制相关联的Pod，或者先定义RC来创建Pod，然后定义相关联的Service。这里使用后面的方式。

首先为redis-master创建一个名为redis-master的RC定义文件redis-master-controller.yaml。
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master
  labels:
    name: redis-master
spec:
  replicas: 1
  selector:
    name: redis-master
  template:
    metadata:
      labels:
        name: redis-master
    spec:
      containers:
      - name: master
        image: kubeguide/redis-master
        ports:
        - containerPort: 6379
```
其中，`kind`字段的值是`ReplicationController`，标识这是一个RC；`spec.selecctor`是RC的Pod选择权，即监控和管理拥有这些标签的Pod实例，确保当前集群上始终有且仅有replicas个Pod实例在运行。注意，这里的labels必需匹配RC的spec.selector，否则就是为他人做嫁衣。

执行`kubectl create -f redis-master-controller.yaml`。

Kubernetes会根据RC的定义自动创建Pod实例。由于Pod的调度和创建需要花费一定的时间，比如需要确定调度到哪个节点上，以及下载相关镜像等。

下面创建一个与之关联的Service：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-master
  labels:
    name: redis-master
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    name: redis-master
```
其中，metadata.name是Service的服务名，spec.selector确定了选择哪些Pod。port属性定义的是Service的虚拟机端口号，targetPort属性指定后端Pod内容器应用监听的端口号。

创建service：`kubectl create -f redis-master-service.yaml`

使用`kubectl get services`查看services状态，可以看到redis-master服务被分配了一个虚拟IP地址10.254.213.212，随后Kubernetes机器中其他新建的Pod就可以通过该IP+端口6379来访问这个服务了。本例中的redis-slave和fronted两组Pod都会以10.254.213.212:6379的方式访问redis-master服务。

Kubernetes使用了环境变量在每个Pod的容器里都增加一组Service相关的环境变量，用来记录从服务名到虚拟IP地址的映射关系。已redis-master服务为例，在容器的环境变量里可以增加如下两个记录：
```shell
REDIS_MASTER_SERVICE_HOST=10.254.213.212
REDIS_MASTER_SERVICE_PORT=6379
```

于是，在redis-slave和fronted等Pod中就可以通过环境变量来获取redis-master服务的IP地址和端口号。这样就完成了对服务地址的查询功能。

## 创建redis-slave RC和Service

首先创建一个redis-slave的RC定义文件`redis-slave-controller.yaml`：
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-slave
  labels:
    name: redis-slave
spec:
  replicas: 2
  selector:
    name: redis-slave
  template:
    metadata:
      labels:
        name: redis-slave
    spec:
      containers:
      - name: slave
        image: kubeguide/guestbook-redis-slave
        env:
        - name: GET_HOSTS_FROM
          value: env
        ports:
        - containerPort: 6379
```
其中配置了一个环境变量`GET_HOSTS_FROM=env`，意思就是从环境变量中获取redis-master服务的IP地址。

redis-slave镜像中的启动脚本`run.sh`内容为：
```shell
if [[ $(GET_HOSTS_FROM:-dns) == "env" ]]; then
  redis-server --slaveof ${REDIS_MASTER_SERVICE_HOST} 6379
else
  redis-server --slaveof redis-master 6379
fi
```
在创建redis-slave Pod时，系统将自动在容器内部生成之前已经创建好的redis-master service相关的环境变量，所以redis-slave应用可以直接使用环境变量`REDIS_MASTER_SERVICE_HOST`来后去redis-master服务的IP地址。

如果在容器配置部分不设置该env，则将使用redis-master服务的名称“redis-master”来访问，这将使用DNS方式的服务发现，需要预先启动Kubernetes集群的`skydns`服务。

同样的创建slave服务：
```yaml
kind: Service
metadata:
  name: redis-slave
  labels:
    name: redis-slave
spec:
  ports:
  - port: 6379
  selector:
    name: redis-slave
```

## 创建frontend RC和Service

frontend-controller.yaml配置文件：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: frontend
  labels:
    name: frontend
spec:
  replicas: 3
  selector:
    name: frontend
  template:
    metadata:
      labels:
        name: frontend
    spec:
      containers:
      - name: frontend
        image: kubeguide/guestbook-php-frontend
        env:
        - name: GET_HOSTS_FROM
          value: env
        ports:
        - containerPort: 80
```

服务定义文件frontend-service.yaml：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    name: frontend
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30001
  selector:
    name: frontend
```
这里的关键是设置`type: NodePort`并指定一个NodePort的值，表示使用Node上的物理端口提供对外访问的能力。

# 深入了解Pod

## Pod定义详解

完整内容：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: string
  namespace: string
  labels:
    - name: string
  annotations:
    - name: string
spec:
  containers:
  - name:string
    image: string
    imagePullPolicy: [Always|Nerver|IfNotPresent]
    command: [string]
    args: [string]
    workingDir:string
    volumeMounts:
    - name: string
      mountPath: string
      readOnly: boolean
    ports:
    - name: string
      containerPort: int
      hostPort: int
      protocol: string
    env:
    - name: string
      value: string
    resources:
      limits:
        cpu: string
        memory: string
      requests:
        cpu: string
        memory: string
    livenessProbe:
      exec:
        command: [string]
      httpGet:
        path: string
        port: number
        host: string
        scheme: string
        httpHeaders:
        - name: string
          value: string
      tcpSocket:
        port: number
      initialDelaySeconds: 0
      timeoutSeconds: 0
      periodSeconds: 0
      successThreshold: 0
      failureThreshold: 0
    securityContext:
      privileged: false
  restartPolicy: [Always|Never|OnFailure]
  nodeSelector: object
  imagePullSecrets:
  - name: string
  hostNetwork: false
  volumes:
  - name: string
    emptyDir: {}
    hostPath:
      path: string
    secret:
      secretName: string
      items:
      - key: string
        path: string
    configMap:
      name: string
      items:
      - key: string
        path: string
```

![Pod定义文件属性说明1][3]

![Pod定义文件属性说明2][4]

![Pod定义文件属性说明3][5]

## Pod的基本用法
在Kubernetes系统中对长时间运行容器的要求是：其主程序需要一直在前台执行。

Pod可以由1个或者多个容器组合而成。








  [1]: https://www.github.com/hoxis/token4md/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/snipaste20170901_112248.png
  [2]: https://www.github.com/hoxis/token4md/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1504243380177.jpg
  [3]: https://www.github.com/hoxis/token4md/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1504516071055.jpg
  [4]: https://www.github.com/hoxis/token4md/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1504516407438.jpg
  [5]: https://www.github.com/hoxis/token4md/raw/master/%E5%B0%8F%E4%B9%A6%E5%8C%A0/1504516523310.jpg