## 轮询带来的问题

##### 会话保持问题来源

```shell
当用户向服务器请求一个页面，服务器在返回结果时会附带发送一个session_id，存于浏览器的cookies中，下次同再发送请求时会携带这个cookies中的session_id，服务器收到请求会与服务器本地/var/lib/php/session中的id校验，一致则保持登陆状况。

但负载均衡分配请求给web集群是通过http协议，而http是无状态协议，不会保存登陆信息，所以当网页刷新再次请求页面时会通过负载轮询到别的web，导致丢失登陆信息，退出登陆。比如用户在写文章，只要页面刷新就会直接退出，文章丢失。
```

##### 解决办法

```shell
 1.ip_hash						 //如遇NAT,n台主机共用一个出口IP，会导致单台服务器压力过大
 2.session复制					//麻烦
 3.session存入mysql			//较redis慢
`4.session存入redis 		//优先，因为redis是内存存储，速度快
```

## 故障模拟

因为只有phpMyAdmin是把session存放在web服务器本地的/var/lib/php/session目录中,所以用phpMyAdmin来做实验。

##### 1.在web01上线代码

```shell
1.配置Nginx
[root@web01 conf.d]# vim php123.com.conf
server {
	listen 80;
	server_name php.oldboy.com;
	root /code/phpMyAdmin-4.8.4-all-languages;

	location / {
		index index.php index.html;
	}

	location ~ \.php$ {
		fastcgi_pass 127.0.0.1:9000;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		include fastcgi_params;
	}
}
[root@web01 conf.d]# systemctl restart nginx

2.phpmyadmin代码上线 （web01和web02上都装）
[root@web01 conf.d]# cd /code
[root@web01 code]# wget https://files.phpmyadmin.net/phpMyAdmin/4.8.4/phpMyAdmin-4.8.4-all-languages.zip
[root@web01 code]# unzip phpMyAdmin-4.8.4-all-languages.zip

3.配置phpmyadmin连接远程的数据库
[root@web01 code]# cd phpMyAdmin-4.8.4-all-languages/
[root@web01 phpMyAdmin-4.8.4-all-languages]# cp config.sample.inc.php config.inc.php
[root@web01 phpMyAdmin-4.8.4-all-languages]# vim config.inc.php
/* Server parameters */
$cfg['Servers'][$i]['host'] = '172.16.1.51';

4.配置授权
[root@web01 conf.d]# chown -R www.www /var/lib/php/
```

##### 2.复刻web01到web02

```shell
1.安装nginx和php-fpm 并启动服务

2.将web01上配置好的phpmyadmin以及nginx的配置文件推送到web02主机上
[root@web01 code]# scp -rp  phpMyAdmin-4.8.4-all-languages root@172.16.1.8:/code/
[root@web01 code]# scp /etc/nginx/conf.d/php.conf  root@172.16.1.8:/etc/nginx/conf.d/

3.在web02上重载Nginx服务
[root@web02 code]# systemctl restart nginx

4.授权
[root@web02 code]# chown -R www.www /var/lib/php/
```

##### 3.接入七层负载

```shell
#安装nginx并配置
[root@lb01 /etc/nginx/conf.d]# vim proxy_php.123.com.conf 
upstream php {
        server 172.16.1.7:80;
        server 172.16.1.8:80;
}
server{
        listen 80;
        server_name php.123.com;

        location /{
                proxy_pass http://php;
                proxy_set_header HOST $http_host;
        }
}
[root@lb01 conf.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@lb01 conf.d]# systemctl restart nginx
```

##### 结果

原因分析：负载轮询分配用户请求，一台web收到请求后，输入用户名密码，再次点击，请求分配到另一台web，永远无法登陆

![image-20191002144402582](https://tva1.sinaimg.cn/large/006y8mN6ly1g7juqke8y3j30wt0iymyw.jpg)

## 故障解决

##### 使用redis解决会话登录问题

把redis安装在db01上

```shell

1.安装redis内存数据库
[root@db01 ~]# yum install redis -y

2.配置redis监听在172.16.1.0网段上
[root@db01 ~]# sed  -i '/^bind/c bind 127.0.0.1 172.16.1.51' /etc/redis.conf

3.启动redis
[root@db01 ~]# systemctl start redis
[root@db01 ~]# systemctl enable redis


4.php配置session连接redis
#1.修改/etc/php.ini文件
[root@web ~]# vim /etc/php.ini
session.save_handler = redis
session.save_path = "tcp://172.16.1.51:6379"
;session.save_path = "tcp://172.16.1.51:6379?auth=123" #如果redis存在密码，则使用该方式
session.auto_start = 1

#2.注释php-fpm.d/www.conf里面的两条内容，否则session内容会一直写入/var/lib/php/session目录中
;php_value[session.save_handler] = files
;php_value[session.save_path]    = /var/lib/php/session

3.重启php-fpm
[root@web01 code]# systemctl restart php-fpm

4.将web01上配置好的文件推送到web02
[root@web01 code]# scp /etc/php.ini root@172.16.1.8:/etc/php.ini  
[root@web01 code]# scp /etc/php-fpm.d/www.conf root@172.16.1.8:/etc/php-fpm.d/www.conf 

5.上web02上重启php-fpm
[root@web02 code]# systemctl restart php-fpm

6.redis查看数据
[root@db01 ~]# redis-cli 
127.0.0.1:6379> keys *
1) "PHPREDIS_SESSION:89f1fc340e4680f46e503df129d9ef67"
```

