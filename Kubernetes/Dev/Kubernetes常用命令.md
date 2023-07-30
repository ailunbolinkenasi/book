### Kubernetes常用命令

### 创建一个Deployment资源

- 语法详细: `kubectl create 资源类型 资源名称 --image 镜像名称 --replicas 副本数量`

```shell
[root@containerd-kube-master ~]# kubectl create deployment test --image nginx:latest --replicas 3
```

### 查看Yaml当中的各项意思

- `kubectl explain 资源类型`

```shell
[root@containerd-kube-master ~]# kubectl explain deployment.spec.template
```

### 查看Kubernetes中事件状态

```shell
kubectl get events
```

### 通过kubelet 转发端口

```shell
[root@containerd-kube-master ~]#  kubectl port-forward web 8080:80
```

### 标签的选择
```shell
kubectl get pods --all-namespaces --field-selector spec.nodeName=master1
```

### 检测文件差异
```shell
kubectl diff -f init.yaml
```

### 开发一个kubectl插件
```shell
vim /usr/local/bin/kubectl-hello
kubectl hello
````	
