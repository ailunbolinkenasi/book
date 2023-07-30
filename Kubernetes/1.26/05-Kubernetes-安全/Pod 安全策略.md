# Pod 安全策略

Pod 安全策略使得对 Pod 创建和更新进行细粒度的权限控制成为可能。默认情况下，Kubernetes 允许创建一个有特权容器的 Pod，这些容器很可能会危机系统安全，而 Pod 安全策略（PSP）则通过确保请求者有权限按配置来创建 Pod，从而来保护集群免受特权 Pod 的影响。

> **注意：**PodSecurityPolicy 在 Kubernetes v1.21 版本中被弃用，**将在 v1.25 中删除**。 我们建议迁移到 [Pod 安全性准入](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-admission)， 或者第三方的准入插件。 若需了解迁移指南，可参阅[从 PodSecurityPolicy 迁移到内置的 PodSecurity 准入控制器](https://v1-24.docs.kubernetes.io/zh-cn/docs/tasks/configure-pod-container/migrate-from-psp/)。 关于弃用的更多信息，请查阅 [PodSecurityPolicy Deprecation: Past, Present, and Future](https://v1-24.docs.kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/)。

`PodSecurityPolicy` 是 Kubernetes API 对象，它能够控制 Pod 规范中与安全性相关的各个方面，`PodSecurityPolicy` 对象定义了一组 Pod 运行时必须遵循的条件及相关字段的默认值，只有 Pod 满足这些条件才会被系统接受，Pod 安全策略允许管理员控制如下方面：

## 什么是Pod安全策略

**Pod 安全策略（Pod Security Policy）** 是集群级别的资源，它能够控制 Pod 规约中与安全性相关的各个方面。 [PodSecurityPolicy](https://v1-24.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.24/#podsecuritypolicy-v1beta1-policy) 对象定义了一组 Pod 运行时必须遵循的条件及相关字段的默认值，只有 Pod 满足这些条件才会被系统接受。 Pod 安全策略允许管理员控制如下操作：

| 控制的角度                   | 字段名称                                                                                                                                                                         |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 运行特权容器                  | [`privileged`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#privileged)                                                                |
| 使用宿主名字空间                | [`hostPID`、`hostIPC`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#host-namespaces)                                                    |
| 使用宿主的网络和端口              | [`hostNetwork`、`hostPorts`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#host-namespaces)                                              |
| 控制卷类型的使用                | [`volumes`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#volumes-and-file-systems)                                                     |
| 使用宿主文件系统                | [`allowedHostPaths`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#volumes-and-file-systems)                                            |
| 允许使用特定的 FlexVolume 驱动   | [`allowedFlexVolumes`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#flexvolume-drivers)                                                |
| 分配拥有 Pod 卷的 FSGroup 账号  | [`fsGroup`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#volumes-and-file-systems)                                                     |
| 以只读方式访问根文件系统            | [`readOnlyRootFilesystem`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#volumes-and-file-systems)                                      |
| 设置容器的用户和组 ID            | [`runAsUser`、`runAsGroup`、`supplementalGroups`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#users-and-groups)                         |
| 限制 root 账号特权级提升         | [`allowPrivilegeEscalation`、`defaultAllowPrivilegeEscalation`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#privilege-escalation)      |
| Linux 权能字（Capabilities） | [`defaultAddCapabilities`、`requiredDropCapabilities`、`allowedCapabilities`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#capabilities) |
| 设置容器的 SELinux 上下文       | [`seLinux`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#selinux)                                                                      |
| 指定容器可以挂载的 proc 类型       | [`allowedProcMountTypes`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#allowedprocmounttypes)                                          |
| 指定容器使用的 AppArmor 模板     | [annotations](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#apparmor)                                                                   |
| 指定容器使用的 seccomp 模板      | [annotations](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#seccomp)                                                                    |
| 指定容器使用的 sysctl 模板       | [`forbiddenSysctls`、`allowedUnsafeSysctls`](https://v1-24.docs.kubernetes.io/zh-cn/docs/concepts/security/pod-security-policy/#sysctl)                                       |
