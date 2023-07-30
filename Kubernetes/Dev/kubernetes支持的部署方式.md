## kubernetes支持的部署方式

- 通过`Json`格式部署资源
- 通过`Yaml`格式部署资源

### 输出yaml格式

```shell
[root@containerd-kube-master ~]# kubectl get deployment web -o yaml
```

## Yaml格式

1. 大小写敏感
2. 使用缩进标识关系层级
3. 缩进禁止使用tab,只能使用空格
4. 缩进要求同级相等
5. 所有数据左侧对齐
6. 可以使用`#`进行注释

```yaml
# Kind: 描述你要创建的kubernetes资源类型
kind: Deployment
# apiVersion: 表示你创建Kind所需要使用的API版本
apiVersion: apps/v1
# metadata: 当前这个Kind的描述信息,即deployment的描述信息
metadata:
  # name: 描述一下当前这个deployment的名字
  name: web
  # namespace: 描述当前这个deployment存在那个空间当中
  namespace: default
# 当前deployment的详细内容
spec:
  # selector: 定义一个当前deployment的标签
  selector: 
    # matchLabels: 定义多选标签
    matchLabels: 
      # 标签以 key value的形式出现
      app: nginx
      services_name: web-services
  # replicas: 定义当前Kind的Pod数量
  replicas: 2
  # template: 定义模板数据
  template:
  # metadata: 代表Pod的元数据
    metadata:
      # labels: 定义选择器标签,要与selector中的标签相互匹配
      labels: 
        app: nginx
        services_name: web-services
    # 定义Pod的行为规范
    spec:
      # containers: 定义容器,如果是是多个容器是containers
      containers:
      - name: web-services
        image: nginx:latest
        imagePullPolicy: Alaways
        ports: 
        - containerPort: 80
```

### 如何查看对应的资源类型应该使用那个API呢

- [官网地址](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#-strong-api-overview-strong-)

```shell
# 也可以通过如下命令去查看当前Kubernetes版本可用的API版本
[root@containerd-kube-master ~]# kubectl api-versions
```
## Json格式
```json
[root@master1 demo]# kubectl run nginx-deployment --image=nginx --port=80 --replicas=3 --dry-run -o json
{
    "kind": "Deployment",
    "apiVersion": "apps/v1beta1",
    "metadata": {
        "name": "nginx-deployment",
        "creationTimestamp": null,
        "labels": {
            "run": "nginx-deployment"
        }
    },
    "spec": {
        "replicas": 3,
        "selector": {
            "matchLabels": {
                "run": "nginx-deployment"
            }
        },
        "template": {
            "metadata": {
                "creationTimestamp": null,
                "labels": {
                    "run": "nginx-deployment"
                }
            },
            "spec": {
                "containers": [
                    {
                        "name": "nginx-deployment",
                        "image": "nginx",
                        "ports": [
                            {
                                "containerPort": 80
                            }
                        ],
                        "resources": {}
                    }
                ]
            }
        },
        "strategy": {}
    },
    "status": {}
}
```
## 资源创建流程
1. 主机发送`kubectl create`类似的操作命令发送到`ApiServer`
2. `ApiServer`从`controller Manager`中选择Pod控制器`Replicaset`
3. `ControllerManager`将控制器发送到`Scheduler`进行调度,然后发送到`kubelet`中进行容器的创建