## 一、Nginx与PHP集成

##### 1、Fastcgi工作原理

![image-20190820083939536](http://ww3.sinaimg.cn/large/006tNc79ly1g65uk5wq70j30mr076tdd.jpg)

​	1)、用户发起请求到达Nginx服务器，Nginx的/location判断请求是否静态资源，静态资源直接处理，返回给用户

​	2)、如果用户请求是动态资源，Nginx处理不了，就会通过Fastcgi协议把请求交给php-fpm管理进程；

​	3)、php-fpm管理进程生成工作进程wrapper来接管，如果请求只是解析php代码则直接返回结果，如果需要调用数据库			等，则php连接数据库，再原路返回结果

​	4)、返回路径：mysql > php > php-fpm > Fastcgi > ngnix > http > user

##### 2、配置Nginx和Fastcgi

![image-20190906104221564](https://tva1.sinaimg.cn/large/006y8mN6gy1g6pln0346oj30r307x40s.jpg)



## 二、php与远程mysql集成

##### 1、在db01主机上准备数据库

```shell
[root@db01 ~]# yum install -y mariadb-server
[root@db01 ~]# systemctl start mariadb
[root@db01 ~]# systemctl enable mariadb
[root@db01 ~]# mysqladmin password 123
[root@db01 ~]# mysql -uroot -p123
MariaDB [(none)]> grant all on *.* to wp@'172.16.1.%' identified by '123';
```

##### 2、web01上写测试文件

```shell
[root@web01 /code]# vim tomysql.php 
<?php
        $servername="172.16.1.51";
        $username="wp";
        $password="123";
        //创建连接
        $conn = mysqli_connect($servername,$username,$password);
        //检测连接
        if (!$conn) {
                die ("Connection failed:".mysqli_connect_error());
        }
        echo "php连接mysql数据库成功";
?>

```

##### 3、测试结果

![image-20190906111054635](https://tva1.sinaimg.cn/large/006y8mN6gy1g6pmgpnouxj30o2072q3l.jpg)

## 三、wordpress配置示例

##### 0、环境准备

| 主机名 |   角色    |   内网IP    |  外网IP   |
| :----: | :-------: | :---------: | :-------: |
| web01  | web服务器 | 172.16.1.7  | 10.0.0.7  |
|  db01  |  数据库   | 172.16.1.51 | 10.0.0.51 |

##### 1、web01安装nginx、php

```shell
1）更换官方源
#Nginx
[root@web01 ~]# vim /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1

#php
[root@web01 ~]# vim /etc/yum.repos.d/php.repo
[php-webtatic]
name = PHP Repository
baseurl = http://us-east.repo.webtatic.com/yum/el7/x86_64/
gpgcheck = 0

2）安装软件
#安装nginx
[root@web01 ~]# yum install -y nginx
#安装php
[root@web01 ~]# yum remove php-mysql-5.4 php php-fpm php-common
[root@web01 ~]# yum -y install php71-php php71-php-cli php71-php-common php71-php-devel php71-php-embedded php71-php-gd php71-php-mcrypt php71-php-mbstring php71-php-pdo php71-php-xml php71-php-fpm php71-php-mysqlnd php71-php-opcache php71-php-pecl-memcached php71-php-pecl-redis php71-php-pecl-mongodb
#检查是否安装上
[root@web01 ~]# nginx -v
nginx version: nginx/1.16.1
[root@web01 ~]# php -v
PHP 7.1.31 (cli) (built: Aug  4 2019 09:25:59) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.1.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.1.31, Copyright (c) 1999-2018, by Zend Technologies
    
3）修改启动用户
[root@web01 ~]# vim /etc/nginx/nginx.conf 
user  www;
[root@web01 ~]#  vim /etc/php-fpm.d/www.conf 
user = www
group = www

4）新建统一用户
[root@web01 ~]# groupadd www -g 666
[root@web01 ~]# useradd www -u 666 -g 666 -s /sbin/nologin -M

5）启动并加入开机自启
[root@web01 ~]# systemctl start nginx php-fpm
[root@web01 ~]# systemctl enable nginx php-fpm
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/php-fpm.service to /usr/lib/systemd/system/php-fpm.service.

6）查看进程是否启动
[root@web01 ~/php71w-7.1.31]# ps -ef |grep -E 'php|nginx'
root       7236      1  0 19:27 ?        00:00:00 php-fpm: master process (/etc/php-fpm.conf)
root       7237      1  0 19:27 ?        00:00:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
www        7238   7237  0 19:27 ?        00:00:00 nginx: worker process
www        7239   7236  0 19:27 ?        00:00:00 php-fpm: pool www
www        7240   7236  0 19:27 ?        00:00:00 php-fpm: pool www
www        7241   7236  0 19:27 ?        00:00:00 php-fpm: pool www
www        7242   7236  0 19:27 ?        00:00:00 php-fpm: pool www
www        7243   7236  0 19:27 ?        00:00:00 php-fpm: pool www
root       7262   7064  0 19:29 pts/0    00:00:00 grep --color=auto -E php|nginx

7）测试php
[root@web01 ~]# vim /code/index.php
<?php
		phpinfo();
?>
```

##### 2、配置nginx

```shell
#重写nginx配置文件
[root@web01 /etc/nginx/conf.d]# rm -f default.conf 
[root@web01 /etc/nginx/conf.d]# vim blog.drz.com.conf

server {
        listen 80;
        server_name blog.drz.com;
        root /code/wordpress;

        location / {
                index index.php index.html;
        }
        location ~ \.php$ {
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
        }
}
```

##### 3、部署站点

```shell
#新建站点目录
[root@web01 ~]# mkdir /code
#上传
[root@web01 ~]#rz
#解压代码
[root@web01 ]# tar xf wordpress-5.2.2-zh_CN.tar.gz -C /code
#新建静态资源目录
[root@web01 /code]# mkdir /wordpress/wp-content/uploads
#站点授权
[root@web01 /code]#chown -R www.www /code
```

##### 4、域名解析

```shell
Macbook:~ zhouxy$ sudo vim /etc/hosts
Password:
10.0.0.7 blog.drz.com zh.drz.com edu.drz.com
```

##### 5、访问网站

输入：blog.drz.com

![image-20190826125233555](https://tva1.sinaimg.cn/large/006y8mN6ly1g6czl33gpfj313z0m1n0r.jpg)

##### 6、db01部署数据库

```shell
1.安装mariadb并启动

#安装
[root@db01 ~]# yum install -y mariadb-server
#启动
[root@db01 ~]# systemctl start mariadb
[root@db01 ~]# systemctl enable mariadb

2）配置数据库

#设置密码
[root@db01 ~]# mysqladmin  password '123'

#创建数据库
方法一：[root@db01 ~]# mysqladmin -uroot -p123 create wordpress
方法二：[root@db01 ~]# mysql -uroot -p123 -e 'create database wordpress'

#免交互创建程序连接MySQL用户
[root@db01 ~]# mysql -uroot -p123 -e "grant all on wordpress.* to wp@'172.16.1.%' identified by '1'"

#在web01上测试
[root@web01 ~]# mysql -uwp -p1 -h172.16.1.51
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 42
Server version: 5.5.60-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```

##### 7、浏览器访问

![image-20190826130100905](https://tva1.sinaimg.cn/large/006y8mN6ly1g6cztwgc4uj313z0m1wgd.jpg)

![image-20190906121414126](https://tva1.sinaimg.cn/large/006y8mN6gy1g6poalm08xj313z0lytbl.jpg)

##### 8、nginx和php上传文件限制

```shell
#nginx上server层默认为1m,改为100m
[root@web01 /etc/nginx/conf.d]# vim blog.123.com.conf 
server{
	client_max_body_size 100m;
}

#php.ini的配置
[root@web01 /code/wordpress]# vim /etc/php.ini 
upload_max_filesize = 200M
post_max_size = 200M
```



##### 其它开源代码：

phpmyadmin：https://www.phpmyadmin.net/downloads/

zblog：https://www.zblogcn.com/zblogphp/

wordpress：https://cn.wordpress.org/download/

wecenter：http://www.wecenter.com/downloads/

edusohu：http://www.edusoho.com/open/show



##### edusoho配置文件

```shell
server {
    listen 80;
    # [改] 网站的域名
    server_name edu.drz.com;
    #301跳转可以在nginx中配置
    # 程序的安装路径
    root /code/edusoho/web;
    # 日志路径
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;
    location / {
        index app.php;
        try_files $uri @rewriteapp;
    }
    location @rewriteapp {
        rewrite ^(.*)$ /app.php/$1 last;
    }
    location ~ ^/udisk {
        internal;
        root /var/www/edusoho/app/data/;
    }
    location ~ ^/(app|app_dev)\.php(/|$) {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
        fastcgi_param  HTTPS              off;
        fastcgi_param HTTP_X-Sendfile-Type X-Accel-Redirect;
        fastcgi_param HTTP_X-Accel-Mapping /udisk=/code/edusoho/app/data/udisk;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 8 128k;
    }

    # 配置设置图片格式文件
    location ~* \.(jpg|jpeg|gif|png|ico|swf)$ {
        # 过期时间为3年
        expires 3y;

        # 关闭日志记录
        access_log off;

        # 关闭gzip压缩，减少CPU消耗，因为图片的压缩率不高。
        gzip off;
    }

    # 配置css/js文件
    location ~* \.(css|js)$ {
        access_log off;
        expires 3y;
    }

    # 禁止用户上传目录下所有.php文件的访问，提高安全性
    location ~ ^/files/.*\.(php|php5)$ {
        deny all;
    }

    # 以下配置允许运行.php的程序，方便于其他第三方系统的集成。
    location ~ \.php$ {
        # [改] 请根据实际php-fpm运行的方式修改
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
        fastcgi_param  HTTPS              off;
    }
}
```





 











