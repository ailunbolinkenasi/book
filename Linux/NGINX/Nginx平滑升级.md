## Nginx平滑升级
> 介于最近的nginx爆出的最新安全漏洞,现在需要基于不停服对nginx进行版本升级.

nginx 安全漏洞(CVE-2021-23017)和NGINX 环境问题漏洞(CVE-2019-20372)，受影响版本为`0.6.18-1.20.0`
## Nginx平滑升级的原理
- 在不停掉老进程的情况下，启动新的进程。
- 老进程负责处理仍然没有处理完的请求，但不再接受处理请求。
- 新进程接受新请求。
- 老进程处理完所有请求，关闭所有连接后停止。
## Nginx信号介绍
- 主进程支持的信号
   - `TERM`, `INT`: 立刻退出
   - `QUIT`: 等待工作进程结束后再退出
   - `KILL`: 强制终止进程
   - `HUP`: 重新加载配置文件，使用新的配置启动工作进程，并逐步关闭旧进程。
   - `USR1`: 重新打开日志文件
   - `USR2`: 启动新的主进程，实现热升级
   - `WINCH`: 逐步关闭工作进程
- 工作进程支持的信号
   - `TERM`, `INT`: 立刻退出
   - `QUIT`: 等待请求处理结束后再退出
   - `USR1`: 重新打开日志文件
## 开始进行平滑升级
### 查看现有nginx的编译参数
- [1.23.1版本](http://nginx.org/download/nginx-1.23.1.tar.gz)
- 后期自动化升级的脚本我会写哦
```bash
[root@bogon ~]# nginx -V
nginx version: nginx/1.19.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx-1.19.0 --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module --add-module=/usr/local/fastdfs-nginx-module-1.22/src/
```

按照原来的编译参数安装 nginx 的方法进行安装，只需要`make`千万不要`make install`

> 如果你直接执行make install会覆盖掉你原来nginx的配置!!!

```bash
[root@bogon ~]# tar -zxf nginx-1.23.1.tar.gz
[root@bogon ~]# cd nginx-1.23.1
[root@bogon nginx-1.23.1]# ./configure --prefix=/usr/local/nginx-1.19.0 --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module --add-module=/usr/local/fastdfs-nginx-module-1.22/src/
[root@bogon nginx-1.23.1]# make
```

### 备份原有nginx二进制文件
```bashz
[root@bogon nginx-1.23.1]# mv /usr/local/nginx-1.19.0/sbin/nginx /usr/local/nginx-1.19.0/sbin/nginx_$(date +
```
### 复制新的nginx二进制文件
```bash
[root@bogon nginx-1.23.1]# cp objs/nginx /usr/local/nginx-1.19.0/sbin/
```
### 平滑升级重启
```shell
[root@bogon tmp]# nginx -s reload
[root@bogon tmp]# nginx -V
nginx version: nginx/1.23.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/usr/local/nginx-1.19.0 --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --without-mail_pop3_module --without-mail_imap_module --without-mail_smtp_module --add-module=/usr/local/fastdfs-nginx-module-1.22/src/
```