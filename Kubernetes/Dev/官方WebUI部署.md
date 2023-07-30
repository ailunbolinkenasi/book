## 官方WebUI部署
- [Dashboard](https://github.com/kubernetes/dashboard)

### 安装部署

```shell
[root@containerd-kube-master ~]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.1/aio/deploy/recommended.yaml
```

查看Dashboard的`Pod`是否运行

```shell
[root@containerd-kube-master ~]# kubectl get pods --all-namespaces
NAMESPACE              NAME                                        READY   STATUS    RESTARTS   AGE
kubernetes-dashboard   dashboard-metrics-scraper-8c47d4b5d-t92v9   1/1     Running   0          26m
kubernetes-dashboard   kubernetes-dashboard-6c75475678-zlqn7       1/1     Running   0          26m
```

修改Dashboard的`Services`



```shell
[root@containerd-kube-master ~]# kubectl edit services kubernetes-dashboard  -n kubernetes-dashboard
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kubernetes-dashboard"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: "2022-09-06T02:58:06Z"
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "5359"
  uid: 31019fac-9b16-46e5-9172-2cc3f63f2c86
spec:
  clusterIP: 10.101.114.92
  clusterIPs:
  - 10.101.114.92
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
    nodePort: 30000 # 添加一行nodePort
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP # 修改为NodePort
status:
  loadBalancer: {}
```

改完以后一定要看一眼生效无生效啊

- 查看到`TYPE`如果是`NodePort`
- 查看`PORT`对应443映射到了30000端口

```shell
[root@containerd-kube-master ~]# kubectl get services --all-namespaces
NAMESPACE              NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
default                kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP                  98m
kube-system            kube-dns                    ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   98m
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.103.193.200   <none>        8000/TCP                 43m
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.101.114.92    <none>        443:30000/TCP            43m
```

### 访问页面

- 访问https://IP:PORT 例如: https://10.1.6.45:30000/#/login

![img](https://cdn.nlark.com/yuque/0/2022/png/1142005/1662436310857-defad447-c883-48c0-9e60-916e3452e4f1.png)

新版本中默认不存储`secret`的加密数据了,所以如果你想使用Token进行登录的话,你需要新建一个用户并且绑定角色

- 新建一个`user.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
[root@containerd-kube-master ~]# kubectl apply -f user.yaml
```

基于`admin-user`创建一个token,然后把输出的token粘贴到Dashboard

```shell
[root@containerd-kube-master ~]# kubectl create token admin-user -n kubernetes-dashboard
```

官方的WebUi到此结束了,WebUi操作起kubernetes非常的方便

## 推荐几个其他的WebUi
- [KubeGems](https://www.kubegems.io/)
- [KubeSphere](https://kubesphere.io/zh/)
- [Kuboard](https://kuboard.cn/)