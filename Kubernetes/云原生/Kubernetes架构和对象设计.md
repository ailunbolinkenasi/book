# Kubernetes架构和对象设计

## API Server
`kube-apiserver`是`kubernetes`最重要的核心组件,主要提供以下功能: 
- 提供集群管理的REST API接口
    - 认证(Authentication)
    - 授权(Authorization)
    - 准入(Admission)
- 其他模块之间的数据交互和通信的枢纽(其他模块通过APIServer查询或者修改数据,只有APIServer才直接操作etcd数据库)
- APIServer 提供etcd数据缓存以减少集群对etcd的访问。

### APIServer 展开
下面看一下APIServer的具体组成

APIHandler(方法处理)->AuthN(认证服务)->Rate Limit(限流)->Auditing(记录访问日志)->AuthZ(K8s RBAC)->Aggregator(一个聚合器)-Mutating-Webhook->SchemaValidation->ValidatingWebhook

> 在此步骤所有都经历完成后数据才会进入etcd中进行存储。


## Controller Manager
- `Controller Manager`是集群的大脑,是确保整个集群动起来的关键
- 作用是确保Kubernetes遵循声明式的系统规范,确保系统的真实状态(Actual State)与用户定义的期望状态(Desired State)一致
- `Controller Manager`是多个控制器的组合,每个`Controller`事实上都是一个`control loop`,负责侦听其管控的对象,当对象发生变更时完成配置。
- `Controller`配置失败通常会触发自动重试,整个集群会在控制器的不断重试机制下确保最终一致性(Eventual Consistency)

> `Controller Manager`可以理解成为一个控制器的聚合,其中包含很多个控制器,例如: `Deployment Controller`、`ReplicaSet Controller`....
### 控制器的工作流程
1. 在`Controller`的接口当中会定义`informer`和`lister`去监听某个对象
2. 例如`informer`接口当中会监听`AddFunc` 、`DeleteFunc`、`UpdateFunc` 不同的Event
3. 然后取出这个对象的`name`和`namespaces`,然后放到队列中,可以开启多个gorountine来进行处理

> 简单的来讲就是生产者和消费者模型。 

but...如果你真的要用kubernetes无非就是
1. 搭建平台
2. 把应用程序运行在kubernetes平台中
> 往往越简单的东西,背后越复杂。

### 控制器的协同工作原理
1. 发起创建`Deployment`请求->`API Server`,这个时候APIServer会进行一系列的逻辑处理,例如: 鉴权、查看你是否有权限操作、Deployment创建是否合法等等,然后将请求存储到etcd当中并且转发给`Controller Manager`
2. `Controller Manager`会监听`API Server`,这个时候假设监听到的是一个创建`Deployment`的请求,则会把请求转发到`Deployment Controller`
3. `Deployment Controller`接受到请求后创建`ReplicaSet`,然后`ReplicaSet Controller`会根据`yaml`当中定义的`template`模板来进行创建Pod,然后返回给`API Server`
4. 在创建之初的Pod属性中`nodeName`为空,也就是没有被调度过的,这个时候调度器就会对它进行调度,调度去watchPod对象,然后分析那个节点最适合这个Pod,然后将节点的名字通过类似于bind的这种方法写入到`nodeName`当中。
5. 然后该节点的kubelet会进行一系列的判断,然后进入Create Pod的流程,然后进行一系列的`CNI`和`CSI`的过程。

> 为什么不建议去创建孤独的Pod: 如果你创建一个没有控制器控制的Pod,那么当你删除该Pod后,则该Pod将会被`ReplicaSet Controller`进行监听删除。
> 由于`ReplicaSet Controller`会根据用户期望的Pod进行创建或者删除,这个时候要保证状态一致


## Kubelet
kubernetes的初始化系统(init system)
- 从不同资源获取Pod清单,并且按照需求启动停止Pod的核心组件。
    - Kubelet将运行时,网络和存储抽象成了`CRI`、`CNI`、`CSI`
- 负责汇报当前节点的资源信息和健康状态
- 负责Pod的健康检查和状态汇报


## Kube—Porxy
- 监控集群中用户发布的服务,并完成负载均衡配置。
- 每个节点上的`kube-proxy`都会配置相同的负载均衡策略,使得整个集群的服务发现建立在分布式负载均衡器之上,服务调用无需经过额外的网络跳转。
- 负载均衡基于不同的插件实现
    - `userspce`
    - `iptables`
    - `ipvs`

    
## 推荐的Add-ons
- kube-dns: 负责为整个集群提供DNS服务
- IngressController: 为服务提供外网入口
- MetricsServer: 提供资源监控
- Dashboard: 提供GUI界面
- Fluentd-Elasticsearch: 提供集群日志采集,查询和存储相关。


## 一些常用的属性介绍
### Namespace
`Namespace`是对一组资源和对象的抽象集合,比如可以用将来系统内部的对象划分不同的项目组。
常见的`pods` 、`service`等都是属于某一个`Namespace`的。
> - `node`不属于任何一个`Namespace`。


### Finalizer和ResourceVersion
`Finalizer`本质上是一个资源锁,kubernetes在接受某对象删除请求时,会检查`Finalizer`是否为空,如果为空的话只对其做逻辑删除。只会更新对象中的`metadata.deletionTimestamp`字段

---

`ResourceVersion`可以被看作一种乐观锁,每个对象在任意时刻都有`ResourceVersion`当kubernetes对象被客户端读取以后,`ResourceVersion信息也一并被读取。此机制确保了分布式系统中任意多线程能够无锁并发访问对象。

### Label
- Label是识别kubernetes对象的标签,通过`key/value`的方式附加到对象当中话
- `key`最长不能超过63字节,`value`可以是空值,但是不可以超过253字节
- Label不提供唯一性,并且实际上经常是很多对象(如Pod)都使用相同的`label`来标示具体的应用
- Label定义好以后其他对象可以使用`Label Selector`来选择一组相同的对象
- Label Selector支持以下几种方式[参考链接]()

### Annotations
- Annotations是`key`和`value`形式附加于对象的注解。
- 不同于`labels`用于标志和选择对象,Annotations则用来记录一些附加信息,用来辅助应用部署和安装。

## 常用kubernetes对象分组

### 策略管理
- `rbac/v1`: Role RoleBinding ClusterRole ClusterRoleBinding
- `extensions/v1beta1`: PodSecurityPolicy
- `networkPolicy/v1`: NetworkPolicy

### 自动化
- `autoscaling/v2beta1`: HorizontalPodAutoscaler
- `policy/v1beta1`: PodDisruptionBudget
- `setting.k8s.io/v1alpha1`： PodPreset

### 服务发布
- `core/v1`: service
- `extensions/v1beta1`: ingress


### 应用管理
- `apps/v1`: StatefulSet Deployment DaemonSet ReplicaSet
- `batch/v1`: Job
- `batch/v2alpha1`: CronJob

### 核心对象
- `core/v1`: Node Namespace ResourceQuota Event PV PVC Pod ConfigMap Secret ServiceAccount Service Endpoints
- `storage/v1`: StroageClass


## 什么是Pod
- Pod是一组紧密关联的容器集合,它们共享PID、IPC、Network和UTS namespace,是kubernetes调度的最小单位。
- Pod的设计理念是支持多个容器在一个Pod中共享网络和文件系统,可以通过进程间通信这种简单高效地方式组合完成服务
- 同一个Pod中不同容器可以共享资源:
    - 共享网络Namespace
    - 可通过挂载存储卷共享存储
    - 共享Security Context

## 健康检查
kubernetes作为一个面向应用的集群管理工具,需要确保容器在部署后确实处于正常的工作状态。
kubernetes所拥有的探针类型如下
- LivenessProbe: 探测应用是否处于健康状态,如果不正常则删除并且重启容器
- ReadinessProbe: 探测应用是否就绪并且处于正常服务状态,如果不正常则不会接受来自`kubernetes service`的流量
- startupProbe: 探测应用是否启动完成


探测方式
1. Exec
2. TCP Socket
3. HTTP


## ConfigMap
- ConfigMap用来将非机密性的数据保存到键值当中
- 使用的时候可用作Pod的环境变量、命令行参数或者存储卷的配置文件
- ConfigMap将环境配置信息和容器镜像解耦


## Secret密钥对象
- Secret是用来保存和传递密码、密钥、认证凭证这些敏感的对象
- 使用Secret的好处是可以避免把敏感信息明文写在配置文件当中
- kubernetes集群中配置和使用服务不可避免要用到各种敏感信息登录,例如访问存储桶的用户名和密码

## UserAccount & ServiceAccount
用户账户为人提供账户标识,而服务账户则为计算机进程和kubernetes集群中运行的Pod提供服务标识
- 用户账户和服务账户一个区别是作用范围:
    -  用户账户对应的是人的身份,人的身份与服务的`namespace`无关,所以用户账户是跨`namespace`的
    -  而服务账户对应的是一个运行中程序的身份,与特定的`namespace`是相关的 