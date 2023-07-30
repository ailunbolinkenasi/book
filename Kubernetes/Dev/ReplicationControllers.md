# ReplicationControllers
ReplicationController 确保在任何时候都有特定数量的 Pod 副本处于运行状态。 换句话说，ReplicationController 确保一个 Pod 或一组同类的 Pod 总是可用的。
- 现在目前已经推荐用`Deployment`和`ReplicaSet`来进行管理Pod

## ReplicationController 如何工作 
当 Pod 数量过多时，`ReplicationController` 会终止多余的 Pod。当 Pod 数量太少时，`ReplicationController` 将会启动新的 Pod。 与手动创建的 Pod 不同，由 `ReplicationController` 创建的 Pod 在失败、被删除或被终止时会被自动替换。 例如，在中断性维护（如内核升级）之后，你的 Pod 会在节点上重新创建。 因此，即使你的应用程序只需要一个 Pod，你也应该使用 `ReplicationController` 创建 Pod。 `ReplicationController` 类似于进程管理器，但是 `ReplicationController` 不是监控单个节点上的单个进程，而是监控跨多个节点的多个 Pod。
## 运行一个简单的Rc实例
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

# ReplicaSet
ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。


## ReplicaSet 的工作原理
ReplicaSet 是通过一组字段来定义的，包括一个用来识别可获得的 Pod 的集合的选择算符、一个用来标明应该维护的副本个数的数值、一个用来指定应该创建新 Pod 以满足副本个数条件时要使用的 Pod 模板等等。 每个 ReplicaSet 都通过根据需要创建和删除 Pod 以使得副本个数达到期望值， 进而实现其存在价值。当 ReplicaSet 需要创建新的 Pod 时，会使用所提供的 Pod 模板。

ReplicaSet 通过 Pod 上的 `metadata.ownerReferences` 字段连接到附属 Pod，该字段给出当前对象的属主资源。 ReplicaSet 所获得的 Pod 都在其 ownerReferences 字段中包含了属主 ReplicaSet 的标识信息。正是通过这一连接，ReplicaSet 知道它所维护的 Pod 集合的状态， 并据此计划其操作行为。

ReplicaSet 使用其选择算符来辨识要获得的 Pod 集合。如果某个 Pod 没有 OwnerReference 或者其 OwnerReference 不是一个控制器， 且其匹配到某 ReplicaSet 的选择算符，则该 Pod 立即被此 ReplicaSet 获得。

## 如何使用ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # 按你的实际情况修改副本数
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

