# kubectl常用命令

## 查看输出详细信息格式

如果你想查看的是`yaml`格式的信息

```bash
kubectl get pods -n kube-system -oyaml
```

相反,如果你想要查看`json`格式的信息请使用`-ojson`
另外如果你需要监听该资源的信息请使用`-w`也就是我们的`watch`

## 简单的Kubectl调用原理

> 可以通过以下命令查看`kubectl`具体执行了那些操作

`kubectl`默认会读取`~/.kube/config`这个配置文件

```bash
[root@kube-master1 ~]# kubectl get pods -v 9
```
