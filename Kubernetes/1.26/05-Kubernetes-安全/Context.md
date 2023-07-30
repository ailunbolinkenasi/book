## Security Context

安全上下文（Security Context）定义 Pod 或 Container 的特权与访问控制设置。 安全上下文包括但不限于：

- 自主访问控制（Discretionary Access Control）： 基于[用户 ID（UID）和组 ID（GID）](https://wiki.archlinux.org/index.php/users_and_groups) 来判定对对象（例如文件）的访问权限。

- [安全性增强的 Linux（SELinux）](https://zh.wikipedia.org/wiki/安全增强式Linux)： 为对象赋予安全性标签。

- 以特权模式或者非特权模式运行。

- [Linux 权能](https://linux-audit.com/linux-capabilities-hardening-linux-binaries-by-removing-setuid/): 为进程赋予 root 用户的部分特权而非全部特权。

- [AppArmor](https://kubernetes.io/zh-cn/docs/tutorials/security/apparmor/)：使用程序配置来限制个别程序的权能。

- [Seccomp](https://kubernetes.io/zh-cn/docs/tutorials/security/seccomp/)：过滤进程的系统调用。

- `allowPrivilegeEscalation`：控制进程是否可以获得超出其父进程的特权。 此布尔值直接控制是否为容器进程设置 [`no_new_privs`](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt)标志。 当容器满足一下条件之一时，`allowPrivilegeEscalation` 总是为 true：
  
  - 以特权模式运行，或者
  - 具有 `CAP_SYS_ADMIN` 权能

- readOnlyRootFilesystem：以只读方式加载容器的根文件系统。

以上条目不是安全上下文设置的完整列表 -- 请参阅 [SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/#securitycontext-v1-core) 了解其完整列表。

> 主要是来限制容器非法操作宿主节点的系统级别的内容！

## 常用的字段内容

| 字段名称         | 类型        |                                                                                                                                                                                                     |
| ------------ | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| runAsNonRoot | *boolean* | 指示容器必须以非根用户身份运行。如果为 true，Kubelet 将在运行时验证镜像以确保它不会以 UID 0（root）运行，如果是则无法启动容器。如果未设置或为假，则不会执行此类验证。也可以在 PodSecurityContext 中设置。如果同时在 SecurityContext 和 PodSecurityContext 中设置，则 SecurityContext 中指定的值优先。 |
| runAsUser    | *integer* | 运行容器进程入口点的 UID。如果未指定，则默认为图像元数据中指定的用户。也可以在 PodSecurityContext 中设置。如果同时在 SecurityContext 和 PodSecurityContext 中设置，则 SecurityContext 中指定的值优先。请注意，当 spec.os.name 为 windows 时，不能设置此字段。                   |
| runAsGroup   | *integer* | 运行容器进程入口点的 GID。如果未设置，则使用运行时默认值。也可以在 PodSecurityContext 中设置。如果同时在 SecurityContext 和 PodSecurityContext 中设置，则 SecurityContext 中指定的值优先。请注意，当 spec.os.name 为 windows 时，不能设置此字段。                         |

## 为Pod设置SecurityContext

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false
```

通过上面的配置我们可以看到为Pod容器进行设置了`securityContext`

- `runAsUser`: 表示当前Pod以UID为1000的用户运行
- `runAsGroup`: 表示当前Pod中所有的容器进程都以GID3000的身份运行， 如果忽略此字段，则容器的主组 ID 将是 root（0）
- `fsGroup`: 容器中所有进程也会是附组 ID 2000 的一部分。 卷 `/data/demo` 及在该卷中创建的任何文件的属主都会是组 ID 2000。
- `allowPrivilegeEscalation`： 控制进程是否可以获得超出其父进程的特权

运行当前这个`yaml`文件

```bash
kubectl apply -f security-context.yaml
```

然后进入Pod内部

```bash
kubectl exec -it security-context-demo -- sh
# 查看当前的进程 输出显示进程以用户 1000 运行，即 runAsUser 所设置的值
/ $ ps
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 1h
    6 1000      0:00 sh
   13 1000      0:00 ps
# 查看挂载的Volume,输出显示 /data/demo 目录的组 ID 为 2000，即 fsGroup 的设置值：
/ $ cd /data/
/data $ ls -alh
total 0
drwxr-xr-x    3 root     root          18 Feb 21 02:23 .
drwxr-xr-x    1 root     root          63 Feb 21 02:23 ..
drwxrwsrwx    2 root     2000           6 Feb 21 02:23 demo
```

## 为容器设置SecurityContext

若要为 Container 设置安全性配置，可以在 Container 清单中包含 `securityContext` 字段。`securityContext` 字段的取值是一个 [SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/#securitycontext-v1-core) 对象。你为 Container 设置的安全性配置仅适用于该容器本身，并且所指定的设置在与 Pod 层面设置的内容发生重叠时，会重写 Pod 层面的设置。Container 层面的设置不会影响到 Pod 的卷。

下面是一个 Pod 的配置文件，其中包含一个 Container。Pod 和 Container 都有 `securityContext` 字段：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: containers-demo
    image: busybox:latest
    command: ["sh","-c","sleep 60m"]
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false
```

## Linux Capabilities

要了解 `Linux Capabilities`，这就得从 Linux 的权限控制发展来说明。在 Linux 2.2 版本之前，当内核对进程进行权限验证的时候，Linux 将进程划分为两类：特权进程（UID=0，也就是超级用户）和非特权进程（UID!=0），特权进程拥有所有的内核权限，而非特权进程则根据进程凭证（effective UID, effective GID，supplementary group 等）进行权限检查。

比如我们以常用的 `passwd` 命令为例，修改用户密码需要具有 root 权限，而普通用户是没有这个权限的。但是实际上普通用户又可以修改自己的密码，这是怎么回事呢？在 Linux 的权限控制机制中，有一类比较特殊的权限设置，比如 SUID(Set User ID on execution)，允许用户以可执行文件的 owner 的权限来运行可执行文件。因为程序文件 `/bin/passwd` 被设置了 `SUID` 标识，所以普通用户在执行 passwd 命令时，进程是以 passwd 的所有者，也就是 root 用户的身份运行，从而就可以修改密码了。

但是使用 `SUID` 却带来了新的安全隐患，当我们运行设置了 `SUID` 的命令时，通常只是需要很小一部分的特权，但是 `SUID` 却给了它 root 具有的全部权限，一旦 被设置了 `SUID` 的命令出现漏洞，是不是就很容易被利用了。

为此 Linux 引入了 `Capabilities` 机制来对 root 权限进行了更加细粒度的控制，实现按需进行授权，这样就大大减小了系统的安全隐患。

### 什么是 Capabilities

从内核 2.2 开始，Linux 将传统上与超级用户 root 关联的特权划分为不同的单元，称为 `capabilites`。`Capabilites` 每个单元都可以独立启用和禁用。这样当系统在作权限检查的时候就变成了：**在执行特权操作时，如果进程的有效身份不是 root，就去检查是否具有该特权操作所对应的 capabilites，并以此决定是否可以进行该特权操作**。比如如果我们要设置系统时间，就得具有 `CAP_SYS_TIME` 这个 capabilites。

- 详细内容请参考: [文档](    )

### 如何使用 Capabilities

我们可以通过 `getcap` 和 `setcap` 两条命令来分别查看和设置程序文件的 `capabilities` 属性。比如当前我们是`zuiapp` 这个用户，使用 `getcap` 命令查看 `ping` 命令目前具有的 `capabilities`：

```bash
[root@hadoop37 ~]# getcap /bin/ping
/bin/ping = cap_net_admin,cap_net_raw+p
```

我们可以看到具有 `cap_net_admin` 这个属性，所以我们现在可以执行 `ping` 命令

但是如果我们把命令的 `capabilities` 属性移除掉：

```bash
sudo setcap cap_net_admin,cap_net_raw-p /bin/ping
getcap /bin/ping
/bin/ping =
```

这个时候我们执行 `ping` 命令可以发现已经没有权限了

```bash
ping www.baidu.com
ping: socket: Operation not permitted
```

因为 ping 命令在执行时需要访问网络，所需的 `capabilities` 为 `cap_net_admin` 和 `cap_net_raw`，所以我们可以通过 `setcap` 命令可来添加它们：

```bash
sudo setcap cap_net_admin,cap_net_raw+p /bin/ping
getcap /bin/ping
/bin/ping = cap_net_admin,cap_net_raw+p
```

### Kubernetes 配置 Capabilities

上面我介绍了在 Docker 容器下如何来配置 `Capabilities`，在 Kubernetes 中也可以很方便的来定义，我们只需要添加到 Pod 定义的 `spec.containers.securityContext.capabilities`中即可，也可以进行 `add` 和 `drop` 配置，同样上面的示例，我们要给 busybox 容器添加 `NET_ADMIN` 这个 `Capabilities`，对应的 YAML 文件可以这样定义：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: busybox:latest
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
```
