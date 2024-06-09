[kubectl 备忘单 | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/kubectl/cheatsheet/)

## 01 基本概念

![image-20221120164652802](attachments/image-20221120164652802.png)

- Pod：K8s最小部署单元，一组容器的集合 

- Deployment：最常见的控制器，用于更高级别部署和管理Pod 
- Service：为一组Pod提供负载均衡，对外提供统一访问入口 
- Label ：标签，附加到某个资源上，用于关联对象、查询和筛选 
- Namespaces ：命名空间，将对象逻辑上隔离，也利于权限控制
- Context 上下文，通过 kubeconfig 文件中的 context 元素，使用简便的名称来对访问参数进行分组。每个上下文都有三个参数：cluster、namespace 和 user。默认情况下，kubectl 命令行工具使用 当前上下文 中的参数与集群进行通信。

## 02 常用命令

### 2.1 自动补全工具

```bash
# 安装依赖包
yum install bash-completion
# 刷新并配置开机启动
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~./bashrc
```

### 2.2 kubeconfig配置文件

![image-20221120171438873](attachments/image-20221120171438873.png)

可以通过以下命令查看切换上下文

```bash
# 查看当前所在的context
kubectl config current-context
output: kubernetes-admin@kubernetes

# 使用指定的context
kubectl config use-context kubernetes-admin@kubernetes  
output: Switched to context "kubernetes-admin@kubernetes".
```

2.2 命令集合

```bash
kubectl label nodes k8s-node1 disk=stat # 创建标签
kubectl label nodes k8s-node1 disk- # 删除标签
kubectl get nodes --show-labels # 查看标签

# 创建各种资源，例如deployment、service、pod等
kubectl crate 
kubectl run

# 为deployment创建service 
kubectl expose 
kubectl expose deployment web --port=80 --taret-port=80 --type=NodePort
--port # 代表service端口
--target-port # 代表容器中服务端口，例如nginx默认为80，tomcat默认为8080
--type # 代表service类型，NodePort就是让集群外部浏览器访问的

# 查看各种资源
kubectl get

# 删除各种资源，例如deployment、service、pod等
kubectl delete 资源名

# 查看节点详情
kubectl describe node k8s-master

# 查看pod详情
kubectl describe pod pod-name

# 查看pod网络状态详细信息和Service暴露的端口

# 查看关联后端的节点
kubectl get endpoints

# 查看pod更多信息
kubectl get pods -o wide

# 导出yaml配置
kubectl get pods pod-name -o yaml

# 查看集群资源利用率
kubectl top pod / node

# 查看容器日志
kubectl logs name -n namesapce-name

# 查看API版本
kubectl api-versions

# 查看API资源
kubectl api-resoureces

# 查看service关联的pod
kubectl get endpoint
```

![image-20221120171941425](attachments/image-20221120171941425.png)

![image-20221120172013428](attachments/image-20221120172013428.png)

2.3 常见缩写

命名空间

- 命令行 加 -n 指定命名空间
- yaml资源元数据里指定namespace字段

svc serviced的缩写

## 03 应用部署

### 3.1 部署 nginx 案例

> 集群外节点访问
>
> 外部流量访问K8s的一种方式，即**nodeIP:nodePort**，是提供给外部流量访问K8s集群资源的一种方式。
>
> 例如需要暴露服务的端口给外界访问的话，需要首先实现Kubenetes 里 Pod 和 Service 绑定

```bash
# 1 创建一个叫nginx-deployment的deployment
kubectl create deploy nginx-deployment --image=nginx
# 2 然后创建一个同名的 service
kubectl create service nodeport nginx-deployment --tcp 80:80
# 3 查看服务
kubectl get svc
# 4 得到名称为 nginx-deployment 对外暴露的端口号：32617：
# 5 通过虚拟机 ip 加端口号 32617，实现集群外访问
```

![image-20221120160718111](attachments/image-20221120160718111.png)

![image-20221120160602072](attachments/image-20221120160602072.png)

### 3.2 应用部署流程

![image-20221122230724836](attachments/image-20221122230724836.png)

其流程与3.1中nginx的部署类似，对于单个服务我们可以通过以上命令操作，实际生产中应使用yaml文件编排部署应用，

## 04 服务编排 YAML

### 4.1  生成YAML文件

```yaml
# 生成 deployment.yaml
kubectl create deployment nginx --image=nginx:1.16 -o yaml --dry-run=client > my-deploy.yaml

# 生成 pod.yaml
kubectl run --image=nginx my-pod -o yaml --dry-run=client >my-pod.yaml

# 生成 service my-deploy-service.yaml
kubectl expose deployment my-deploy --name=front-end-svc --port=80 --target-port=80 \
--type=NodePort --dry-run=client > my-deploy-service.yaml

# --dry-run 运行但不实际执行，参数 1 client 本地kubectl内置验证  参数 2 提交到apiserver验证
kubectl get deployment nginx -o yaml > my-deploy.yaml

--image # 指定模板镜像
my-deploy # 运行标签名称
--dry-run # 只测试运行,不会实际运行pod
-o yaml # 指定输出格式

# 查询字段信息
kubectl explain pods # 每一个层级的指令都会有字段信息
kubectl explain pods.spec
kubectl explain pods.spec.containers
```

### 4.2 deployment

#### 4.2.1 deployment 生命周期

```mermaid
graph LR
应用程序 --> A(部署)
A --> B(升级)
B --> C(回滚)
C --> D(下线)
```

```bash
# (1) 部署镜像
kubectl create deployment web --image=nginx:1.11.9 --replicas=3 \ 
--dry-run=client -o yaml > web.yaml
kubectl apply -f web.yaml

# (2) 应用升级
kubectl set image deployment/web nginx=nginx:1.19
kubectl edit deployment/web

# 滚动升级：kubernetes 对 pod 升级的默认策略，通过使用新版本的pod逐步更新旧版本的pod，实现零停机发布，用户无感知
# 滚动升级在kubernetes中的实现 1 个 deployment 2 个replicaSet
# 水平扩缩容（启动多实例，提高并发）
kubectl scale deployment web --replicas=10

# (3) 回滚 : 回滚是重新部署某一次部署时的状态，即当时版本的所有配置
# 查看 replicaSet
kubectl describe rs | grep revision
# 查看历史发布版本
kubectl rollout history deployment/web
# 回滚上一个版本
kubectl rollout undo deployment/web
# 回滚历史指定版本
kubectl rollout undo deployment/web -to-revision=2

# (4) 下线
kubectl delete deployment web
kubectl delete service web-service
```

#### 4.2.2 配置模板

https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/

```yaml
# 1.9.0 之前的版本使用 apps/v1beta2，可通过命令 kubectl api-versions 查看
# 之前的版本使用 apps/v1
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: deployment-name  # 设置deployment的名字
  labels:
    app: deployment-label  # 设置deployment的标签
spec: <Object>
  minReadySeconds: <integer> #设置pod准备就绪的最小秒数
  paused: <boolean> #表示部署已暂停并且deploy控制器不会处理该部署
  progressDeadlineSeconds: <integer>
  strategy: <Object> #将现有pod替换为新pod的部署策略
    rollingUpdate: <Object> #滚动更新配置参数，仅当类型为RollingUpdate
      maxSurge: <string> #滚动更新过程产生的最大pod数量，可以是个数，也可以是百分比
      maxUnavailable: <string> #
    type: <string> #部署类型，Recreate，RollingUpdate
  replicas: <integer> #pods的副本数量
  selector: <Object> #pod标签选择器，匹配pod标签，默认使用pods的标签
    matchLabels: <map[string]string> 
      key1: value1
      key2: value2
    matchExpressions: <[]Object>
      operator: <string> -required- #设定标签键与一组值的关系，In, NotIn, Exists and DoesNotExist
      key: <string> -required-
      values: <[]string>   
  revisionHistoryLimit: <integer> #设置保留的历史版本个数，默认是10
  rollbackTo: <Object> 
    revision: <integer> #设置回滚的版本，设置为0则回滚到上一个版本
  template: <Object> -required-    # pod 模板
    metadata:                      # pod 元数据
      labels:                      
        key: values 
    spec:
      containers: <[]Object> #容器配置
      - name: <string> -required- #容器名、DNS_LABEL
        image: <string> #镜像
        imagePullPolicy: <string> #镜像拉取策略，Always、Never、IfNotPresent
        ports: <[]Object>
        - name: #定义端口名
          containerPort: #容器暴露的端口
          protocol: TCP #或UDP
        volumeMounts: <[]Object>
        - name: <string> -required- #设置卷名称
          mountPath: <string> -required- #设置需要挂载容器内的路径
          readOnly: <boolean> #设置是否只读
        livenessProbe: <Object> # 存活检查，重启容器
          exec: 
            command: <[]string>
          httpGet:
            port: <string> -required-
            path: <string>
            host: <string>
            httpHeaders: <[]Object>
              name: <string> -required-
              value: <string> -required-
            scheme: <string> 
          initialDelaySeconds: <integer> #设置多少秒后开始探测
          failureThreshold: <integer> #设置连续探测多少次失败后，标记为失败，默认三次
          successThreshold: <integer> #设置失败后探测的最小连续成功次数，默认为1
          timeoutSeconds: <integer> #设置探测超时的秒数，默认1s
          periodSeconds: <integer> #设置执行探测的频率（以秒为单位），默认1s
          tcpSocket: <Object> #TCPSocket指定涉及TCP端口的操作
            port: <string> -required- #容器暴露的端口
            host: <string> #默认pod的IP
        readinessProbe: <Object> # 就绪检查，从Service中剔除容器，参数配置同 livenessProbe
        resources: <Object> #资源配置
          requests: <map[string]string> #最小资源配置
            memory: "1024Mi"
            cpu: "500m" #500m代表0.5CPU
          limits: <map[string]string> #最大资源配置
            memory:
            cpu:         
      volumes: <[]Object> #数据卷配置
      - name: <string> -required- #设置卷名称,与volumeMounts名称对应
        hostPath: <Object> #设置挂载宿主机路径
          path: <string> -required- 
          type: <string> #类型：DirectoryOrCreate、Directory、FileOrCreate、File、Socket、CharDevice、BlockDevice
      - name: nfs
        nfs: <Object> #设置NFS服务器
          server: <string> -required- #设置NFS服务器地址
          path: <string> -required- #设置NFS服务器路径
          readOnly: <boolean> #设置是否只读
      - name: configmap
        configMap: 
          name: <string> #configmap名称
          defaultMode: <integer> #权限设置0~0777，默认0664
          optional: <boolean> #指定是否必须定义configmap或其keys
          items: <[]Object>
          - key: <string> -required-
            path: <string> -required-
            mode: <integer>
      restartPolicy: <string> #重启策略，Always、OnFailure、Never
      nodeName: <string>
      nodeSelector: <map[string]string>
      imagePullSecrets: <[]Object>
      hostname: <string>
      hostPID: <boolean>
status: <Object> # 记录了对象的当前状态，Kubernetes 系统负责填充或更新，用户不能手动进行定义
```

#### 4.2.3 nginx deployment

通过yaml文件配置创建一个名为 deploy-nginx 的 deployment 里面pod名为 deploy-nginx 镜像为nginx 副本数为 3

```yaml
# 初始化得到deploy-nginx.yaml
kubectl create deploy web-nginx --image=nginx --dry-run=client -o yaml > web-nginx.yaml 

# 修改如下
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
# 创建 deployment
kubectl apply -f web-nginx.yaml
```

### 4.3 service

https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/

#### 4.3.1  作用

**(1) 作用**

Service引入主要是解决pod的动态变化，提供统一的访问入口：

- 防止Pod 失联，找到提供同一个服务的POD（服务发现）
- 定义一组POD的访问策略（负载均衡）
- 通过标签关联一组pod，为其提供统一的访问入口以及负载均衡

**(2) 统一访问，负载均衡**

分流常规思路，例如在Nginx中

- 基于域名
- 基于端口
- 基于IP

在kubernetes中访问Service则类似于后两种，即基于IP和端口，各个Servcie之间集群内访问通过CLUSTER-IP，集群外则通过定义NodePort属性对外暴露服务

一般负载均衡器所说的四层和七层怎么理解

四层（传输层）：基于IP和端口进行转发

七层（应用层）：基于协议转发，例如http，可以实现基于域名、URL、Cookie等

**(3) 多端口Service定义**

对于某些服务，需要公开多个端口，Service也需要配置多个端口定义，通过端口名称区分

**(4) 三种常用类型**

- ClusterIP 集群内部使用

  默认，分配一个稳定的IP地址，即VIP，只能在集群内部访问

- NodePort 对外暴露应用

  在每个节点上启用一个端口来暴露服务，可以在集群外部访问，同样会分配一个稳定的内部集群IP地址。

  访问地址：任意Node节点IP:NodePort

  端口范围：30000-32767

  **问题：**实际情况下，对用户暴露应只有一个IP与端口，那么多个Node该使用那台让用户访问呢？

  **解决办法：**通过添加一个公网负载均衡器为项目提供统一访问入口。

  ss -antp | rep 30211

- LoadBalance 对外暴露应用，适用公有云

  与NodePort类似，在每个节点上启用一个端口来暴露服务，除此之外，kubernetes会请求底层云平台（例如阿里云、腾讯云、AWS等）上的负载均衡器，将每个Node([NodeIP]:[NodePort])，作为后端添加进去。

**(5) 代理模式**

Service的底层实现主要有Iptables和IPVS两种网络模式，决定了如何转发流量

```bash
# 查看当前使用的代理模式
kubectl logs kube-proxy-9kc94 -n kube-system |more
```

- Iptables (默认)

  规则遍历匹配和更新，呈线性时延
  
- IPVS

  工作在内核态，有更好的性能
  
  调度算法更丰富：rr、wrr、lc、ip hash
  
  ```bash
  # 修改ipvs模式：
  kubectl edit configmap kube-proxy -n kube-system
  ...
  mode: “ipvs“
  ...
  kubectl delete pod kube-proxy-btz4p -n kube-system
  
  # 注：
  # 1、kube-proxy配置文件以configmap方式存储
  # 2、如果让所有节点生效，需要重建所有节点kube-proxy pod
  
  # 二进制方式修改ipvs模式：
  vi kube-proxy-config.yml 
  mode: ipvs
  ipvs:
  scheduler: "rr“
  systemctl restart kube-proxy
  
  ```

![image-20221129173439441](attachments/image-20221129173439441.png)

访问一个ServiceIP 其背后实际上是iptables转发到具体的pod上，service的规则则由kube-proxy维护，kube-proxy从apiserver获取到所有创建的service，进而创建相应的规则。

第一步：集群外NodePort访问 或 集群内ClusterIP访问

第二步：负载均衡

第三步：DNAT转发到容器

![image-20221129173208548](attachments/image-20221129173208548.png)

**(6) Service DNS**

CoreDNS 是一个DNS服务器，Kubernetes默认采用，以Pod部署在集群中，CoreDNS服务监视KubernetesAPI，为一个Service创建DNS记录用于域名解析

ClusterIP A记录格式： service-name.namespace-name.svc.cluster.local

例如：nslookup web.default.svc.cluster.local



```bash
# 获取 服务对应的后端
kubectl get endpoint -w
<service-name>
# 创建
# 基于现有deployment创建
kubectl expose pod pod-test --name=front-end-service --port=80 --target-port=80 \ 
--type=NodePort --dry-run=client -o yaml > front-end-service.yaml

# yaml文件
kubectl apply -f service.yaml

# 查看
kubectl get service
kubectl get svc
# 流程包流程：客户端 ->NodePort/ClusterIP（iptables/Ipvs负载均衡规则） -> 分布在各节点Pod
# 查看负载均衡规则：
# -- iptables模式
iptables-save |grep service-name
# -- ipvs模式
ipvsadm -L -n
```

#### 4.3.2 Service yaml 模板

```yaml
apiVersion: v1
kind: Service
matadata:                                #元数据
  name: string                           #service的名称
  namespace: string                      #命名空间
  labels:                                #自定义标签属性列表
    - name: string
  annotations:                           #自定义注解属性列表
    - name: string
spec:                                    #详细描述
  selector: []                           #label selector配置，将选择具有label标签的Pod作为管理 范围
  type: string                           #service的类型，指定service的访问方式，默认为clusterIp
  clusterIP: string                      #虚拟服务地址
  sessionAffinity: string                #是否支持session
  ports:                                 #service需要暴露的端口列表
  - name: string                         #端口名称
    protocol: string                     #端口协议，支持TCP和UDP，默认TCP
    port: int                            #服务监听的端口号，Service端口，通过Cluster IP访问
    targetPort: int                      #容器端口（应用程序监听端口）镜像内服务端口，如nginx镜像为80
    nodePort: int                        #当type = NodePort时，指定映射到物理机的端口号
  status:                                #当spce.type=LoadBalancer时，设置外部负载均衡器的地址
    loadBalancer:                        #外部负载均衡器
      ingress:                           #外部负载均衡器
        ip: string                       #外部负载均衡器的Ip地址值
        hostname: string                 #外部负载均衡器的主机名
```

(2) 创建web-nginx-service 服务，并绑定上一步创建的名为web-nginx 的 deployment

```yaml
# 初始化得到deploy-nginx.yaml
kubectl create deploy web-nginx-service --image=nginx --dry-run=client -o yaml > web-nginx-service.yaml 
kubectl expose deployment web-nginx-service --port=80 --target-port=8080 --type=NodePort -o yaml > web-nginx-service.yaml
# 修改如下
apiVersion: v1
kind: Service
metadata:
  name: web-nginx-service
spec:
  selector:
    app: nginx-web
  ports:
    - protocol: TCP
      port: 8002
      targetPort: 80
  type: NodePort
# 创建 Service
kubectl apply -f web-nginx-service.yaml
```

https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/

集群外访问效果

![image-20221123101346203](attachments/image-20221123101346203.png)

### 4.4 pod

#### 4.4.1 概念及组件

**(0) 一个pod创建的工作流程**

kubectl run pod-name --image=nginx

1、kubectl 向apiserver 发起一个创建POD的请求

2、apiserver 收到创建POD请求，将配置信息写入ETCD

3、scheduler检测到未分配到节点的POD，根据自身的分配算法选择一个合适的节点进行绑定，例如绑定到nodeName=k8s-node1

4、kubelet收到apiserver反馈的信息：通知kubelet有新pod要在你的节点上运行，kubelet获取分配到自己节点上的pod配置，调用docker api 创建容器，kubelet再将pod状态上报到api server 

---

controller-manger : 创建 deployment --> 创建一个RS --> 创建指定POD副本 --> 创建具体POD

kube-proxy :负责 service 资源的规则维护

---

**(0) pause**

重要概念：Pod内的容器都是平等的关系，共享Network Namespace、共享文件

pause容器的最主要的作用：创建共享的网络名称空间，以便于其它容器以平等的关系加入此网络名称空间

pause进程是pod中所有容器的父进程（即第一个进程）；

我们看下在node节点上都会起很多pause容器，和pod是一一对应的。

每个Pod里运行着一个特殊的被称之为Pause的容器，其他容器则为业务容器，这些业务容器共享Pause容器的网络栈和Volume挂载卷，

因此他们之间通信和数据交换更为高效，在设计时我们可以充分利用这一特性将一组密切相关的服务进程放入同一个Pod中。同一个Pod里的容器之间仅需通过localhost就能互相通信。

**kubernetes中的pause容器主要为每个业务容器提供以下功能：**

- PID命名空间：Pod中的不同应用程序可以看到其他应用程序的进程ID。

- 网络命名空间：Pod中的多个容器能够访问同一个IP和端口范围。

- IPC命名空间：Pod中的多个容器能够使用SystemV IPC或POSIX消息队列进行通信。

- UTS命名空间：Pod中的多个容器共享一个主机名；Volumes（共享存储卷）：

Pod中的各个容器可以访问在Pod级别定义的Volumes。

关于 Pod 最重要的一个事实是：它只是一个逻辑概念。

 **(1) 生命周期**: https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/

** (2)主要用法**：

- 运行单个容器，可以将pod看作是单个容器的抽象封装
- 运行多个容器：边车模式（Sidecar），通过在Pod中定义专门容器来执行主业务容器需要的辅助工作，进而将辅助功能同主业务容器解耦，实现独立发布和能力重用，例如日志收集、应用监控。

**(3) 资源共享实现机制**

- 共享网络：将业务容器网络加入到网络容器info container (docker ps -a | grep pause)，且主容器和辅助容器共享网络容器的网络命名空间。
- 共享存储：容器通过数据卷共享数据。

**(4) init container**

- 负责初始化工作，为一次性任务，支持大部分应用容器配置，但不支持健康检查，优先于应用容器执行。
- 环境检查：例如确保应用容器依赖的服务启动后在启动应用容器。
- 初始化配置：例如给应用容器准备配置文件。

示例：部署一个web应用，应用程序未在镜像中，希望从代码仓库中动态拉取放到应用容器中。

思路，initContainers 和 主容器 共享存储，initContainers 拉取web页面至共享存储即可

**(5) Static POD**

- Pod 由特定节点上的kubelet管理
- 不能使用控制器
- Pod名称标识当前节点名称

在kubelet配置文件启用 Static Pod 的参数

```bash
vi /var/lib/kubelet/config.yaml

staticPodPath: /etc/kubernetes/manifests
# 注：将部署的 pod yaml 放到该目录会由kubelet自动创建
# 验证 看 k8s-master 下 /etc/kubernetes/manifests 内 包含了etcd、kube-apiserver等yaml
```

**(6) 重启策略与健康检查**

- restartPolicy : Always、OnFailure、Never
- livenessProbe 存活检查
- redinessProbe 就绪检查
- strartupProbe 启动检查
- 三种检查方法：httpGet、exec、tcpSocket

**(7) 环境变量**

**(8) 资源限制**

- resources : 资源限制和请求的设置

    requests:

  ​    memory: 512Mi

  ​    cpu: 1

    limits:

  ​    memory: 1024Mi

  ​    cpu: 2

  m 是 CPU 的单位，即毫核

  1 核 = 1000 m

  0.5 核 = 500 m

1、一般建议requests的值小于limits的值的20%-30%
2、requests的值必须小于limits的值
3、limits是可以设置大于宿主机实际物理配置的，但不建议设置高于宿主机配置，否则没有限制的意义，一般最高设置低于主机20%
4、requests的值没有节点能够满足时，pod处于pendding
5、如果requests的值设置太大，占用节点资源分配，会导致宿主机资源利用率低，而节点资源分配饱和，导致无法分配新的POD   

**(9) 调度**

**(9.1) pod 角度**

- nodeSelector ：用于将pod调度到指定Label的Node节点上
- nodeAffinity：节点亲和度，同样是根据指定标签来调度pod至相应Node节点，支持操作符：In、NotIn、Exists、DoesNotExist等，又分为软策略(preferredDuringSchedulingIgnoreDuringExecution)和硬策略(requiredDuringSchedulingIgnoreDuringExecution)。

**(9.2) node角度**

- Taint：避免POD调度到特定的Node节点上
- Tolerations
- nodeName

**(9.3) 常见调度失败的原因 (pending)**

CPU 不足提示

没有匹配节点标签提示

所有的节点都有污点，但调度的pod未配置污点容忍

```bash
# 第一步：给节添加污点
kubectl taint node [node] key=value:[effect]

# 例子:
kubectl taint node k8s-node1 gpu=yes:NoSchedule
验证：
kubectl describe node k8s-node1 | grep Taint

NoSchedule 可取值 ：
NoSchedule：一定不能被调度
PreferNoschedule：尽量不要调度，非必要配置容忍
NoExecute：不仅不会调度，还会驱逐Node上已有的POD
```

#### 4.4.2 pod yaml模板

```yaml
apiVersion: v1        　　          #必选，版本号，例如v1,版本号必须可以用 kubectl api-versions 查询到 .
kind: Pod       　　　　　　         #必选，Pod
metadata:       　　　　　　         #必选，元数据
  name: string        　　          #必选，Pod名称
  namespace: string     　　        #必选，Pod所属的命名空间,默认为"default"
  labels:       　　　　　　         #自定义标签
    - name: string      　          #自定义标签名字
  annotations:        　　                 #自定义注释列表
    - name: string
spec:         　　　　　　　            #必选，Pod中容器的详细定义
  containers:       　　　　            #必选，Pod中容器列表
  - name: string      　　                #必选，容器名称,需符合RFC 1035规范
    image: string     　　                #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像,否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]     　　        　　#容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      　　             #容器的启动命令参数列表
    workingDir: string                  #容器的工作目录
    volumeMounts:     　　　　        　　#挂载到容器内部的存储卷配置
    - name: string      　　　        　　#引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string                 #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean                 #是否为只读模式
    ports:        　　　　　　        　　 #需要暴露的端口库号列表
    - name: string      　　　        　　#端口的名称
      containerPort: int                #容器需要监听的端口号
      hostPort: int     　　             #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string                  #端口协议，支持TCP和UDP，默认TCP
    env:        　　　　　　            #容器运行前需设置的环境变量列表
    - name: string      　　            #环境变量名称
      value: string     　　            #环境变量的值
    resources:        　　                #资源限制和请求的设置
      limits:       　　　　            #资源限制的设置
        cpu: string     　　            #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string                  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:       　　                #资源请求的设置
        cpu: string     　　            #Cpu请求，容器启动的初始可用数量
        memory: string                    #内存请求,容器启动的初始可用数量
    livenessProbe:      　　            #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:       　　　　　　        #对Pod容器内检查方式设置为exec方式
        command: [string]               #exec方式需要制定的命令或脚本
      httpGet:        　　　　        #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:      　　　　　　#对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
    restartPolicy: [Always | Never | OnFailure]   # Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject   # 设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:     　# Pull镜像时使用的secret名称，以key：secretkey格式指定
      - name: string
    hostNetwork: false     　# 是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    
  volumes:        　　　　　  # 在该pod上定义共享存储卷列表
    - name: string     　　  # 共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}      　　 # 类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string       # 类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string      　 # Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:       　　　　　# 类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:      　　　　# 类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
      nfs:                         # 类型为NFS的存储卷
        server: 192.168.66.50      # nfs服务器ip或是域名 
        path: "/test"              # nfs服务器共享的目录
      persistentVolumeClaim:       # 类型为persistentVolumeClaim的存储卷
        claimName: test-pvc        # 名字一定要正确，使用的是kind为PersistentVolumeClaim中的name

```

### 4.5 ingress

https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/

![image-20221130230104052](attachments/image-20221130230104052.png)

弥补NodePort的不足，因为NodePort只支持四层负载均衡，一个端口只能一个服务使用，端口需要提前规划。

实际上，ingress相当于一个7层的负载均衡器，是k8s对反向代理的一个抽象。大概的工作原理也确实类似于Nginx，**可以理解成在 Ingress 里建立一个个映射规则 , ingress Controller 通过监听 Ingress里的配置规则并转化成 Nginx 的配置** , 然后对外部提供服务。ingress包括：在这里有两个核心概念：

Ingress：公开从集群外部到集群内服务的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。是kubernetes中的一个抽象资源，给管理员提供一个暴露应用入口的定义方法。作用是定义请求如何转发到service的规则

Ingress Controller：负责流量路由，根据Ingress生成的具体路由规则，并对Pod负载均衡，Ingress管理的负载均衡器，为集群提供全局的负载均衡能力。

- 部署Ingress Controller
- 创建Ingress规则

#### 4.5.1 工作原理

**以nginx为例**

1、用户编写ingress规则，说明哪个域名对应 kubernetes 集群中的哪个service；
2、ingress控制器动态感知ingress服务的规则的变化，然后生成一段对应的nginx配置；
3、ingress控制器会将生成的nginx配置写入到一个运行着的nginx服务中，并动态更新；
4、到此为止，其实真正工作的就是一个nginx了，内部配置了用户定义的请求转发规则；

#### 4.5.2  Ingress-nginx部署

 (1) 拉取 controller yaml

[Installation Guide - NGINX Ingress Controller (kubernetes.github.io)](https://kubernetes.github.io/ingress-nginx/deploy/#bare-metal-clusters)

```bash
# 拉取 ingress-nginx-deploy.yaml
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/baremetal/deploy.yaml
```

(2) 拉取  Ingress-nginx controller 镜像

 Ingress-nginx controller 与kubernetes 版本信息可以查看下面的链接

[GitHub - kubernetes/ingress-nginx: Ingress-NGINX Controller for Kubernetes](https://github.com/kubernetes/ingress-nginx)

镜像拉取不到，可以用下面的镜像加速转推到hub.docker.com上

https://github.com/anjia0532/gcr.io_mirror

```bash

sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": [
  "http://hub-mirror.c.163.com",
  "https://j16wttpi.mirror.aliyuncs.com",
  "https://mirror.baidubce.com"
  ]
}
systemctl restart docker.service

# 拉取镜像
docker pull anjia0532/google-containers.ingress-nginx.controller:v1.5.1
docker pull anjia0532/google-containers.ingress-nginx.kube-webhook-certgen:v20220916-gd32f8c343
```

(3) 创建 ingress-nginx controller

```bash
kubectl apply -f ingress-nginx-deploy.yaml
kubectl get pods -n ingress-nginx
```

#### 4.5.3 案例准备工作

![image-20221202005609799](attachments/image-20221202005609799.png)

```yaml
#（1）deployment nginx-deployment 和 tomcat-deployment 配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---------------------------------------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat
        ports:
        - containerPort: 8080

#（1）创建nginx-deployment 和 tomcat-deployment
kubectl apply -f nginx-deployment.yaml
kubectl apply -f tomcat-deployment.yaml

#（2）service nginx-service 和 tomcat-service
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
    - name: nginx
      port: 80
      targetPort: 80
  clusterIP: None
  selector:
    app: nginx-pod

--------------------------------------------------

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
spec:
  ports:
    - name: tomcat
      port: 8080
      targetPort: 8080
  clusterIP: None
  selector:
    app: tomcat-pod
    
#（2）创建 service nginx-service 和 tomcat-service
kubectl apply -f nginx-service.yaml
kubectl apply -f tomcat-service.yaml
# 查看
kubectl get svc
nginx-service        ClusterIP   None           <none>        80/TCP         67s
tomcat-service       ClusterIP   None           <none>        8080/TCP       27s
```

#### 4.5.4 http代理

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-http
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: one.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: two.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
# 创建
kubectl apply -f ingress-http.yaml
# 查看
kubectl get ingress
NAME                    CLASS           HOSTS                       ADDRESS   PORTS   AGE
ingress-http            nginx-example   test.one.com,test.two.com             80      12s
#查看svc端口
kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.5.17.108    <none>        80:30265/TCP,443:30571/TCP   24h
```

本地电脑 配置hosts

C:\Windows\System32\drivers\etc\hosts

```bash
# localhost name resolution is handled within DNS itself.
#	127.0.0.1       localhost
#	::1             localhost
192.168.22.160      one.com
192.168.22.160      two.com
```

验证：这里我手动修改了ngxin-deployment 3个pod 里的页面，可以看到三个pod随机访问，起到了负载均衡作用

![image-20221202001808869](attachments/image-20221202001808869.png)

![image-20221202005032495](attachments/image-20221202005032495.png)

#### 4.5.5 https 代理

(1) 创建证书

```bash
#生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=itheima.com"
#创建密钥，将证书文件保存到Secret
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

(2) 创建 ingress-https

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-https
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  tls:
  - hosts:
    - one.com
    - two.com
  rules:
  - host: one.com
    http:
      paths:
      - path: /s
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
  - host: two.com
    http:
      paths:
      - path: /s
        pathType: Prefix
        backend:
          service:
            name: tomcat-service
            port:
              number: 8080
              
# 创建
kubectl apply -f ingress-https.yaml

```

(3) 验证

![image-20221202102056769](attachments/image-20221202102056769.png)

![image-20221202144141313](attachments/image-20221202144141313.png)

80:30265/TCP,  443:32613/TCP

http                        https

域名 -> 公网负载均衡器(80,443) -> ingress controller(nodeport 30265) -> 分散到各个节点Pod

**公网负载均衡器用途？**

1、将kubernetes节点暴露到互联网上

2、统一访问入口(80, 443), 避免用户使用端口访问 (主要针对service nodeport)

3、加强kubernetes集群(内网)的安全性

### 4.6 volume

容器中的文件在磁盘中是临时存放的，这样给运行比较重要的应用程序带来一些问题。

- 问题1 当容器升级或者崩溃时，kubelet会重建容器，容器内文件会丢失
- 问题2 一个Pod中运行多个容器并需要共享文件

数据卷 volume 可以解决以上问题

常用的数据卷：

- 节点本地 (hostPath, emptyDir)
- 网络 (NFS, Ceph, GlusterFS)
- 公有云 (AWS EBS)
- K8S 资源 (configmap, secret)

#### 4.6.1 emptyDir

临时数据卷，伴随Pod生命周期存在，当Pod被销毁时同时被删除

应用场景：Pod中容器之间数据共享

```yaml
# 1 用--dry-run命令得到一份yaml文件
kubectl run log --image=busybox --dry-run=client -o yaml > log.pod.yaml
# 官网连接：https://kubernetes.io/docs/concepts/storage/volumes/

# 2 yaml文件如下
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: log
  name: log
spec:
  containers:
  - image: busybox
    name: log-pro
    command: ["sh","-c","echo important information >> /log/data/output.log;sleep 1d"]
    volumeMounts:
    - name: date-log
      mountPath: /log/data
    resources: {}
  - image: busybox
    name: log-cus
    command: ["sh","-c","cat /log/data/output.log;sleep 1d"]
    volumeMounts:
    - name: date-log
      mountPath: /log/data
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: date-log
    emptyDir: {}
status: {}

# 3 修改完yaml后，启动此yaml
kubectl  create -f log-pod.yaml

# 4 测试
kubectl logs log -c log-cus -- bash
# 输出 important information
kubectl  exec -it log -c log-pro -- cat /log/data/output.log
# 输出 important information

# node 节点查看 pod共享存储 在宿主机实际的路径
cd /var/lib/kubelet/pods/9ce9a5ea-eb10.../volumes/kubernetes.io~empty-dir/data
```

#### 4.6.2 hostpath

节点数据卷 hostpath：挂载Node文件系统 (Pod 所在节点) 上文件或者目录到 Pod 中的容器

应用场景：Pod中容器需要访问宿主机文件

```yaml
apiVersion: v1
kind: Pod
metadata:
name: my-pod
spec:
containers:
- name: busybox
image: busybox
args:
- /bin/sh
- -c
- sleep 36000
volumeMounts:
- name: data
mountPath: /data
volumes:
- name: data
hostPath:
path: /tmp
type: Directory

apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: busybox
    name: test-container
    command: ["sh","-c","i=0; while true; do echo $(date) $i >> /data/big-corp-app.log; i=$((i+1)); sleep 1; done"]
    volumeMounts:
    - mountPath: /data
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # 宿主上目录位置
      path: /home/kubernetes/data
      # 此字段为可选
      type: Directory
```

#### 4.6.3 NFS

网络数据卷：提供对NFS挂载支持，可以自动将NFS共享路径挂载到Pod中

mount -t nfs 192.168.31.73:/ifs/kubernetes /mnt/

umount /mnt/

```bash
# 各个Node上安装nfs-utils
yum install nfs-utils

vi /etc/exports
/ifs/kubernetes *(rw,no_root_squash)

mkdir -p /ifs/kubernetes
systemctl start nfs
systemctl enable nfs

```

**(1) 将Nginx网站程序根目录持久化到 NFS存储，为多个Pod提供网站程序文件**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-nfs
  name: web-nfs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-nfs
  template:
    metadata:
      labels:
        app: web-nfs
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html
        ports:
        - containerPort: 80
      volumes:
      - name: wwwroot
        nfs:
          server: 192.168.22.161
          path: /ifs/kubernetes
          
kubectl appply -f web-nfs.yaml
kubectl expose deployment web-nfs --port=80 --target-port=80 --type=NodePort
# 进入 pod
kubectl exec -it web-nfs-68557b665f-vwjck -- bash
cd /usr/share/nginx/html
echo "1234" > index.html

# 进入 slave1  查看
cat /ifs/kubernetes/index.html
```

#### 4.6.4 Persistent Volume

PersistemtVolume (PV)：对存储资源创建和使用的抽象，使得存储作为集群中的资源管理

PersistentVolumeClaim (PVC)：让用户不需要关心具体的Volume实现细节

Pod申请PVC作为卷来使用，kubernetes通过PVC查找绑定的PV，并挂载相应的Pod

存储类型：
- 块存储，例如云硬盘。不能共享的，独用
- 文件系统存储，例如阿里云NAS。可以共享的
- 对象存储，例如阿里云OSS。可以共享的

**(0) PV 生命周期**
AccessModes（访问模式）： AccessModes 是用来对 PV 进行访问模式的设置，用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：

- ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载 
- ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载 
- ReadWriteMany（RWX）：读写权限，可以被多个节点挂载 

RECLAIM POLICY（回收策略）： 目前 PV 支持的策略有三种： 

- Retain（保留）： 保留数据，需要管理员手工清理数据 
- Recycle（回收）：清除 PV 中的数据，效果相当于执行 rm -rf /ifs/kuberneres/*
- Delete（删除）：与 PV 相连的后端存储同时删除 STATUS（状态）：

 一个 PV 的生命周期中，可能会处于4中不同的阶段：

- Available（可用）：表示可用状态，还未被任何 PVC 绑定
- Bound（已绑定）：表示 PV 已经被 PVC 绑定
- Released（已释放）：PVC 被删除，但是资源还未被集群重新声明 
- Failed（失败）： 表示该 PV 的自动回收失败

**(1) pv+pvc 范例**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-deploy-pvc
  name: web-deploy-pvc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-deploy-pvc
  template:
    metadata:
      labels:
        app: web-deploy-pvc
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
      volumes:
      - name: www
        persistentVolumeClaim:
          claimName: my-pvc

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteMany
  nfs:
    path: /ifs/kubernetes
    server: 192.168.22.161

# 创建
kubectl apply -f web-deploy-pvc.yaml
# 验证
[root@master kubernetes]# kubectl get pv
NAME  CAPACITY ACCESS MODES RECLAIM POLICY STATUS  CLAIM    STORAGECLASS  REASON   AGE
my-pv 5Gi      RWX          Retain         Bound   default/my-pvc                  84s
[root@master kubernetes]# kubectl get pvc
NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
my-pvc   Bound    my-pv    5Gi        RWX                           87s
[root@master kubernetes]# kubectl get deployment
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
web-deploy-pvc   2/2     2            2           98s
web-nfs          3/3     3            3           4d23h
```

1、pvc与pv怎么匹配的？
存储容量和访问模式

2、容器匹配策略
匹配最接近的PV容器，如果匹配不到，则Pod处于pending状态。

3、pv与pvc关系
一对一

4、容量有限制作用？
没有限制作用， 目前这个主要用于匹配一种标记，具体可使用多少取决于实际宿主机文件系统容量限制的。

pv + pvc 的使用方式 是一种静态供给，需要提前创建PV，供开发使用，效率较低，维护成本高。

#### 4.6.5 StorageClass 

**(0) StorageClass 实现PV动态供给**

K8s默认不支持NFS动态供给，需要单独部署社区开发的插件。 

插件地址：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

选择手动部署，主要是执行github上的三个yaml文件

- rbac.yaml \# 授权访问apiserver
- deployment.yaml # 修改里面NFS服务器地址与共享目录 为自己的
- class.yaml # 创建存储类

**(1) SC + PVC 范例**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-deploy-pvc
  name: web-deploy-pvc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-deploy-pvc
  template:
    metadata:
      labels:
        app: web-deploy-pvc
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
      volumes:
      - name: www
        persistentVolumeClaim:
          claimName: my-pvc2

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc2
spec:
  storageClassName: "managed-nfs-storage"
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
---

kubectl apply -f sc-pvs.yaml
```

#### 4.6.6 ConfigMap

创建ConfigMap后，数据实际会存储至Kubernetes的ETCD中，当创建Pod时会引用该数据，故一般存储配置文件。

- 变量注入
- 数据卷挂载
- 数据类型：键值、多行数据

```yaml
# 命令行
kubectl create configmap c1 --from-literal=key-one=123 --dry-run=client \
-o yaml > pod-use-configmap.yaml

---
# configMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
data:
  abc: "123"
  cde: "456"
  redis.properties: |
    port: 6379
    host: 192.168.31.10
---
# pod use configMap
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
  - name: demo
    image: nginx
    env:
    - name: ABCD
      valueFrom:
        configMapKeyRef:
          name: configmap-demo
          key: abc
    - name: CDEF
      valueFrom:
        configMapKeyRef:
          name: configmap-demo
          key: cde
    volumeMounts:
    - name: config
      mountPath: "/config"
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: configmap-demo
      items:
      - key: "redis.properties"
        path: "redis.properties"

# 验证
kubectl exec -it configmap-demo-pod -- bash
echo $ABCD
cat /config/redis.properties
```

#### 4.6.7 Secret

Sercet主要存储敏感数据，所有的数据要经过base64编码

kubectl create secret 支持三种数据类型

- docker-registry：存储镜像仓库认证信息
- generic：密码
- tls：存储证书，例如从文件、目录或者字符串创建，如存储用户名 Https证书
- 使用数据的两种方式：变量注入、数据卷挂载

```yaml
# 将用户名密码进行编码：
echo -n 'admin' | base64
echo -n '1f2d1e2e67df' | base64

# 解密
echo -n 'YWRtaW4=' | base64 -d
# 创建 pod-use-secret.yaml
---
# secret
apiVersion: v1
kind: Secret
metadata:
  name: db-user-pwd
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm

---
# pod use secret
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo-pod
spec:
  containers:
  - name: demo
    image: nginx
    env:
    - name: USER
      valueFrom:
        secretKeyRef:
          name: db-user-pwd
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-user-pwd
          key: password
    volumeMounts:
    - name: config
      mountPath: "/config"
      readOnly: true
  volumes:
  - name: config
    secret:
      secretName: db-user-pwd
      items:
      - key: "username"
        path: "my-username"
      - key: "password"
        path: "my-password"

# 验证
kubectl get pods,secret
kubectl exec -it secret-demo-pod -- bash
echo $USER $PASSWORD
ls /config
```

### 4.7 StatefulSet

**(0) 定义**

> RC、Deployment、DaemonSet都是面向无状态的服务，它们所管理的Pod的IP、名字，启停顺序等都是随机的，而StatefulSet是什么？顾名思义，有状态的集合，管理所有有状态的服务，比如MySQL、MongoDB集群等。为了解决有状态服务的问题，它所管理的Pod拥有固定的Pod名称，启停顺序，在StatefulSet中，Pod名字称为网络标识(hostname)，还必须要用到共享存储。在Deployment中，与之对应的服务是service，而在StatefulSet中与之对应的headless service，headless service，即无头服务，与service的区别就是它没有Cluster IP，解析它的名称时将返回该Headless Service对应的全部Pod的Endpoint列表。除此之外，StatefulSet在Headless Service的基础上又为StatefulSet控制的每个Pod副本创建了一个DNS域名，这个域名的格式为：
> $(podname).(headless server name)
> FQDN：$(podname).(headless server name).namespace.svc.cluster.local

无状态应用特点

- 各个pod互相平等，无相互连接关系

- 使用共享存储
- 管理的所有Pod均相同，提供同一个服务，可随意扩容和缩容。典型的如Web服务 

有状态应用特点

MySQL 多个副本，主从架构，读写分离

- 各个pod间角色不同（不对等）
- pod 之间有连接关系
- 每个 pod 都有独立的存储
- 例如：分布式应用，会部署多个实例，这些实例之间往往有依赖关系，例如主从关系，主备关系等，典型的如MySQL主从，Etcd集群，zookeeper集群。

**(1) 特点**

Pod一致性：包含次序（启动、停止次序）、网络一致性。此一致性与Pod相关，与被调度到哪个node节点无关；

- 稳定的次序：对于N个副本的StatefulSet，每个Pod都在[0，N)的范围内分配一个数字序号，且是唯一的；

- 稳定的网络：使用Headless Service（相比普通Service只是将spec.clusterIP定义为None）来维护Pod网络身份。 并且添加serviceName: “nginx”字段指定StatefulSet控制器要使用这个Headless Service。
  ``` DNS解析名称：<statefulsetName-index>.<service-name>.<namespace-name>.svc.cluster.local ```

- 稳定的存储：StatefulSet的存储卷使用 VolumeClaimTemplate 创建，称为卷申请模板，当 StatefulSet 使用VolumeClaimTemplate创建 一个PersistentVolume时，同样也会为每个Pod分配并创建一个编号的PVC。

**(2) 部署 StatefulSet**

Headless Service：用来定义Pod网络标识( DNS domain)；
volumeClaimTemplates ：存储卷申请模板，创建PVC，指定pvc名称大小，将自动创建pvc，且pvc必须由存储类供应；
StatefulSet ：定义具体应用，名为Nginx，有三个Pod副本，并为每个Pod定义了一个域名部署statefulset。

```yaml
vi statefulset.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # 必须匹配 .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # 默认值是 1
  minReadySeconds: 10 # 默认值是 0
  template:
    metadata:
      labels:
        app: nginx # 必须匹配 .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-nfs-storage"
      resources:
        requests:
          storage: 1Gi

kubectl apply -f statefulset.yaml
# 查看
```

![image-20221211163334939](attachments/image-20221211163334939.png)

```bash
# 验证DNS名称 
kubectl run bs --image=busybox -- sleep 24h
kubectl exec -it bs -- sh
nslookup nginx.default.svc.cluster.local
nslookup web.default.svc.cluseter.local

# pod hostname
kubectl get pods
kubectl exec web-0 --hostname
```

StatefulSet 与 Deployment 的区别：前者是有身份的，即域名、主机名、存储（PVC）

针对有状态应用 通过 operator 进行管理

### 4.8 RBAC

K8S 安全控制框架主要由下面3个阶段进行控制，每一个阶段都支持插件方式，通过API Server配置来启用插件。 

1. Authentication（鉴权） ：kubernetes Apiserver 提供三种客户端身份认证

   - HTTPS证书认证：基于CA证书签名的数字证书认证 ( kubeconfig )
   - HTTP Token认证：通过一个Token来识别用户 (serviceaccount)
   - HTTP Base 认证：用户名+密码的方式认证 (1.19版本弃用)

2. Authorization（授权） ：RBAC根据API请求属性，决定允许还是拒绝。 比较常见的授权维度：

   - user：用户名
   - group：用户分组
   - 资源，例如pod、deployment
   - 资源操作方法：get，list，create，update，patch，watch，delete
   - 命名空间
   - API 组

3. Admission Control（准入控制）：Adminssion Control 实际上是一个准入控制器插件列表，发送到API Server的请求都需要经过这个列表中的每个准入控制器插件的检查，检查不通过，则拒绝请求。

   - 启用一个准入控制器：

     kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger 

   - 关闭一个准入控制器：

     kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny 

   - 查看默认启用：

     kubectl exec kube-apiserver-k8s-master -n kube-system -- kube-apiserver -h | grep enable-admission-plugins

**RBAC（[Role-Based Access Control](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#command-line-utilities)，基于角色的访问控制）**， 是K8s默认授权策略，并且是动态配置策略（修改即时生效）

- 主体：User, Group, ServiceAccount
- 角色：Role, ClusterRole
- 角色绑定：RoleBinding, ClusterRoleBinding

RoleBinding 在指定命名空间中执行授权，ClusterRoleBinding 在集群范围执行授权。

**(0) 范例：为指定用户授权访问不同命名空间权限**

- 用kubernetes CA 签发客户端证书

  https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/certificates/#cfssl
  
  https://kubernetes.io/zh-cn/docs/setup/best-practices/certificates/
```bash
# kubernetes 根证书
ls /etc/kubernetes/pki/ca.crt
# 安装 cfssl
tar -zxvf cfssl.tar.gz -C /usr/bin

cert.sh
---
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF

cat > test-csr.json <<EOF
{
  "CN": "test",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key -config=ca-config.json -profile=kubernetes test-csr.json | cfssljson -bare test
---
```

- 生成kubeconfig授权文件
```bash
kubeconfig.sh
---
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://192.168.22.160:6443 \
  --kubeconfig=test.kubeconfig

# 设置客户端认证
kubectl config set-credentials test \
  --client-key=test-key.pem \
  --client-certificate=test.pem \
  --embed-certs=true \
  --kubeconfig=test.kubeconfig

# 设置默认上下文
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=test \
  --kubeconfig=test.kubeconfig

# 设置当前使用配置
kubectl config use-context kubernetes --kubeconfig=test.kubeconfig
---
```

- 创建RBAC权限策略
```yaml
rbac.yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: test
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
kubectl apply -f rbac.yaml
```

- 指定kubeconfig文件测试权限：kubectl get pods --kubeconfig=aling.kubeconfig
```bash
[root@master rbac]# kubectl get pods --kubeconfig=test.kubeconfig 
NAME                                      READY   STATUS    RESTARTS        AGE
nfs-client-provisioner-7cd48d6855-q495f   1/1     Running   2 (2d2h ago)    5d1h
nginx-dns-78c98b6889-czcmw                1/1     Running   4 (2d2h ago)    7d22h
nginx-dns-78c98b6889-w9c6q                1/1     Running   4 (2d2h ago)    7d22h
web-0                                     1/1     Running   1 (2d2h ago)    2d5h
web-1                                     1/1     Running   1 (2d2h ago)    2d5h
web-2                                     1/1     Running   1 (2d2h ago)    2d5h
[root@master rbac]# kubectl get svc --kubeconfig=test.kubeconfig 
Error from server (Forbidden): services is forbidden: User "test" cannot list resource "services" in API group "" in the namespace "default"
```

**(1) serviceaccount 范例**

```bash
# (0) kubectl 命令
# 创建集群角色
kubectl create clusterrole deployment-clusterrole \
--verb=create --resource=deployments,daemonsets,statefulsets
# 创建服务账号
kubectl create serviceaccount cicd-token -n app-team1
# 将服务账号绑定角色
kubectl create rolebinding cicd-token \
--serviceaccount=app-team1:cicd-token \
--clusterrole=deployment-clusterrole -n app-team1
# 测试服务账号权限
kubectl --as=system:serviceaccount:app-team1:cicd-token \
get pods -n app-team1
```

```yaml
# (1) yaml 文件
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-token
  namespace: app-team1
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-clusterrole
rules:
- apiGroups: ["apps"]
  resources: ["deployments","daemonsets","statefulsets"] 
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-token
  namespace: app-team1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: deployment-clusterrole
subjects:
- kind: ServiceAccount
  name: cicd-token
  namespace: app-team1
```

### 4.9 NetworkPolicy

https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/

 **(0) 准备工作**

```bash

kubectl create ns test-one
kubectl create ns test-two
kubectl -n test-one run bs --image=busybox --labels=run=bs -- sleep 24h
kubectl -n test-one run web-1 --image=nginx

kubectl -n test-two run bs2 --image=busybox --labels=run=bs -- sleep 24h
kubectl -n test-two run web-2 --image=nginx

kubectl -n test-one get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP               NODE           
bs      1/1     Running   0          4h32m   192.168.36.120   k8s-node1
web-1   1/1     Running   0          4h25m   192.168.36.121   k8s-node1

kubectl -n test-two get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP               NODE  
bs2     1/1     Running   0          4h14m   192.168.36.116   k8s-node2
web-2   1/1     Running   0          4h14m   192.168.36.119   k8s-node1
```

**(1) 拒绝命名空间下所有Pod出入站流量**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: test
spec:
  podSelector: {} # 匹配本命名空间所有pod
  policyTypes: 
  - Ingress
  - Egress
# ingress和egress没有指定规则，则不允许任何流量进出pod
```

**(2) 拒绝其他命名空间Pod访问**

需求：test-one命名空间下所有pod可以互相访问，也可以访问其他。
命名空间Pod，但其他命名空间不能访问test-one命名空间Pod。

```yaml
# (1) 此时无网络策略，pod内外命名空间都能访问
kubectl -n test-one exec -it bs -- sh
ping 192.168.36.119 # 通
kubectl -n test-two exec -it bs2 -- sh
192.168.36.121 # 通

# (2) 创建网络策略
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-namespaces 
  namespace: test-one
spec:
  podSelector: {} 
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
kubectl apply -f deny-all-namespaces.yaml

# (3) 验证
kubectl -n test-one get networkpolicy
kubectl -n test-one exec -it bs -- sh
ping 192.168.36.119 # 通
kubectl -n test-two exec -it bs2 -- sh
ping 192.168.36.121 # 不通
```

**(3) 允许其他命名空间Pod访问指定应用**

需求：允许其他命名空间访问test-one命名空间指定Pod

```yaml
kubectl -n test-one get pods --show-labels
NAME    READY   STATUS    RESTARTS   AGE     LABELS
bs      1/1     Running   0          5h41m   run=bs
web-1   1/1     Running   0          5h34m   run=web-1
web-2   1/1     Running   0          14s     run=web-2

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: test-one
spec:
  podSelector:
    matchLabels:
      run: web-1
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector: {} # 匹配所有命名空间的POD
---

kubectl apply -f test-network-policy.yaml

kubectl -n test-one get networkpolicy

kubectl -n test-two exec -it bs2 -- sh
ping 192.168.36.121 # 通
```

**(4) 同一个命名空间下应用之间限制访问**

需求：将test-one命名空间中标签为run=web的pod隔离， 只允许标签为run=client1的pod访问80端口

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-to-app
  namespace: test-one
spec:
  podSelector:
    matchLabels:
      run: web-1
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: web-2
    ports:
    - protocol: TCP
      port: 80
 
kubectl -n test-one exec -it bs -- sh
ping 192.168.36.121 # 不通

kubectl -n test-one run bs2 --image=busybox --labels=run=web-2 -- sleep 24h

```

## 05 监控

### 5.1 查看资源集群状态

| 含义                            | 命令                                      |
| ------------------------------- | ----------------------------------------- |
| 查看master组件状态              | kubectl get cs                            |
| 查看node状态                    | kubectl get node                          |
| 查看ApiServer代理的URL          | kubectl cluster-info                      |
| 查看集群详细信息                | kubectl cluster-info dump                 |
| 查看资源的详细                  | kubectl describe <资源>  <名称 >          |
| 查看资源信息                    | kubectl get <资源>                        |
| 查看所有deployments信息         | kubectl get deploy -a                     |
| 将deployment的副本数目调整为n个 | kubectl scale deployment web --replicas=n |
| 查看Node资源消耗：              | kubectl top node <node name>              |
| 查看Pod资源消耗：               | kubectl top pod <pod name>                |
|                                 |                                           |

补充：

- kubectl get <资源类型> <资源名称> # -o wide、-o yaml

- master和node怎么区分？

  根据部署的组件区分

  master：apiserver、control manager、schedule

  nodes：kubelet、kube-proxy、pod

### 5.2 监控集群资源利用率

> 从Kubernetes 1.8开始，Kubernetes通过Metrics API提供资源使用指标，例如容器CPU和内存使用率。这些度量可以由用户直接访问（例如，通过使用kubectl top命令），或者由集群中的控制器（例如，Horizontal Pod Autoscaler）使用来进行决策，具体的组件为Metrics-Server。其是一个集群范围的资源使用情况的数据聚合器。作为一 个应用部署在集群中。Metric server从每个节点上Kubelet API收集指 标，通过Kubernetes聚合器注册在Master APIServer中。 为集群提供Node、Pods资源利用率指标。
>
> 地址：https://github.com/kubernetes-sigs/metrics-server

```bash
# 下载 metric server 组件
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 增加一个kubelet-insecure-tls参数，
# 这个参数作用是告诉metrics-server不验证kubelet提供的https证书
```
![image-20221121211544441](attachments/image-20221121211544441.png)

```bash
# 安装 metrics-server
kubectl apply -f components.yaml

# 查看安装状态
kubectl get deploy -A

# 一直处于未ready状态，进一步查看pod信息
kubectl get pods -n kube-system | grep metrics

# 通过 describe 查看pod信息
kubectl describe pod/metrics-server-7fb756c8f5-c9w59 -n kube-system

# 拉取镜像k8s.gcr.io/metrics-server/metrics-server:v0.6.1 超时，手动拉取
docker pull cnbugs/metrics-server:latest

# 同时修改镜像地址为
kubectl edit deployment metrics-server -n kube-system
```

![image-20221121210224090](attachments/image-20221121210224090.png)

```bash
# 再次查看metrics-server状态已正常
kubectl get deploy -A
# 检查是否部署成功：
kubectl get apiservices |grep metrics
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes

```

![image-20221121211625943](attachments/image-20221121211625943.png)

```bash
# 查看Node资源消耗：
kubectl top node <node name>

# 查看Pod资源消耗：
kubectl top pod <pod name>
```



kubectl top -> 聚合器(metric-server) -> 采集器 (cadvisor)

kubectl top -> apiserver -> metric-server (pod) -> kubelet(cadvisor)

一个自签证书站点，有一个http客户端要访问它，需要两种方式：

1、客户端（例如curl）携带CA证书和客户端证书

2、客户端忽略证书验证

### 5.3 监控系统

![image-20221121211208556](attachments/image-20221121211208556.png)

02 日志

|                                  |                                                              |
| -------------------------------- | ------------------------------------------------------------ |
| systemd守护进程管理的组件        | journalctl -u kubelet                                        |
| Pod部署的组件：                  | kubectl logs pod-name -n kube-system                         |
| 系统日志：                       | /var/log/messages                                            |
| 查看容器标准输出日志：           | kubectl logs pod名称  \| kubectl logs -f  pod名称            |
| 标准输出在宿主机的路径           | /var/lib/docker/containers/<container-id>/<container-id>-json.log |
| 日志文件，进入到终端日志目录查看 | kubectl exec -it <Pod名称> -- bash                           |

查看容器日志的流程
kubectl logs -> apiserver -> kubelet -> docker (托管容器标准输出写到某个文件)

注意：
kubeadm容器化部署kubernetes集群，除kubelet组件外。通过kubectl logs查看的日志都是标准输出的日志。

2.1 针对标准输出

![image-20221121215941293](attachments/image-20221121215941293.png)

以DaemonSet方式在每个Node 上部署一个日志收集程序，采集 /var/lib/docker/containers/目录下所有容器日志

DaemonSet : https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/daemonset/

2.2 针对容器内日志文件

![image-20221121220013228](attachments/image-20221121220013228.png)

在Pod中增加一个容器运行日志采集器，使用emtyDir共享日志目录让日志采 集器读取到日志文件

Volumes：https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#emptydir



## 06 维护

### 6.1 Etcd 数据库备份与恢复

Kubernetes 使用 Etcd 数据库实时存储集群中的数据，故需进行备份

- kubeadm部署方式：

```bash
# (0) 备份
ETCDCTL_API=3 etcdctl \
snapshot save snap.db \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key

# (1) 恢复
# 1.1 先暂停kube-apiserver和etcd容器
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak
mv /var/lib/etcd/ /var/lib/etcd.bak
# 1.2 恢复
ETCDCTL_API=3 etcdctl \
snapshot restore snap.db \
--data-dir=/var/lib/etcd
# 1.3 启动kube-apiserver和etcd容器
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests
```

- 二进制部署方式

```bash
# (0) 备份
ETCDCTL_API=3 etcdctl \
snapshot save snap.db \
--endpoints=https://192.168.31.71:2379 \
--cacert=/opt/etcd/ssl/ca.pem \
--cert=/opt/etcd/ssl/server.pem \
--key=/opt/etcd/ssl/server-key.pem

# (1) 恢复
# 1.1 先暂停kube-apiserver和etcd
systemctl stop kube-apiserver 
systemctl stop etcd
mv /var/lib/etcd/default.etcd /var/lib/etcd/default.etcd.bak
# 1.2 在每个节点上恢复
ETCDCTL_API=3 etcdctl snapshot restore snap.db \
--name etcd-1 \
--initial-cluster="etcd-1=https://192.168.31.71:2380,etcd2=https://192.168.31.72:2380,etcd-3=https://192.168.31.73:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.31.71:2380 \
--data-dir=/var/lib/etcd/default.etcd
# 1.3 启动kube-apiserver和etcd
systemctl start kube-apiserver 
systemctl start etcd
```

### 6.2 使用 kubeadm 进行集群版本升级

```mermaid
graph LR
升级管理节点--> A(升级其它管理节点)
A--> 升级工作节点
```

注意事项：

- 升级前必须备份所有组件及数据，例如etcd 等。
- 不要跨多个小版本进行升级，例如从1.16升级到1.19。

#### 6.2.1 管理节点升级

```bash
# 1、查找最新版本号
yum list --showduplicates kubeadm --disableexcludes=kubernetes
# 2、升级kubeadm
yum install -y kubeadm-1.21.0-0
# 3、驱逐node上的pod，且不可调度
kubectl drain k8s-master --ignore-daemonsets
# 4、检查集群是否可以升级，并获取可以升级的版本
kubeadm upgrade plan
# 5、执行升级
kubeadm upgrade apply v1.21.0
# 6、升级kubelet和kubectl
yum install -y kubelet-1.21.0-0 kubectl-1.21.0-0
# 7、重启kubelet
systemctl daemon-reload
systemctl restart kubelet
# 8、取消不可调度
kubectl uncordon k8s-master
```

#### 6.2.2 工作节点升级

```bash
# 1、升级kubeadm
yum install -y kubeadm-1.21.0-0
# 2、驱逐node上的pod，且不可调度
kubectl drain k8s-node1 --ignore-daemonsets
# 3、升级kubelet配置
kubeadm upgrade node
# 4、升级kubelet和kubectl
yum install -y kubelet-1.21.0-0 kubectl-1.21.0-0
# 5、重启kubelet
systemctl daemon-reload
systemctl restart kubelet
# 6、取消不可调度，节点重新上线
kubectl uncordon k8s-node1
```

### 6.3 集群节点下线流程

当某个节点需要维护或删除时，流程如下

```bash
# 1、获取节点列表
kubectl get node
# 2、驱逐节点上的Pod并设置不可调度（cordon）
kubectl drain <node_name> --ignore-daemonsets
# 3、设置可调度或者移除节点
kubectl uncordon <node_name> 
kubectl delete node <node_name>
```

### 6.4 集群排错思路

```bash
# 查看资源详情
kubectl describe TYPE/NAME
# 查看Pod日志
kubectl logs TYPE/NAME [-c CONTAINER]
# 进入容器终端
kubectl exec POD [-c CONTAINER] -- COMMAND [args...]
```

常见问题：

- 集群组件不能正常工作

  1、网络不通。 2、启动失败，一般配置文件或者依赖服务。3、操作系统，平台不兼容，版本兼容性

- Service 访问异常

  1、Service是否关联Pod？ 2、Service指定target-port端口是否正常？ 3、Pod正常工作吗？ 4、Service是否通过DNS工作？ 5、kube-proxy正常工作吗？ 6、kube-proxy是否正常写iptables规则？ 7、cni网络插件是否正常工作？

## 07 项目部署案例

### 7.1 容器交付流程

```mermaid
graph LR
开发阶段 --> B(持续集成阶段)
B --> C(应用部署阶段)
C --> D(运维阶段)
```
- 开发阶段：开发应用程序，编写Dockerfile；

- 持续集成阶段：对应用程序编译打包、使用Dockerfile构建镜像，把镜像推送到镜像仓库；

- 应用部署阶段：基于镜像创建pod,使用deploy控制器暴露服务得到service，使用ingress提供域名访问服务；

- 运维阶段：对应用程序进行监控和版本升级等；

### 7.2 在K8s平台部署项目流程

```mermaid
graph LR
制作镜像 --> B(使用控制器部署镜像)
B --> C(对外暴露应用)
C --> D(日志与监控)
D --> E(日常运维)
```

### 7.3 在K8s平台部署Java网站项目

- 制作镜像

  ```bash
  # 使用镜像仓库（私有仓库、公共仓库）：
  # 1、配置可信任 集群内节点全部都要配置（如果仓库是HTTPS访问不用配置）
  # vi /etc/docker/daemon.json
  {
  "insecure-registries": ["192.168.22.160"]
  }
  # 2、将镜像仓库认证凭据保存在K8s Secret中
  kubectl create secret docker-registry registry-auth \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --docker-server=192.168.22.160
  # 3、在yaml中使用这个认证凭据
  imagePullSecrets:
  - name: registry-auth
  ```

- 使用控制器部署镜像

  (1) 使用configmap保存项目配置文件

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: java-demo-config
  data:
    application.yml: |
      server:
        port: 8080
      spring:
        datasource:
          url: jdbc:mysql://localhost:3306/test?characterEncoding=utf-8
          username: root
          password: 123456
          driver-class-name: com.mysql.jdbc.Driver
        freemarker:
          allow-request-override: false
          cache: true
          check-template-location: true
          charset: UTF-8
          content-type: text/html; charset=utf-8
          expose-request-attributes: false
          expose-session-attributes: false
          expose-spring-macro-helpers: false
          suffix: .ftl
          template-loader-path:
          - classpath:/templates/
  ---
  ```

  (2) Pod配置数据卷使用configmap

  ```yaml
  kubectl create deployment java-demo --image=java-demo:v1 --replicas=3 --dry-run=client -o yaml > /home/test-2/java-demo.yaml
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    labels:
      app: java-demo
    name: java-demo
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: java-demo
    strategy: {}
    template:
      metadata:
        labels:
          app: java-demo
      spec:
        imagePullSecrets:
        - name: registry-auth
        containers:
        - name: java-demo
          image: 192.168.22.160/library/java-demo:v1
          resources: {}
          volumeMounts:
          - name: config
            mountPath: "/usr/local/tomcat/webapps/ROOT/WEBINF/classes/application.yml"
            subPath: application.yml
        volumes:
        - name: config
          configMap:
          - name: java-demo-config 
            items:
            - key: "application.yml"
              path: "application.yml"
  ```

- 部署数据库

  (0) 使用deployment部署一个mysql实例，service暴露访问

  (1) mysql实例访问

  (2) 导入项目sql文件

- 对外暴露应用

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-http
    annotations:
      kubernetes.io/ingress.class: nginx
  spec:
    rules:
    - host: one.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: java-demo
              port:
                number: 80
  ```

- 增加公网负载均衡器

  ```mermaid
  graph LR
  user --> B(nginx)
  B --> C(service nodeport)
  C --> D(ingress controller)
  D --> E(pod)
  ```
  
  ```yaml
  upstream ingress-controller {
    server 192.168.31.72:80;
    server 192.168.31.73:80;
  }
  server {
    listen 80;
    server_name _;
    location / {
      proxy_pass http://ingress-controller;
      proxy_set_header Host $Host; #把浏览器输入的域名传递到ingress controller(根据域名分流)
    }
  }
  ```

