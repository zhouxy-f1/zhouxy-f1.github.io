## 配置文件分类

##### 主配置文件

| **路径**                       | **类型** | **作用**         |
| :----------------------------- | :------- | :--------------- |
| /etc/nginx/nginx.conf          | 配置文件 | nginx主配置文件  |
| /etc/nginx/conf.d/default.conf | 配置文件 | 默认网站配置文件 |

##### 代理参数

| **路径**                  | **类型**   | **作用**            |
| :------------------------ | :--------- | :------------------ |
| /etc/nginx/fastcgi_params | 代理php    | Fastcgi代理配置文件 |
| /etc/nginx/scgi_params    | 代理       | scgi代理配置文件    |
| /etc/nginx/uwsgi_params   | 代理python | uwsgi代理配置文件   |

##### 编码

| **路径**              | **类型** | **作用**              |
| :-------------------- | :------- | :-------------------- |
| /etc/nginx/win-utf    | 配置文件 | Nginx编码转换映射文件 |
| /etc/nginx/koi-utf    | 配置文件 | Nginx编码转换映射文件 |
| /etc/nginx/koi-win    | 配置文件 | Nginx编码转换映射文件 |
| /etc/nginx/mime.types | 配置文件 | Content-Type与扩展名  |

##### 命令

| **路径**              | **类型** | **作用**                  |
| :-------------------- | :------- | :------------------------ |
| /usr/sbin/nginx       | 命令     | Nginx命令行管理终端工具   |
| /usr/sbin/nginx-debug | 命令     | Nginx命令行与终端调试工具 |

##### 日志

| **路径**               | **类型** | **作用**              |
| :--------------------- | :------- | :-------------------- |
| /var/log/nginx         | 目录     | Nginx默认存放日志目录 |
| /etc/logrotate.d/nginx | 配置文件 | Nginx默认的日志切割   |

## 配置文件详解

##### 主配置文件

/etc/nginx/nginx.conf是一个纯文本类型的文件，整个配置文件是以区块的形式组织的。每个区块以一对大括号{}来表示开始与结束。整体分为三块：CoreModule(核心模块)、EventModule(事件驱动模块)、HttpCoreModule(http内核模块)

```shell

------------------------- CoreModule（核心模块） -------------------------------------
#正常运行必备
1、Nginx进程所使用的用户
user www;

2、Nginx服务运行后产生的pid进程号
pid /var/run/nginx.pid

3、指明包含进来的其它配置文件片断
nclude file | mask;

4、指明要装载的动态模块
load_module file;


#性能优化相关配置
1、子进程worker的数量，auto指当前物理主机CPU核心数，指定数字的话应该小于等于当前cpu核心数
worker_processes 1|auto;

2、cpu的内核跟worker绑定，可用lscpu查看
worker_cpu_affinity auto [cpumask];
补充：计算cpu mask方法（以四核为例）： 0001 指第0号cpu 0010 指第1号cpu 0100 指第2号cpu 1000第3号cpu

3、指定worker进程的nice值，设定worker进程优先级；[-20,20]
worker_priority number;

4、worker进程所能够打开的文件数量上限
worker_rlimit_nofile number;

#调度定位问题
1、是否以守护进程方式运行nginx
daemon on|off;

2、是否以master/worker模型运行nginx；默认为on；
master_process on|off;

3、错误日志
errot_log file [level];

4、Nginx错误日志存放路径
error_log /log/nginx/error.log

------------------------- EventModule(事件驱动模块) ----------------------------------
events {            
1、每个worker进程支持的最大连接数
    worker_connections 25535;   
    
2、事件驱动模型, epoll默认
    use epoll;
    注：select/poll 是遍历所有添加进fd_set的fd。并且需要将所有用户态的fd拷贝到内核态。数量巨大时这个效率比较慢
    
3、处理新连接请求的方法：on是各worker轮流处理新请求，off是所有worker都收到请求，由闲的worker来处理    
 		accept_mutex on|off
}

------------------------- HttpCoreModule(http内核模块) -------------------------------

#http层开始
http {
#包含资源类型文件
    include       /etc/nginx/mime.types;
#当资源在mime.types中无法找到时，默认下载的方式传输给浏览器
    default_type  application/octet-stream;
#日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
#指定访问日志 调用main格式
    access_log  /var/log/nginx/access.log  main;
#高效文件传输
    sendfile        on;
#搭配sendfile使用
    #tcp_nopush     on;
#长连接超时时间
    keepalive_timeout  65;
#是否开启压缩
    #gzip  on;

#包含/etc/nginx/conf.d/目录下所有以.conf结尾的文件，可用于配置虚拟主机
    include /etc/nginx/conf.d/*.conf;  
#http结束层 
} 

```

##### 子配置文件

/etc/nginx/conf.d/*.conf，被调用进主配置文件，默认的网站配置文件default.conf中，使用server配置网站，每个server{}代表一个网站，简称虚拟主机

```shell

1、配置虚拟主机开始层
server {
2、监听端口
		listen PORT|address[:port] 
		listen address[:port] [default_server] [ssl] [http2 | spdy]  [backlog=number] [rcvbuf=size] [sndbuf=size]
      				default_server：设定为默认虚拟主机；
	    				ssl：限制仅能够通过ssl连接提供服务；
	    				backlog=number：后援队列长度；
	    				rcvbuf=size：接收缓冲区大小；
	    				sndbuf=size：发送缓冲区大小；
    
3、虚拟主机的主机名，后可跟多个由空白字符分隔的字符串
    server_name  name ...;
    					支持*通配任意长度的任意字符；server_name *.magedu.com  www.magedu.*
							支持~起始的字符做正则表达式模式匹配；server_name ~^www\d+\.magedu\.com$
							匹配机制：
							(1) 首先是字符串精确匹配;
							(2) 左侧*通配符；
							(3) 右侧*通配符；
							(4) 正则表达式；
4、在keepalived模式下的连接是否启用TCP_NODELAY选项
		tcp_nodelay on | off;
		
5、是否启用sendfile功能
		sendfile on | off;
		
    
#调用字符集
    charset koi8-r;
#控制网站访问路径
   location / {
#存放网站源代码的位置
        root   /usr/share/nginx/html;
#默认返回网站的文件
        index  index.html index.htm;
    }
#服务端出错时默认返回页面
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

```

## 虚拟主机

##### 1.基于不同IP

```shell
[root@web01 /etc/nginx/conf.d]# vim ip.conf
server{
        listen 10.0.0.7:80;
        server_name _;

        location /{
                root /code/ip_10;
                index index.html;
        }
}
server{
        listen 172.16.1.7:80;
        server_name _;

        location /{
                root /code/ip_172;
                index index.html;
        }
}
[root@web01 /etc/nginx/conf.d]#mkdir /code/ip_{10,172} -p
[root@web01 /etc/nginx/conf.d]#echo "ip-10.0.0.7">/code/ip_10/index.html
[root@web01 /etc/nginx/conf.d]#echo "ip-172.16.1.7">/code/ip_172/index.html
[root@web01 /etc/nginx/conf.d]#nginx -s reload
#测试
[root@web02 ~]# curl 10.0.0.7
ip-10.0.0.7
[root@web02 ~]# curl 172.16.1.7
ip-172.16.1.7

```

##### 2.基于不同端口

一般在内部测试环境下使用

```shell
[root@web01 /etc/nginx/conf.d]# vim port.conf 
server{
        listen 81;

        location /{
                root /code/hosts_81;
                index index.html;
        }
}
server{
        listen 82;

        location /{
                root /code/hosts_82;
                index index.html;
        }
}
[root@web01 /etc/nginx/conf.d]# nginx -t 
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@web01 /etc/nginx/conf.d]# nginx -s reload
[root@web01 /etc/nginx/conf.d]# mkdir /code/hosts_{82,81} -p
[root@web01 /etc/nginx/conf.d]# echo "81"> /code/hosts_81/index.html
[root@web01 /etc/nginx/conf.d]# echo "82"> /code/hosts_82/index.html 
#浏览器访问验证
10.0.0.7:81
10.0.0.7:82
```

##### 3.基于不同域名

```shell
#写到不同的.conf文件中，便于管理维护各子网页
[root@web01 /etc/nginx/conf.d]# vim xxx.123.com.conf 
server{
        listen 80;
        server_name xxx.123.com;

        location /{
                root /code/xxx;
                index index.html;
        }
}
[root@web01 /etc/nginx/conf.d]# vim yyy.123.com.conf 
server{
        listen 80;
        server_name yyy.123.com;

        location /{
                root /code/yyy;
                index index.html;
        }
}
[root@web01 /etc/nginx/conf.d]#mkdir /code/{xxx,yyy} 
[root@web01 /etc/nginx/conf.d]#echo "xxx.123.com">/code/xxx/index.html
[root@web01 /etc/nginx/conf.d]#echo "yyy.123.com">/code/yyy/index.html
[root@web01 /etc/nginx/conf.d]#systemctl restart nginx

Macbook:~ zhouxy$ sudo vim /etc/hosts
10.0.0.7 xxx.123.com yyy.123.com

```

**常见错误：当域名解析错了，.conf文件不认识这个域名，但ip是本主机，服务器就会返回在conf.d目录下最上面的.conf文件**

## 日志管理

##### 内置变量

```shell

$remote_addr        # 记录上一跳的IP地址（非真实IP，是CDN或负载均衡的IP）
$remote_user        # 记录客户端用户名
$time_local         # 记录通用的本地时间
$time_iso8601       # 记录ISO8601标准格式下的本地时间
$request            # 记录请求的方法以及请求的http协议
$status             # 记录请求状态码(用于定位错误信息)
$body_bytes_sent    # 发送给客户端的资源字节数，不包括响应头的大小
$bytes_sent         # 发送给客户端的总字节数
$msec               # 日志写入时间。单位为秒，精度是毫秒。
$http_referer       # 记录从哪个页面链接访问过来的
$http_user_agent    # 记录客户端浏览器相关信息
$http_x_forwarded_for #记录客户端IP地址
$request_length     # 请求的长度（包括请求行， 请求头和请求正文）。
$request_time       # 请求花费的时间，单位为秒，精度毫秒
# 注:如果Nginx位于负载均衡器，nginx反向代理之后， web服务器无法直接获取到客 户端真实的IP地址。
# $remote_addr获取的是反向代理的IP地址。 反向代理服务器在转发请求的http头信息中，
# 增加X-Forwarded-For信息，用来记录客户端IP地址和客户端请求的服务器地址。

$connection_requests：TCP连接当前的请求数量 (1.3.8, 1.2.5)
$remote_addr：客户端地址
$remote_port：客户端端口
$hostname：主机名
$request_filename：当前连接请求的文件路径，由root或alias指令与URI请求生成。
$server_addr：服务器端地址，需要注意的是：为了避免访问linux系统内核，应将ip地址提前设置在配置文件中。
$server_name：服务器名，www.cnphp.info
$server_port：服务器端口
```

##### 定义格式

```shell
#在主配置文件/etc/nginx/nginx.conf里的http模块设置
log_format ipaddr '$remote_addr - $http_x_forwarded_for'
```

##### 调用日志

```shell
#可用于http层、server层、locaton层、if in location层、limit_except层等
access_log /var/log/nginx/www.123.com.log aaa;
```

##### 我的日志

```shell
log_format  main  '客户端IP:$http_x_forwarded_for 上层IP：$remote_addr 客户端端口：$remote_port '
									'请求的文件：$request_filename '
                  '返回状态码：$status "从哪跳转来：$http_referer" '
                  '服务端地址：$server_addr 服务器端口：$server_port';
```

##### 日志作用域

```shell
当在http层配置并调用了main日志
主机A和B没有调用日志，C调用了日志，则C的日志会独立出来到调用时指定的位置，A和B的日志还在http层调用时指定的位置

#一个server里配置一个日志就够了，不要在location里再配置日志，否则会非常乱
```

##### 不记录日志

```shell
#当用户请求ico图标时不记录日志
location /favicon.ico {
	access_log off;
}
```



##### 日志切割

yum安装的nginx会自动做切割，把access.log文件移动改名为2019_9_5.access.log，自动切割的配置文件如下

```shell
[root@web01 ~]# cat /etc/logrotate.d/nginx 
/var/log/nginx/*.log {
        daily									//每天切割
        missingok							//日志丢失忽略
        rotate 52							//保留52天的日志，52天之前的自动删除
        compress							//日志文件压缩，为.gz格式
        delaycompress					//延迟压缩
        notifempty						//不切割空文件
        create 640 nginx adm	//日志权限640，属主nginx 属组adm
        sharedscripts
        postrotate						//切割日志前执行命令：移动改名完成后重载access.log文件，否则新日志进不来	
                if [ -f /var/run/nginx.pid ]; then
                        kill -USR1 `cat /var/run/nginx.pid`
                fi
        endscript
}
```

