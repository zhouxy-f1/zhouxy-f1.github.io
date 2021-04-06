## 单台服务器实现动静分离

```shell
[root@lb01~ ]# vim /etc/nginx/conf.d/blog.com.conf
server {
	listen 80;
	server_name blog.com;
	root /code/wordpress;
	index index.php;
#动态资源匹配	
	location ~ \.php$ {
			fastcgi_pass 127.0.0.1:9000;
			fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
			include fastcgi_params;
	}
#静态资源匹配	
	location ~ \.(png|jpg|gif)$ {
			root /code/wordpress/wp-content/uploads;
			gzip on;
	}
}
```

## 通过负载实现动静分离

此次用于做实验，实际生产中会用上面的那种（负载只做分发，每个web上做动静location分离）

```shell
[root@lb01 ~]vim /etc/nginx/conf.d/ds.123.com.conf
upstream static {
   server 172.16.1.7:80;
}
upstream java {
	server 172.16.1.8:8080;
}
server{
	listen 80;
	index index.html;
	root /code;
	
	location ~* \.(png|jpg|gif)$ {
			proxy_pass http://static;
			proxy_set_header Host $http_host;
	}
	location ~ \.php$ {
			proxy_pass http://java;
			proxy_set_header Host $http_host;
	}
}
```



## 根据不同终端实现资源分离

通过负载均衡实现手机端与PC端访问时返回不同页面

##### 1.环境规划

| 主机角色        | 外网IP   | 提供的端口 |
| --------------- | -------- | ---------- |
| 七层负载        | 10.0.0.5 | 80         |
| 提供Android页面 | 10.0.0.7 | 9090       |
| 提供Iphone页面  | 10.0.0.7 | 9091       |
| 提供PC页面      | 10.0.0.7 | 9092       |

##### 2.配置web节点

```shell
[root@web01 /etc/nginx/conf.d]# cat www.up.com.conf 
server {
        listen 9090;
        location / {
                root /code/android;
                index index.html;
        }
}
server {
        listen 9091;
        location / {
                root /code/iphone;
                index index.html;
        }
}
server {
        listen 9092;
        location / {
                root /code/pc;
                index index.html;
        }
}
```

##### 3.配置负载

```shell
[root@lb01 /etc/nginx/conf.d]# cat www.up.com.conf 
upstream android {
        server 172.16.1.7:9090;
}
upstream iphone {
        server 172.16.1.7:9091;
}
upstream pc {
        server 172.16.1.7:9092;
}
server {
        listen 80;
        server_name www.up.com;
        location / {
                if ( $http_user_agent ~* "Android" ){
                        proxy_pass http://android;
                }
                if ($http_user_agent ~* "Iphone"){
                        proxy_pass http://iphone;
                }
                        proxy_pass http://pc;
        }
}
```

##### 展示效果

![image-20191015204119879](https://tva1.sinaimg.cn/large/006y8mN6gy1g7z649tuqmj30rl05cdg2.jpg)

![image-20191015203720588](https://tva1.sinaimg.cn/large/006y8mN6gy1g7z605gvh9j310707zq3l.jpg)

![image-20191015204016997](https://tva1.sinaimg.cn/large/006y8mN6gy1g7z634fr12j310u07z0tv.jpg)









