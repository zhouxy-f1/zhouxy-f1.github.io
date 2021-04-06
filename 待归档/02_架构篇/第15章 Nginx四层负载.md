## 理论基础

##### 四层负载定义

指在OSI传输层上的操作，四层只认 ip和port，常用硬件F5或LVS、Haproxy来做四层负载，在nginx1.9 版本以后可以用--with-stream模块实现四层负载，四层负载均衡必须配置在http核心层之外。

##### 四层负载作用

1、通过四层加七层，应对大并发场景；轮询多台七层，保障七层负载的高可用

2、mysql读负载均衡

3、端口映射（转发）

```shell
如:tcp协议的负载均衡，有些请求是TCP协议的(mysql、ssh)，或者说这些请求只需要使用4层进行端口的转发就可以了，所以使用4层负载均衡。
	比如做：mysql读的负载均衡（轮询）
	比如做：端口映射、端口转发         tcp/udp
```

##### 四层负载小结

1.四层负载均衡仅能转发TCP/IP协议、UDP协议，通常用来转发端口，如: tcp/3306，tcp/22，udp/53。
2.四层负载均衡可以用来解决七层负载均衡的端口限制问题。（七层负载均衡最大使用65535个端口号）
3.可以用来解决七层负载均衡的高可用问题。（多台后端七层负载均衡能同时的使用）
4.四层的转发效率比七层的高的多，但仅支持tcp/ip协议，不支持http或者https协议



##### 四层负载与七层对比

四层：内核空间 传输层 转发 （tcp、udp 、端口转发）

七层：用户空间 应用层 代理 （域名匹配、路径规则控制、安全、限速等）

## 配置四层负载均衡

##### 环境准备

| 主机名 | 外网IP    | 内网IP      | 角色         |
| ------ | --------- | ----------- | ------------ |
| lb     | 10.0.0.4  | 172.16.1.4  | 四层负载     |
| lb01   | 10.0.0.5  | 172.16.1.5  | 七层负载     |
| lb02   | 10.0.0.6  | 172.16.1.6  | 七层负载     |
| web01  | 10.0.0.7  | 172.16.1.7  | web服务器    |
| web02  | 10.0.0.8  | 172.16.1.8  | web服务器    |
| db01   | 10.0.0.51 | 172.16.1.51 | 数据库+redis |

##### 1.在web01上线wordpress

```shell
#安装配置程序
[root@web01 php]# rpm -ivh *.rpm
[root@web01 ~]# vim /etc/nginx/nginx.conf 
[root@web01 ~]# vim /etc/php-fpm.d/www.conf 

#布置代码
[root@web01 ~]# mkdir /code
[root@web01 ~]# unzip wordpress-5.2.3-zh_CN.zip  -d /code
[root@web01 ~]# chown -R www.www /code

#配置虚拟主机
[root@web01 /etc/nginx/conf.d]# vim blog.com.conf 
server{
        listen 80;
        server_name blog.com;
        root /code/wordpress;
        index index.php;
        client_max_body_size 100m;

        location ~ \.php$ {
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
        }
}
[root@web01 ~]# systemctl start nginx php-fpm

#配置数据库
[root@db01 ~]# yum install -y mariadb-server
[root@db01 ~]# mysql
MariaDB [(none)]> create database wordpress;
MariaDB [(none)]> grant all on wordpress.* to wp@'172.16.1.%' identified by '1';

#解析域名
Macbook:~ zhouxy$ sudo vim /etc/hosts
10.0.0.7 blog.com
```

##### 2、配置会话共享

```shell
#服务端:db01
[root@db01 ~]# yum install redis -y 
[root@db01 ~]# vim /etc/redis.conf
bind 127.0.0.1 172.16.1.51

#客户端：web01
[root@web01 ~]# vim /etc/php.ini
session.save_handler = redis
session.save_path = "tcp://172.16.1.51:6379"
session.auto_start = 1

[root@web01 ~]# vim /etc/php-fpm.d/www.conf 
;php_value[session.save_handler] = files
;php_value[session.save_path]    = /var/lib/php/session

#重启php服务
[root@web01 ~]# systemctl restart php-fpm
```

##### 3.复刻web01到web02

```shell
#安装nginx、php-fpm
[root@web02 php]# rpm -ivh *.rpm

#传nginx和php的配置
[root@web01 ~]# scp -rp /etc/nginx 172.16.1.8:/etc/

#传代码
[root@web01 ~]# scp -rp  /code 172.16.1.8:/

#解析域名
Macbook:~ zhouxy$ sudo vim /etc/hosts
10.0.0.8 blog.com
```

![image-20191002162203491](https://tva1.sinaimg.cn/large/006y8mN6ly1g7jxkihlxij313z0e0mz3.jpg)

##### 4.接入七层负载

```shell
#安装nginx
#启动服务
#配置虚拟主机
[root@lb01 /etc/nginx/conf.d]# cat blog.up.com.conf 
upstream blog.up.com {
        server 172.16.1.7:80;
        server 172.16.1.8:80;
}
server{
        listen 80;
        server_name blog.up.com;

        location / {
                proxy_pass http://blog.up.com;
                proxy_set_header HOST $http_host;
        }
}
```

##### ![image-20191005193032098](https://tva1.sinaimg.cn/large/006y8mN6ly1g7njx7tb5cj30nz0bkgmi.jpg)

##### 5.复刻lb01到lb02

```shell
[root@lb01 ~]# scp blog.up.com.conf 172.16.1.6:/etc/nginx/conf.d/
```

##### 6.配置nginx四层负载均衡

```shell
#删除http的80端口,否则请求会打到default主机
[root@lb ~]# rm -f /etc/nginx/conf.d/default.conf  

#四层负载配置在events和http中间
[root@lb ~]# vim /etc/nginx/nginx.conf
events {
  ....
}
include /etc/nginx/conf.c/*.conf;

http {
	....
}

#创建四层配置目录和虚拟主机
[root@lb ~]# mkdir /etc/nginx/conf.c
[root@lb ~]# cd /etc/nginx/conf.c
[root@lb /etc/nginx/conf.c]# vim lb.4up.com.conf 
stream{
        upstream blog.4up.com {
                server 172.16.1.5:80 weight=5 max_fails=3 fail_timeout=30s;
                server 172.16.1.6:80 weight=5 max_fails=3 fail_timeout=30s;
        }
        server {
                listen 80;
                proxy_pass blog.4up.com;
                proxy_connect_timeout 3s;
                proxy_timeout 3s;
        }
}

#重载服务
[root@lb /etc/nginx/conf.c]# nginx -t 
[root@lb /etc/nginx/conf.c]# nginx -s reload
#域名解析
Macbook:~ zhouxy$ sudo vim /etc/hosts
10.0.0.4 blog.4up.com

```

## 使用四层负载实现tcp的转发

请求负载均衡 5555    --->     172.16.1.7:22;
请求负载均衡 6666    --->     172.16.1.51:3306;

```shell
upstream ssh_7 {
	server 10.0.0.7:22;
}
upstream mysql_51 {
	server 10.0.0.51:3306;
}
#监听本机5555端口，把请求转到资源池ssh_7
server {
    listen 5555;
    proxy_connect_timeout 3s;
    proxy_timeout 300s;
    proxy_pass ssh_7;
}
#监听本机6666端口，请求转到资源池mysql_51
server {
    listen 6666;
    proxy_connect_timeout 3s;
    proxy_timeout 3s;
    proxy_pass mysql_51;
}
```
## nginx四层负载均衡记录日志

##### 定义和调用

```shell
[root@lb /etc/nginx/conf.c]# vim lb.4up.com.conf    
stream{
        upstream blog.4up.com {
                server 172.16.1.5:80 weight=5 max_fails=3 fail_timeout=30s;
                server 172.16.1.6:80 weight=5 max_fails=3 fail_timeout=30s;
        }
#定义
        log_format  proxy '$remote_addr $remote_port - [$time_local] $status $protocol '
                          '"$upstream_addr" "$upstream_bytes_sent" "$upstream_connect_time"' ;
#调用
        access_log /var/log/nginx/proxy.log proxy;

        server {
                listen 80;
                proxy_pass blog.4up.com;
                proxy_connect_timeout 3s;
                proxy_timeout 3s;
        }
}
```

##### 日志展示效果

```shell
[root@lb /etc/nginx]# cat /var/log/nginx/proxy.log 
10.0.0.1 56451 - [05/Oct/2019:20:03:31 +0800] 200 TCP "172.16.1.5:80" "514" "0.004"
10.0.0.1 56452 - [05/Oct/2019:20:03:31 +0800] 200 TCP "172.16.1.6:80" "0" "0.004"
10.0.0.1 56456 - [05/Oct/2019:20:03:36 +0800] 200 TCP "172.16.1.5:80" "514" "0.005"
```