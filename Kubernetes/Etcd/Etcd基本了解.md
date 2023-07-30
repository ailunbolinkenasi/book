# Etcd基本了解


## 什么是Etcd
`Etcd`是CoreOS基于`Raft`协议开发的分布式key-value的存储,可以用于服务发现、配置共享以及一致性保障。
- 基于Key、Value进行存储
- 监听机制
- key的过期续约机制,用于监控和服务发现
- 原子CAS和CAD,用于分布式锁和Leader选举


## 简单的查询一下Etcd中的数据
1. 这里的Etcd采用`docker`的方式进行外部Etcd的安装。

获取所有的`key`列表
- `–keys-only=true`: 表示只输出Key而不输出value
```bash
export ETCDCTL_API=3
etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/member-kube-master1.pem --key=/etc/ssl/etcd/ssl/member-kube-master1-key.pem get --keys-only --prefix /
```
## 监听数据

```bash
etcdctl  --cacert=/etc/ssl/etcd/ssl/ca.pem --cert=/etc/ssl/etcd/ssl/member-kube-master1.pem --key=/etc/ssl/etcd/ssl/member-kube-master1-key.pem watch  --prefix /registry/pods/default/1-9849f78df-789vk
```