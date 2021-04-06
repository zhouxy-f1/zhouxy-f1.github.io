## Rewrite概述

##### 1.什么是rewrite

​	Rewrite主要实现url地址重写以及重定向，把传入web的请求重定向到其它url的过程

##### 2.rewrite的使用场景

```shell
地址跳转：用户访问 www.abc.com/login时，重定向到login.abc.com;	
协议跳转：用户用http协议时，跳转到https协议；
伪静态：减少动态url地址对外暴露过多参数，提升安全性；（一般由开发写好，运维直接用）
搜索引擎收录：简单的url更利于收录。
```

##### 3.基本语法

```shell
Syntax: rewrite regex replacement [flag];
Default: --
context: server ,location,if

#切换维护页时
rewrite ^(.*)$ /page/index.html break;

#rewrite可应用于server \ location。
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
#访问blog.com域名直接跳转到百度
                rewrite ^(.*)$ https://www.baidu.com redirect;
        }
}
```

##### 4.flag标记

##### break与last区别

```shell
#配置文件
[root@web01 /etc/nginx/conf.d]# cat rewrite.com.conf 
server{
        listen 80;
        server_name rewrite.com;
        root /site;

        location ~ ^/break {
                rewrite ^/break /test1/ break;
        }
        location ~ ^/last {
                rewrite ^/last /test2/ last;
        }
        location /test2/ {
                root /site/XXX/;
        }
}
#站点目录
[root@web01 /site]# tree
.
├── index.html
├── test1
│   └── index.html
├── test2
│   └── index.html
└── XXX
    └── test2
        └── index.html

4 directories, 4 files
```

break：只要匹配到规则，则会去本地路径的目录中寻找请求的文件

last：只要匹配到规则，会对其所在的server{……}标签重新发起请求

![image-20191018073444913](https://tva1.sinaimg.cn/large/006y8mN6gy1g8208rdq80j30u303yt98.jpg)

![image-20191018073525164](https://tva1.sinaimg.cn/large/006y8mN6gy1g8209foa0pj30u404bjrx.jpg)

![image-20191018073603008](https://tva1.sinaimg.cn/large/006y8mN6gy1g820a3j7ajj30u304574u.jpg)

##### redirect与permanent区别

区别在于浏览器会不会记录跳转状态,redirect不记录，每次先找服务器，再跳转；permanent会记录，第二次开始直接跳转

```shell
#	redirect：临时跳转，每次请求都会询问服务器，服务器不存在时跳转会失败 
# permanent：永久跳转，只有第一次请求会询问服务器，浏览器记录跳转地址，之后不再询问，直接通过缓存的地址跳转
server{
        listen 80;
        server_name rewrite.123.com;
        root /code;

        location /aa {
                #rewrite ^(.*)$ https:www.baidu.com redirect;
                rewrite ^(.*)$ https:www.baidu.com permanent;
        }
}

#关闭nginx服务测试结果
redirect:关闭nginx后，无法跳转，报错无法连接
permanent:关闭nginx后，依然可以跳转
```

##### 5.Rewrite用法

###### 例1：访问www.rewrite.com/r，实际跳转到www.rewrite.com/x/y/z/index.html

```shell
 [root@web01 /etc/nginx/conf.d]# cat www.rewrite.com.conf 
server{
        listen 80;
        server_name www.rewrite.com;
        root /code;
        index index.html;

        location /r {
                rewrite (.*) /x/y/z/index.html redirect;
        }
}
[root@web01 /etc/nginx/conf.d]# tree /code
/code
├── index.html
└── x
    └── y
        └── z
            └── index.html

3 directories, 2 files
[root@web01 /code]# cat index.html 
/code/index.html
[root@web01 /code]# cat x/y/z/index.html 
/code/x/y/z/index.html
```

![image-20191020122307620](https://tva1.sinaimg.cn/large/006y8mN6gy1g84jw20kh8j30sq04emxn.jpg)

![image-20191020122517398](https://tva1.sinaimg.cn/large/006y8mN6gy1g84jvp28ujj30ys07wab4.jpg)

###### 例2：访问www.rewrite.com/2018，实际跳到www.rewrite.com/2014/x/y/z/index.html

```shell
[root@web01 /etc/nginx/conf.d]# cat www.rewrite.com.conf 
server{
        listen 80;
        server_name www.rewrite.com;
        root /code;
        index index.html;

        location /2014 {
                alias /code/2014/x/y/z;
        }
        location /2018 {
                rewrite ^/2018/(.*)$  /2014/$1 redirect;
        }
}
把/code/2018/x/y/z/index.html  转到/code/2014/x/y/z/index.html
```

![image-20191020133751117](https://tva1.sinaimg.cn/large/006y8mN6gy1g85lxle71ej30ya050wf0.jpg)

![image-20191020133645597](https://tva1.sinaimg.cn/large/006y8mN6gy1g84lyis5jsj30yx07y75c.jpg)

###### 例3：访问www.rewrite.com/test，跳转到百度

```shell
        location /test {
                rewrite ^(.*)$ http://www.baidu.com redirect;

#必须要加http:// 否则跳到www.rewrite.com/www.baidu.com
```

###### 例4：用户访问course-11-22-33.html，实际访问到/course/11/22/33/course_33.html

```shell
[root@web01 /etc/nginx/conf.d]# cat www.rewrite.com.conf 
server{
        listen 80;
        server_name www.rewrite.com;
        root /code;
        index index.html;

        location /test {
                rewrite ^(.*)$ www.baidu.com redirect;
        }
        location ~  ^/course {
                rewrite course-11-22-33.html /course/11/22/33/course_33.html break;
#               rewrite (.*)-(.*)-(.*)-(.*)\.(.*) /$1/$2/$3/$4/$1_$4.$5 break;
        }
}
```

![image-20191021102103049](https://tva1.sinaimg.cn/large/006y8mN6gy1g85lwo8cnfj30yb07umyp.jpg)

###### 例5：把http请求跳转至https

```shell
[root@web01 /etc/nginx/conf.d]# cat www.rewrite.com.conf 
server{
        listen 80;
        server_name www.rewrite.com;
        rewrite ^(.*) https://$server_name$1 redirect;
        return 302 https://$server_name$request_uri;
}
server {
        listen 443;
        server_name www.rewrite.com;
        ssl on;
}
```

























