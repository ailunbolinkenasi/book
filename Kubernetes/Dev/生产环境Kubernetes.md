# 生产环境Kubernetes


## Kubernetes基础环境安装脚本
```yaml
---
- hosts: kubernetes
  gather_facts: true # 是否进行收集主机信息
  remote_user: root
  tasks:
    - name: Set Authorized_key In Linux # 批量拷贝主机密钥
      authorized_key:
        user: root
        key: "{{ lookup('file', '/root/.ssh/id_rsa.pub') }}"
        state: present
      tags: copy_sshkey
      register: auth_success
    - name: Ping All Host 
      ping:
      when: auth_success.failed != true # 当复制主机秘钥成功后执行
    - name: Stop Firewalld
      shell: systemctl stop firewalld && systemctl disable firewalld  
    - name:  Ensure SELinux is set to enforcing mode
      ansible.builtin.lineinfile:
        path: /etc/selinux/config
        regexp: '^SELINUX='
        line: SELINUX=disabled
    - name: Install docker for binnery
      ansible.builtin.unarchive:
        src: ./docker-20.10.9.tgz
        dest: /usr/local/
    - name: Copy file docker.service
      ansible.builtin.copy: 
        src: ./docker.service
        dest: /usr/lib/systemd/system/docker.service
        mode: 0777
      register: copy_sucess
    - name: Create docker links
      ansible.builtin.file:
        src: '/usr/local/docker/{{ item.src }}'
        dest: '{{ item.dest }}'
        state: link
      loop:
        - { src: docker, dest: /usr/bin/docker }
        - { src: dockerd, dest: /usr/bin/dockerd }
      register: link_status
    - name: Just force systemd to reread configs
      ansible.builtin.systemd:
        daemon_reload: yes
    - name: Start service docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes
      when: link_status.results[0].failed != true
```

## 修改初始化的存储(不存储本地)
```yaml
apiVersion: kubekey.kubesphere.io/v1alpha1
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: kube-master1, address: 10.1.6.71 , internalAddress: 10.1.6.71, user: root, password: sjgtw@123}
  - {name: kube-master2, address: 10.1.6.72, internalAddress: 10.1.6.72, user: root, password: sjgtw@123}
  - {name: kube-master3, address: 10.1.6.73 , internalAddress: 10.1.6.73, user: root, password: sjgtw@123}
  - {name: kube-node1, address: 10.1.6.74, internalAddress: 10.1.6.74, user: root, password: sjgtw@123}
  - {name: kube-node2, address: 10.1.6.75, internalAddress: 10.1.6.75, user: root, password: sjgtw@123}
  - {name: kube-node3, address: 10.1.6.76, internalAddress: 10.1.6.76, user: root, password: sjgtw@123}
  roleGroups:
    etcd:
    - kube-master1
    - kube-master2
    - kube-master3
    master:
    - kube-master1
    - kube-master2
    - kube-master3
    worker:
    - kube-node1
    - kube-node2
    - kube-node3
  controlPlaneEndpoint:
    domain: lb.kubesphere.local
    address: "10.1.6.89"
    port: 6443
  kubernetes:
    version: v1.17.9
    imageRepo: kubesphere
    clusterName: cluster.local
    maxPods: 330
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
  registry:
    registryMirrors: []
    insecureRegistries: []
  addons:
  - name: nfs-client
    namespace: kube-system
    sources:
      chart:
        name: nfs-client-provisioner
        repo: https://charts.kubesphere.io/main
        valuesFile: ./values.yaml
```


### config-simple.yaml
```yaml
apiVersion: kubekey.kubesphere.io/v1alpha1
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: kube-master1, address: 10.1.6.71 , internalAddress: 10.1.6.71, user: root, password: sjgtw@123}
  - {name: kube-master2, address: 10.1.6.72, internalAddress: 10.1.6.72, user: root, password: sjgtw@123}
  - {name: kube-master3, address: 10.1.6.73 , internalAddress: 10.1.6.73, user: root, password: sjgtw@123}
  - {name: kube-node1, address: 10.1.6.74, internalAddress: 10.1.6.74, user: root, password: sjgtw@123}
  - {name: kube-node2, address: 10.1.6.75, internalAddress: 10.1.6.75, user: root, password: sjgtw@123}
  - {name: kube-node3, address: 10.1.6.76, internalAddress: 10.1.6.76, user: root, password: sjgtw@123}
  roleGroups:
    etcd:
    - node1
    master:
    - node1
    worker:
    - node1
    - node2
  controlPlaneEndpoint:
    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.17.9
    imageRepo: kubesphere
    clusterName: cluster.local
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
  registry:
    registryMirrors: []
    insecureRegistries: []
  addons: []
```



## 续费证书

> 仅适用与1.19以下版本

```shell
kubeadm alpha certs check-expiration
kubeadm  alpha certs renew apiserver # 更新ApiServer 
kubeadm  alpha certs renew all # 更新所有的证书
```

