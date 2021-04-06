## 1、自动索引模块

ngx_http_autoindex_module模块处理以斜杠字符(/)结尾的请求，并自动生成目录列表 ，只有当ngx_http_index_module模块未找到索引文件时才会传递给autoindex模块。默认是不允许列出整个目录浏览下载。

##### 指令语法

```shell
#启用或禁用目录列表输出，on开启，off关闭
Syntax : autoindex on|off;
Default: autoindex off;
Context: http,server,location

#指定是否应在目录列表中输出确切的文件大小，on代表显示字节，off代表以易读的方式带单位显示大小
Syntax : autoindex_exact_size on|off;
Default: autoindex_exact_size on;
Context: http,server,location

#指定目录列表中的时间以本地时间还是零时区的时间，on是本地时间(GMT+8)，off是零时区的时间
Syntax :autoindex_localtime on|off;
Default:autoindex_localtime off;
Context:http,server,location

#解决中文乱码问题，写在server层，所有location都生效
charset utf-8,gbk;
```

##### 配置实例

```shell
[root@web01 /etc/nginx/conf.d]# vim autoindex.conf 
server{
        listen 80;
        server_name auto.123.com;
        charset utf-8,gbk;
        
        location /download {
                alias /autoindex;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
        }
}

[root@web01 /etc/nginx/conf.d]# mkdir /autoindex/{A..D}{1..3} -p
#解析域名
Macbook:~ zhouxy$ sudo vim /etc/hosts
10.0.0.7 auto.123.com

```

##### 效果展示

![image-20190905132635242](https://tva1.sinaimg.cn/large/006y8mN6gy1g6okrk6tnfj30m808jq43.jpg)



## 2、状态监控模块

ngx_http_stub_status_module模块用于监控网站TCP连接的基本信息，由此模块自动生成一个显示信息的页面。通过yum安装时，默认安装此模块。但编译安装默认情况下不安装此模块，编译时要用参数--with-http_stub_status_module来安装

##### 配置语法

```shell
Syntax: stub_status;
Context: server, location
```

##### 配置实例

```shell
[root@web01 /etc/nginx/conf.d]# vim autoindex.conf 
server{
        listen 80;
        server_name auto.123.com;
        charset utf-8,gbk;

        location /{
                root /code;
                index index.html index.htm;
        }
        location /download {
                alias /autoindex;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
        }
        location /status{
                stub_status;
        }
}
```

##### 效果展示

![image-20190905133627874](https://tva1.sinaimg.cn/large/006y8mN6gy1g6ol1v5d3yj30mj06gwf2.jpg)

##### 状态页参数解读

```shell
Active connections  # 当前活动的连接总数：waiting+writing
accepts             # 已接受的TCP连接总数
handled             # 已处理的TCP连接数
requests            # 客户端发起的http请求总数

Reading             # 当前nginx读取请求头的连接数
Writing             # 当前nginx将响应写回客户端的连接数
Waiting             # 当前等待请求的空闲客户端连接数，开启了keepalive

# 注意, 一次TCP的连接，可以发起多次http的请求, 如下参数可配置进行验证
location /status{
        stub_status;
        keepalive_timeout  0;   # 类似于关闭长连接
				keepalive_timeout  65;  # 65s没有活动则断开连接
}
```

## 3、访问控制过滤模块

ngx_http_access_module 模块是基于IP地址的，可以在location中设置过滤规则，用户访问本location时，会自上而下检查过滤规则来确认放行还是拒绝，（拒绝会报错403），常用于防Dos攻击。这样的配置相对粗放，有漏洞，比如只拒绝某个IP，这个IP依然可以经代理服务器，用代理服务器的IP来访问；再比如只允许某个IP，可以通过VPN连上此IP的服务器，再通过这个服务器来访问。

##### 官方示例

```shell
#自上而下检查过滤规则
location / {
    deny  192.168.1.1;
    allow 192.168.1.0/24;
    allow 10.1.1.0/16;
    allow 2001:0db8::/32;
    deny  all;
}
```

##### 示例1:防dos攻击

```shell
#拒绝指定的IP,其他全部允许
server{
        listen 80;
        server_name auto.123.com;
        charset utf-8,gbk;

        location /{
                root /code;
                index index.html index.htm;
        }

        location /download {
                alias /autoindex;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
        }
        location /status{
                stub_status;
                deny 10.0.0.1/32;
                allow all
        }
}
#
此时通过浏览器访问 auto.123.com/status时报错403
通过web01，用curl 10.0.0.7/status可以正常访问
```

##### 示例2：生产中用法

```shell
#只允许本机回环口访问
server{
        listen 80;
        server_name auto.123.com;
        charset utf-8,gbk;

        location /{
                root /code;
                index index.html index.htm;
        }

        location /download {
                alias /autoindex;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
        }
        location /status{
                stub_status;
                allow 127.0.0.1;
                deny all;
        }
}
```



##  4、基础认证模块

ngx_http_auth_basic_module 模块允许通过使用“HTTP基本身份验证”协议验证用户名和密码来限制对资源的访问。客户端无权限时报错401

##### 配置步骤

```shell
#1、创建密码文件
[root@web01 /etc/nginx/conf.d]# yum install -y httpd-tools
[root@web01 /etc/nginx/conf.d]# htpasswd  -c -b /etc/nginx/htpwd zhouxy zhouxy
Adding password for user zhouxy
[root@web01 /etc/nginx/conf.d]# cat /etc/nginx/htpwd 
zhouxy:$apr1$PfCYvA2c$SEQ5cJV.yAJ1xm4MzhVGZ1

#2、把密码文件应用到/download的location中
server{
        listen 80;
        server_name auto.123.com;
        charset utf-8,gbk;

        location /{
                root /code;
                index index.html index.htm;
        }
        location /download {
                alias /autoindex;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
                auth_basic "username and password please! ";
                auth_basic_user_file /etc/nginx/htpwd;
        }
        location /status{
                stub_status;
                allow 127.0.0.1;
                deny all;
        }
}
```

##### 效果展示

![image-20190905165244846](https://tva1.sinaimg.cn/large/006y8mN6gy1g6oqq378v7j30r80c8jsw.jpg)

##### 报错401

当用户和密码验证不通过时，报错401

![image-20190905165350432](https://tva1.sinaimg.cn/large/006y8mN6gy1g6oqr7viv1j30r807i3za.jpg)

##### 访问日志

```shell
[root@web01 /etc/nginx/conf.d]# tailf /var/log/nginx/access.log
10.0.0.1 - zhouxy [05/Sep/2019:17:01:36 +0800] "GET /download/ HTTP/1.1" 200 1317 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.87 Safari/537.36" "-"
```



## 5、访问连接数限制模块

ngx_http_limit_conn_module模块用于限制同时连接服务器的连接数（一次连接可发起N次请求，所以此设置有点鸡肋）。

并非所有连接都被计算在内 只有当服务器正在处理请求并且已经读取了整个请求标头时，才会计算连接。一般设置为500.

##### 在http层定义

```shell
[root@web01 /etc/nginx/conf.d]# vim ../nginx.conf
http{
	……
	 limit_conn_zone $remote_addr zone=max_conn:1m;
	……
}
```

##### 在server层调用

```shell
server{
        listen 80;
        server_name auto.123.com;
        charset utf-8,gbk;
        limit_conn max_conn 1;

        location /{
                root /code;
                index index.html index.htm;
        }

        location /download {
                alias /autoindex;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
                auth_basic "give me you name and password";
                auth_basic_user_file /etc/nginx/htpwd;
        }
        location /status{
                stub_status;
        }
}
```

##### 超限客户端报错

![image-20190905181157562](https://tva1.sinaimg.cn/large/006y8mN6gy1g6ot0i07nkj30k103yq3g.jpg)

##### 超限制错误日志

![image-20190905181703421](https://tva1.sinaimg.cn/large/006y8mN6gy1g6ot5sy7vuj30zs05gmzy.jpg)

## 6、访问请求频率限制模块

ngx_http_limit_req_module模块是针对用户请求频率做限制，可以基于ip或hostname等，超出请求上限后默认返回503（服务器过载）；打开一个页面需要请求的元素假定为n，那么设置上限频率*连接超时时间的结果不得小于n，否则出现破图或加载不出来

##### 在http层定义

```shell
[root@web01 /etc/nginx/conf.d]# vim ../nginx.conf
http{
	……
	 limit_req_zone $binary_remote_addr zone=max_req:10m rate=1r/s;
	……
}

```

##### 在server或location层调用

```shell
server{
        listen 80;
        server_name auto.123.com;
        charset utf-8,gbk;
        limit_conn max_conn 1;										
        limit_req zone=max_req burst=5 nodelay;		//允许延迟处理5个请求；nodelay是其它全部无延迟拒绝
        limit_req_status 444;											//自字义状态码
        error_page 444 /err_444.html;							//接收到444，后返回444错误页面给用户，返回时重新加载
        location /{																	auto.123.com,提取/code/err_444.html给用户
                root /code;
                index index.html index.htm;
        }

        location /download {
                alias /autoindex;
                autoindex on;
                autoindex_exact_size off;
                autoindex_localtime on;
                auth_basic "give me you name and password";
                auth_basic_user_file /etc/nginx/htpwd;
        }
        location /status{
                stub_status;
        }
}
```

##### 超出限制报错

![image-20190905185446861](https://tva1.sinaimg.cn/large/006y8mN6gy1g6ou92pj85j30vc074t9k.jpg)

##### 超限错误日志

![image-20190905185804935](https://tva1.sinaimg.cn/large/006y8mN6gy1g6ouci9lgej30ow03swg8.jpg)

##### 自定义状态码

```shell
limit_req_status 444;
```

##### 效果展示

![image-20190905190019043](https://tva1.sinaimg.cn/large/006y8mN6gy1g6ouetocqvj30ub0ert9o.jpg)

![image-20190905190206332](https://tva1.sinaimg.cn/large/006y8mN6gy1g6ougo1iw2j30ov03zdh4.jpg)

## 7、反向代理php模块

ngx_http_fastcgi_module模块用于nginx连接php

```shell
location / {
#指定fastcgi监听的端口和地址，localhost或127.0.0.1代表本机
		fastcgi_pass  localhost:9000;
    fastcgi_index index.php;
#
    fastcgi_param SCRIPT_FILENAME $ducument_root$fastcgi_script_name;
    fastcgi_param QUERY_STRING    $query_string;
    fastcgi_param REQUEST_METHOD  $request_method;
    fastcgi_param CONTENT_TYPE    $content_type;
    fastcgi_param CONTENT_LENGTH  $content_length;
}
```

##### 实例:配置wordpress

```shell
server {
        listen 80;
        server_name blog_proxy.drz.com;
        root /code/wordpress;
        index index.php index.html;
        access_log /var/log/nginx/blog.drz.com_access.log main;

        location ~\.php$ {
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
        }
}
```



## 8、location模块

##### 匹配优先级

如果一个server{	}里包含多个location{	}，nginx会根据用户请求的url，按一定的优先级来匹配location：

规则1：优先级相同时，则按照/etc/nginx/conf.d下.conf文件排列顺序执行；

规则2：同一请求，可匹配多个location时，以字符匹配长度高者优先

| **匹配符** |          **匹配规则**          | **优先级** |
| :--------: | :----------------------------: | :--------: |
|     =      |            精确匹配            |     1      |
|     ^*     | 以某个字符串开头，不区分大小写 |     2      |
|     *$     | 以某个字符串结尾，不区分大小写 |     2      |
|     ~      |    严格区分大小写的正则匹配    |     3      |
|     ~*     |     不区分大小写的正则匹配     |     3      |
|     !~     |  严格区分大小写的不匹配的正则  |     4      |
|    !~*     |    不区分大小写不匹配的正则    |     4      |
|     /      |  通用匹配，任何请求都会匹配到  |     5      |

##### 应用场景

```shell

http{
	...
	error page 500 501 502 503 504 /50x.html;
	...
}

server{
	...	
				#错误页面
				location =/50x.html{
				root /code/；
				}
				#动态资源
        location ~ \.php${
                root /code;
                ...
        }
        #静态资源
        location ~ .*\.(jpg|gif|png|js|css)${
                root /code;
        }
        #拒绝请求后缀的文件
        location ~* "\.(sql|bak|tgz|tar.gz|git)$"{
                return 403 "启用访问控制";
        }
}


```







