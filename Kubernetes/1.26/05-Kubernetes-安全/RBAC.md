# RBAC

基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对计算机或网络资源的访问的方法。

RBAC 鉴权机制使用 `rbac.authorization.k8s.io` [API 组](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning)来驱动鉴权决定， 允许你通过 Kubernetes API 动态配置策略。

要启用 RBAC，在启动 [API 服务器](https://kubernetes.io/zh-cn/docs/concepts/overview/components/#kube-apiserver)时将 `--authorization-mode` 参数设置为一个逗号分隔的列表并确保其中包含 `RBAC`。

管理员可以通过 Kubernetes API 动态配置策略来启用`RBAC`，需要在 kube-apiserver 中添加参数`--authorization-mode=RBAC`，如果使用的 kubeadm 安装的集群那么是默认开启了 `RBAC` 的，可以通过查看 Master 节点上 apiserver 的静态 Pod 定义文件：

```yaml
cat /etc/kubernetes/manifests/kube-apiserver.yaml 
...
    - --authorization-mode=Node,RBAC
...
```

## 先了解Kubernetes的API对象

RBAC API 声明了四种 Kubernetes 对象：**Role**、**ClusterRole**、**RoleBinding** 和 **ClusterRoleBinding**。你可以像使用其他 Kubernetes 对象一样，通过类似 `kubectl` 这类工具[描述对象](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/kubernetes-objects/#understanding-kubernetes-objects), 或修补对象。

> 试问: kubectl 如何将我们的Yaml文件转化为Kubernetes的API对象呢？

答案: Kubernetes API 是一个以 JSON 为主要序列化方式的 HTTP 服务，除此之外也支持 Protocol Buffers 序列化方式，主要用于集群内部组件间的通信。为了可扩展性，Kubernetes 在不同的 API 路径（比如`/api/v1` 或者 `/apis/batch`）下面支持了多个 API 版本，不同的 API 版本意味着不同级别的稳定性和支持,也就是我们常说的声明式API

在 Kubernetes 集群中，一个 API 对象在 Etcd 里的完整资源路径，是由：`Group（API 组）`、`Version（API 版本）`和 `Resource（API 资源类型）`三个部分组成的。通过这样的结构，整个 Kubernetes 里的所有 API 对象，实际上就可以用如下的树形结构表示出来：

```bash
[root@Online-Beijing-master1 ~]# kubectl get --raw /
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    ...
  ]
}
```

比如我们来查看批处理这个操作，在我们当前这个版本中存在两个版本的操作：`/apis/apps/v1`暴露了可以查询和操作的不同实体集合，同样我们还是可以通过 kubectl 来查询对应对象下面的数据：

- create：创建新的资源对象
- delete：删除指定的资源对象
- get：获取指定的资源对象
- list：列出所有符合条件的资源对象
- patch：局部更新指定的资源对象
- update：全量更新指定的资源对象
- watch：监听指定的资源对象，当资源对象发生变化时进行通知

```bash
kubectl get --raw /apis/apps/v1 | python -m json.tool
```

```json
{
    "apiVersion": "v1",
    "groupVersion": "apps/v1",
    "kind": "APIResourceList",
    "resources": [
        {
            "kind": "ControllerRevision",
            "name": "controllerrevisions",
            "namespaced": true,
            "singularName": "",
            "storageVersionHash": "85nkx63pcBU=",
            "verbs": [
                "create",
                "delete",
                "deletecollection",
                "get",
                "list",
                "patch",
                "update",
                "watch"
            ]
        },
        {
            "categories": [
                "all"
            ],
            "kind": "DaemonSet",
            "name": "daemonsets",
            "namespaced": true,
            "shortNames": [
                "ds"
            ],
            "singularName": "",
            "storageVersionHash": "dd7pWHUlMKQ=",
            "verbs": [
                "create",
                "delete",
                "deletecollection",
                "get",
                "list",
                "patch",
                "update",
                "watch"
            ]
        },
        {
            "kind": "DaemonSet",
            "name": "daemonsets/status",
            "namespaced": true,
            "singularName": "",
            "verbs": [
                "get",
                "patch",
                "update"
            ]
        },
        {
            "categories": [
                "all"
            ],
            "kind": "Deployment",
            "name": "deployments",
            "namespaced": true,
            "shortNames": [
                "deploy"
            ],
            "singularName": "",
            "storageVersionHash": "8aSe+NMegvE=",
            "verbs": [
                "create",
                "delete",
                "deletecollection",
                "get",
                "list",
                "patch",
                "update",
                "watch"
            ]
        },
        {
            "group": "autoscaling",
            "kind": "Scale",
            "name": "deployments/scale",
            "namespaced": true,
            "singularName": "",
            "verbs": [
                "get",
                "patch",
                "update"
            ],
            "version": "v1"
        },
        {
            "kind": "Deployment",
            "name": "deployments/status",
            "namespaced": true,
            "singularName": "",
            "verbs": [
                "get",
                "patch",
                "update"
            ]
        }
```

> 简单的说一下这个`deletecollection`: 表示删除指定资源类型下的所有资源对象。
>
> - 例如，我们可以使用 `kubectl delete pods --all` 命令来删除所有的 Pod 资源对象，这个命令实际上会发送一个 `deletecollection` 请求给 Kubernetes API Server，让它删除所有的 Pod 资源对象。需要注意的是，`deletecollection` 操作可能会对系统造成较大的影响，因此需要谨慎使用。

## RBAC

其实Kubernete中所有的资源对象都是模型化的API对象,允许对其进行CRUD操作(Create、Read、Update、Delete)等增删改查的操作.

- Pods
- Deployments
- ConfigMaps
- Nodes
- ...

对于上面的一个资源我们可能存在的操作有

- create
- get 
- delete 
- list
- update
- edit
- watch
- exec
- patch

在更上层，这些资源和 API Group 进行关联，比如 Pods 属于 Core API Group，而 Deployements 属于 apps API Group。

## RBAC中的简单地概念

- `Rule`：规则，规则是一组属于不同 API Group 资源上的一组操作的集合
- `Role` 和 `ClusterRole`：角色和集群角色，这两个对象都包含上面的 Rules 元素，二者的区别在于，在 Role 中，定义的规则只适用于`单个命名空间`，也就是和 namespace 关联的，而ClusterRole`是集群范围内的`，因此定义的规则不受命名空间的约束。另外 Role 和 ClusterRole 在Kubernetes 中都被定义为集群内部的 API 资源，和我们前面学习过的 Pod、Deployment 这些对象类似，都是我们集群的资源对象，所以同样的可以使用 YAML 文件来描述，用 kubectl 工具来管理
- `Subject`：主题，对应集群中尝试操作的对象，集群中定义了3种类型的主题资源：
  - `User Account`：用户，这是有外部独立服务进行管理的，管理员进行私钥的分配，用户可以使用 KeyStone 或者 Goolge 帐号，甚至一个用户名和密码的文件列表也可以。对于用户的管理集群内部没有一个关联的资源对象，所以用户不能通过集群内部的 API 来进行管理
  - `Group`：组，这是用来关联多个账户的，集群中有一些默认创建的组，比如 cluster-admin
  - `Service Account`：服务帐号，通过 Kubernetes API 来管理的一些用户帐号，和 namespace 进行关联的，适用于集群内部运行的应用程序，需要通过 API 来完成权限认证，所以在集群内部进行权限操作，我们都需要使用到 ServiceAccount，这也是我们这节课的重点
- `RoleBinding` 和 `ClusterRoleBinding`：角色绑定和集群角色绑定，简单来说就是把声明的 Subject 和我们的 Role 进行绑定的过程（给某个用户绑定上操作的权限），二者的区别也是作用范围的区别：RoleBinding 只会影响到当前 namespace 下面的资源操作权限，而 ClusterRoleBinding 会影响到所有的 namespace。

## Kubernetes中如何使用RBAC

写几个简单地小例子来看一下如何使用

### 允许某个用户只能访问某个命名空间

假设`outsystem`这个用户只能访问`out-apps`这个名称空间下的信息

假设我们的这个`outsystem`用户是一个外部来宾组

```tex
username： outsystem
group: out-group
```

1. 创建用户凭证

我们前面已经提到过，Kubernetes 没有 `User Account` 的 API 对象，不过要创建一个用户帐号的话也是挺简单的，利用管理员分配给你的一个私钥就可以创建了，这个我们可以参考官方文档中的方法，这里我们来使用 `OpenSSL` 证书来创建一个 User，当然我们也可以使用更简单的 `cfssl`工具来创建：

给用户`outsystem`创建一个私钥,名字为`outsystem.key`

```bash
openssl genrsa -out outsystem.key 2048
```

使用我们刚刚创建的私钥创建一个证书签名请求文件：`outsystem.csr`，要注意需要确保在`-subj`参数中指定用户名和组(CN表示用户名，O表示组)：

```bash
openssl req -new -key outsystem.key -out outsystem.csr -subj "/CN=outsystem/O=out-group"
```

然后找到我们的 Kubernetes 集群的 `CA` 证书，我使用的是 kubeadm 安装的集群，CA 相关证书位于 `/etc/kubernetes/pki/` 目录下面，如果你是二进制方式搭建的，你应该在最开始搭建集群的时候就已经指定好了 CA 的目录，我们会利用该目录下面的 `ca.crt` 和 `ca.key`两个文件来批准上面的证书请求。生成最终的证书文件，我们这里设置证书的有效期为 365 天：

先查看我们是否有`ca.crt`和`ca.key`这两个文件

```bash
[root@Online-Beijing-master1 pki]# tree
.
├── apiserver.crt
├── apiserver.key
├── apiserver-kubelet-client.crt
├── apiserver-kubelet-client.key
├── ca.crt
├── ca.key
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key
└── sa.pub
```

如果有再进行证书签入申请

```bash
[root@Online-Beijing-master1 ~]# openssl x509 -req -in outsystem.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out outsystem.crt -days 365
Signature ok
subject=CN = outsystem, O = out-group
Getting CA Private Key
```

正常情况下，我们的目录下应该会有如下三个文件

```bash
[root@Online-Beijing-master1 ~]# ls
outsystem.crt     outsystem.csr     outsystem.key
```

现在我们可以使用刚刚创建的证书文件和私钥文件在集群中创建新的凭证和上下文(Context):

- `--embed-certs=true`： 如果你想要把证书嵌入到以后的`yaml`文件当中可以加入此选项

```bash
kubectl config set-credentials outsystem --client-certificate=outsystem.crt --client-key=outsystem.key
```

我们可以看到一个用户 `outsystem` 创建了，然后为这个用户设置 Context并且允许它访问特定的名称空间：`out-apps`

```bash
kubectl config set-context outsystem-context --cluster=kubernetes --namespace=out-apps --user=outsystem
# 你可以查看当前的配置信息
  kubectl config view
# 切换Context
kubectl config use-context outsystem-context
```

用户创建成功以后，我们可以使用当前`outsystem`这个用户来查看一下当前`out-apps`命名空间都有哪些Pod

```yaml
[root@Online-Beijing-master1 ~]# kubectl get pods --context=outsystem-context
Error from server (Forbidden): pods is forbidden: User "outsystem" cannot list resource "pods" in API group "" in the namespace "out-apps"
```

创建完成用户以后,我们需要创建一个角色用来绑定当前的操作权限有哪些

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: outsystem-role
  namespace: out-apps
rules:
# 针对那个APIGroup,默认不写就是core
- apiGroups: ["","apps"]
# 针对当前API组下面的那个资源进行操作
  resources: ["deployments","pods"]
# 针对当前资源做什么操作
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Role 创建完成了，但是很明显现在我们这个 `Role` 和我们的用户 `outsystem` 没有任何关系这里就需要创建一个 `RoleBinding` 对象

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: outsystem-RoleBinding
  namespace: out-apps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: outsystem-role
subjects:
- kind: User
  name: outsystem
  apiGroup: ""
```

```bash
kubectl apply -f outsystem.yaml
```

然后通过使用`outsystem`用户查看相应的`deployment`、`Pods`即可

```bash
# 切换回默认的管理员
kubectl config use-context kubernetes-admin@kubernetes
# 切换到outsystem-context
kubectl config use-context outsystem-context
# 查看相应的Deployments
[root@Online-Beijing-master1 ~]# kubectl get pods
NAME                 READY   STATUS    RESTARTS   AGE
v1-65b97565f-jp584   1/1     Running   0          4m20s
[root@Online-Beijing-master1 ~]# kubectl get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
v1     1/1     1            1           4m23s
```

### 只能访问某个命名空间的ServiceAccount

上面我们创建了一个只能访问某个命名空间下面的**普通用户**，我们前面也提到过 `subjects` 下面还有一种类型的主题资源：`ServiceAccount`，现在我们来创建一个集群内部的用户只能操作 kube-system 这个命名空间下面的 pods 和 deployments，首先来创建一个 `ServiceAccount` 对象

```bash
kubectl create sa out-system -n kube-system
```

当然你也可以选择通过`yaml`来进行创建

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2023-02-21T00:39:11Z"
  name: out-system
  namespace: kube-system
  resourceVersion: "2556320"
  uid: 8d4151d7-668f-4e64-8865-6208502ed135
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: out-system-role
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["pods","namespaces"]
  verbs: ["get","list","watch"]
- apiGroups: ["apps"]
  resources: ["deployments","replicasets"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: out-system-rolebinding
  namespace: kube-system
roleRef: 
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: out-system-role
subjects:
- kind: ServiceAccount
  name: out-system
  namespace: kube-system
```

如果需要验证的话可以使用官方的`dashboard`使用我们刚刚新创建的`ServiceAccount`进行创建`token`进行登录

### 可以全局访问的 ServiceAccount

我们现在创建一个新的 ServiceAccount，需要他操作的权限作用于所有的 namespace，这个时候我们就需要使用到 `ClusterRole` 和 `ClusterRoleBinding` 这两种资源对象了。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2023-02-21T00:39:11Z"
  name: out-system
  resourceVersion: "2556320"
  uid: 8d4151d7-668f-4e64-8865-6208502ed135
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: out-system-clusterole
rules:
- apiGroups: [""]
  resources: ["pods","namespaces"]
  verbs: ["get","list","watch"]
- apiGroups: ["apps"]
  resources: ["deployments","replicasets"]
  verbs: ["get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: out-system-rolebinding
  namespace: kube-system
roleRef: 
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: out-system-clusterole
subjects:
- kind: ServiceAccount
  name: out-system
  namespace: default
```
