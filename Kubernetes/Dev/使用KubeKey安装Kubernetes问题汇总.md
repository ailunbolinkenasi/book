# 使用KubeKey安装Kubernetes问题汇总
- Kubernetes Verson: 1.17.9
- Docker Version: 20.10.8

> 由于kubelet的默认存储目录存放于`/var/lib/kubelet`,可能由于/目录空间不足从而导致集群无法使用的问题.

## 修改kubelet存储目录

如果你的kubelet是从`/etc/sysconfig/kubelet`进行加载的,那么磨人的配置应该在`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`

- 在`KUBELET_EXTRA_ARGS`后添加：` --root-dir=/data/k8s/kubelet`

```shell
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generate at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
Environment="KUBELET_EXTRA_ARGS=--node-ip=10.1.6.71 --hostname-override=master1 --root-dir=/data/k8s/kubelet"
ExecStart=
ExecStart=/usr/local/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

你需要这样添加

```shell
Environment="KUBELET_EXTRA_ARGS=$KUBELET_EXTRA_ARGS --root-dir=/data/k8s/kubelet"
```

完成后重启kubelet

```shell
systemctl daemon-reload
systemctl restart kubelet
```
取消挂载当前kubelet的目录数据
```shell
umount $(df -HT | grep '/var/lib/kubelet/pods' | awk '{print $7}')
```

### 通过链接方式修改
1. 请先备份`kubelet`数据
2. 将`/var/lib/kubelet`拷贝到`/data/k8s/kubelet`
3. 将`/var/lib/kubelet/`重命名,然后链接`/data/k8s/kubelet`

```shell
ln -s /data/k8s/kubelet /var/lib/kubelet
systemctl daemon-reload
systemctl restart kubelet
```

## 修改Etcd存储目录
```shell
systemctl stop etcd  # 首先停止etcd服务
mv /var/lib/etcd/*   /data/k8s/etcd/ # 将原有的etcd数据移动
```
修改`etcd`服务内容
```shell
vim /usr/local/bin/etcd
```
```shell
#!/bin/bash
/usr/bin/docker run \
  --restart=on-failure:5 \
  --env-file=/etc/etcd.env \
  --net=host \
  -v /etc/ssl/certs:/etc/ssl/certs:ro \
  -v /etc/ssl/etcd/ssl:/etc/ssl/etcd/ssl:ro \
  -v /data/k8s/etcd:/var/lib/etcd:rw \  # 修改你的etcd目录
  --memory=512M \
  --blkio-weight=1000 \
  --name=etcd1 \
  kubesphere/etcd:v3.4.13 \
  /usr/local/bin/etcd \
  "$@
  ```
  ```shell
docker rm -f fec80e864904 # 删除之前的etcd容器
systemctl daemon-reload
systemctl start etcd # 启动etcd
```

## 如果Prometheus监控数据失败
### kube-etcd-client-certs not found
```shell
# 如果没有Etcd证书,请创建一个空的secret
kubectl -n kubesphere-monitoring-system create secret generic kube-etcd-client-certs
```
如果你已经创建了`etcd`证书,请参考如下配置进行生成
```shell
kubectl -n kubesphere-monitoring-system create secret generic kube-etcd-client-certs  \
--from-file=etcd-client-ca.crt=/etc/kubernetes/pki/etcd/ca.crt  \
--from-file=etcd-client.crt=/etc/kubernetes/pki/etcd/healthcheck-client.crt  \
--from-file=etcd-client.key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

## 指定你需要使用的容器运行时
