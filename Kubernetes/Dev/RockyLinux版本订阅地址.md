## RockyLinux版本订阅地址

- [RockyLinux8.6](https://download.rockylinux.org/pub/rocky/8/isos/x86_64/Rocky-8.6-x86_64-minimal.iso): 推荐`Mini版本`

| Os-version      | runc       |
| --------------- | ---------- |
| Rocky-Linux8.6+ | Containerd |

## RockyLinux配置和启动网卡说明

RockyLinux8已经不支持通过`systemctl start network`这种方式启动网卡了!

### 配置IP地址

```shell
[root@localhost ~]# vim /etc/sysconfig/network-scripts/ifcfg-ens192 # 看你自己的网卡名称是什么
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static  # 改成静态IP
DEFROUTE=yes 
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=ens192
UUID=99db262a-16c0-4762-85b4-f2da283af804
DEVICE=ens192
ONBOOT=yes
IPADDR=10.1.6.46     
NETMASK=255.255.255.0
GATEWAY=10.1.6.254
DNS1=223.5.5.5
```

### 启动网卡

```shell
nmcli c reload
nmcli c down ens33 # shutdown ens33网卡
nmcli c up ens33 # 启动ens33网卡
```

## 使用中科大RockyLinux镜像源

替换前请注意备份文件!

```shell
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.ustc.edu.cn/rocky|g' \
    -i.bak \
    /etc/yum.repos.d/Rocky-AppStream.repo \
    /etc/yum.repos.d/Rocky-BaseOS.repo \
    /etc/yum.repos.d/Rocky-Extras.repo \
    /etc/yum.repos.d/Rocky-PowerTools.repo
```