## 准备环境
1. 准备自己的云服务器或者云轻量服务器
2. 准备三大件`MySQL、PHP、Nginx`
```shell
wget -c http://mirrors.linuxeye.com/oneinstack-full.tar.gz && tar xzf oneinstack-full.tar.gz && ./oneinstack/install.sh --nginx_option 1 --php_option 11 --php_extensions imagick,fileinfo,phalcon --db_option 2 --dbinstallmethod 1 --dbrootpwd kysed9v1 --reboot
```
## 安装Composer
- [官网](https://getcomposer.org/)
```bash
[root@bogon ~]# php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
[root@bogon ~]# php -r "if (hash_file('sha384', 'composer-setup.php') === 'e21205b207c3ff031906575712edab6f13eb0b361f2085f1f1237b7126d785e826a450292b6cfd1d64d92e6563bbde02') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
[root@bogon ~]# php composer-setup.php
[root@bogon ~]# php -r "unlink('composer-setup.php');"
```

## 创建一个虚拟主机配置
1. 进入三件套目录
```shell
[root@bogon ~]# cd oneinstack
```

2. 执行`vhost.sh`
```shell
[root@bogon oneinstack]# ./vhost.sh
What Are You Doing?
	1. Use HTTP Only
	2. Use your own SSL Certificate and Key
	3. Use Let's Encrypt to Create SSL Certificate and Key
	q. Exit
Please input the correct option: 1
Please input domain(example: www.example.com): 10.1.6.182
domain=10.1.6.182

Please input the directory for the domain:10.1.6.182 :
(Default directory: /data/wwwroot/10.1.6.182):
Create Virtul Host directory......
set permissions of Virtual Host directory......

Do you want to add more domain name? [y/n]: n

Do you want to add hotlink protection? [y/n]: n

Allow Rewrite rule? [y/n]: n

Allow Nginx/Tengine/OpenResty access_log? [y/n]: n
```
## 开始安装Flarum
```shell
[root@bogon ~]# cd /data/wwwroot/10.1.6.182/ # 进入你自己的域名目录下
[root@bogon 10.1.6.182]# composer create-project flarum/flarum . # 安装flarum
```
