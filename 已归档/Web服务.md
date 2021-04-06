## 一、HTTP

##### 相关概念

```bash
http(hyper text transfer protocol): 超文本传输协议，常见超文本标记语言是HTML
URL(uniform resource locator): 统一资源定位符，用于标识网络中的某个文件。格式为：协议://主机:端口/文件名及路径
```

##### 用户访问架构流程

```bash
浏览器请求 => CDN => 防火墙 => LB => WEB => NFS => REDIS => MYSQL

1.客户端发起http请求，请求会先抵达CDN，CDN未命中则回源到LB，再经前端的防火墙
2.防火墙识别用户身份，正常的请求通过内部交换机通过tcp连接后端的负载均衡，传递用户的http请求
3.负载接收到请求，会根据请求的内容进行下发任务，通过tcp连接后端的web，转发用户的http请求
4.web接收到用户的http请求后，会根据用户请求的内容进行解析，解析分为如下：
    静态请求:web直接返回给负载均衡->防火墙->用户
    动态请求:web向后端的动态程序建立TCP连接，将用户的动态http请求传递至动态程序->由动态程序进行解析
5.动态程序在解析的过程中，如果碰到查询数据库请求，则优先与缓存建立tcp连接，并发起数据查询操作。
6.如果缓存没有对应的数据，动态程序再次向数据库建立tcp连接，并发起查询操作。
7.最后数据由, 数据库->动态程序->缓存->web服务->负载均衡->防火墙->用户
```

##### http协议工作原理

```bash
1.用输入域名 -> 浏览器跳转 -> 浏览器缓存 -> Hosts文件 -> DNS解析（递归查询|迭代查询）
    客户端向服务端发起查询 -> 递归查询
    服务端向服务端发起查询 -> 迭代查询
2.由浏览器向服务器发起TCP连接（三次握手）
    客户端     -->请求包连接 -syn=1 seq=x           服务端
    服务端     -->响应客户端syn=1 ack=x+1 seq=y     客户端
    客户端     -->建立连接 ack=y+1 seq=x+1          服务端
3.客户端发起http请求：
    1）请求的方法是什么:     GET获取
    2）请求的Host主机是:    www.zlwm115.com
    3）请求的资源是什么:     /index.html
    4）请求的端端口是什么:    默认http是80 https是443
    5）请求携带的参数是什么:  属性（请求类型、压缩、认证、浏览器信息、等等）
    6）请求最后的空行
4.服务端响应的内容是
    1）服务端响应使用WEB服务软件
    2）服务端响应请求文件类型
    3）服务端响应请求的文件是否进行压缩
    4）服务端响应请求的主机是否进行长连接
5.客户端向服务端发起TCP断开（四次挥手）
    客户端     --> 断开请求 fin=1 seq=x          -->    服务端
    服务端     --> 响应断开 fin=1 ack=x+1 seq=y  -->    客户端
    服务端     --> 断开连接 fin=1 ack=x+1 seq=z  -->    客户端
    客户端     --> 确认断开 fin=1 ack=x+1 seq=sj -->    服务端
```

##### http报文

```bash
#请求方法
GET: 			请求读取一个web页面
POST: 		请求向指定的资源提交要被处理的数据
DELETE: 	请求删除web页面
CONNECT:	用于代理服务器
HEAD:		 	请求读取一个web页面的头部信息
PUT: 			请求存储一个web页面
TRACE: 		用于测试，要求服务器送回收到的请求
OPTION: 	查询特定的选项

#返回状态码
100: 客户端应当继续提出请求。 服务器返回此代码表示已收到请求的第一部分，正在等待其余部分
101: 客户端已要求服务器切换协议，服务器已确认并准备切换
200: 服务器已成功处理了请求
201: 请求成功并且服务器创建了新的资源
202: 服务器已接受请求，但尚未处理
203: 服务器已成功处理了请求，但返回的信息可能来自另一来源
204: 服务器成功处理了请求，但没有返回任何内容
205: 服务器成功处理了请求，但已重置内容
206: 服务器成功处理了部分GET请求
300: 针对请求，服务器可执行多种操作。服务器可根据请求选择一项操作，或提供操作列表供请求者选择
301: 永久重定向，用户首次访问A域名后，A回复跳转，浏览器记录下来放缓存。下次访问不再询问A，直接跳转到B,除非清缓存
302: 临时重定向，浏览器不记录跳转信息，每次都会访问A，A回复让跳转后再跳转
303: 查看其他位置，请求者应当对不同的位置使用单独的GET请求来检索响应时，服务器返回此代码
304: 用户请求的内容在浏览器缓存中，像脚本、字体、图片随时会执行所以放内存中；css等非脚本文件只用一次所以放在磁盘里
305: 请求者只能使用代理访问请求的网页。 如果服务器返回此响应，还表示请求者应使用代理
307: 内部重定向，服务器目前从不同位置的网页响应请求，但请求者应继续使用原有位置来进行以后的请求
400: 错误请求，服务器不理解请求的语法
401: 认证失败，web服务器要求的auth_basic_user_file认证不通过，没有输入正确的用户名密码
403: 权限不足（被deny了）或找不到主页，在虚拟主机server{}上指定的默认主页文件index.html在站点目录中未找到
404: 找不到用户请求的页面，此页面在站点目录下用户指定的目录里不存在
405: 禁用请求中指定的方法
406: 无法使用请求的内容特性响应请求的网页
407: 需要代理授权，此状态代码与 401（未授权）类似，但指定请求者应当授权使用代理
408: 服务器等候请求时发生超时
409: 服务器在完成请求时发生冲突。 服务器必须在响应中包含有关冲突的信息
410: 如果请求的资源已永久删除，服务器就会返回此响应
411: 服务器不接受不含有效内容长度标头字段的请求
412: 服务器未满足请求者在请求中设置的其中一个前提条件
413: 请求实体过大，超出服务器的处理能力
414: 请求的 URI（通常为网址）过长，服务器无法处理
415: 请求的格式不受请求页面的支持
416: 请求范围不符合要求
417: 服务器未满足"期望"请求标头字段的要求
500: 服务器内部错误，一般是代码问题，导致连不上数据库
501: 尚未实施，服务器不具备完成请求的功能。 例如，服务器无法识别请求方法时可能会返回此代码
502: 找不到后端主机，负载均衡无法通过tcp连接上web服务器
503: 用户请求过多导致服务器过载，或用户请求频率超过上限返回503；也可自定义停机维护页的状态码为503
504: 网关超时，负载均衡能通过tcp连接到后端web服务器，但服务器返回信息太慢，导致超过
505: HTTP版本不受支持，服务器不支持请求中所用的 HTTP 协议版本
```

## 二、Nginx

##### 安装配置

```bash
#yum安装并启动
cat>/etc/yum.repos.d/nginx.repo<<EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/6/$basearch/
enabled=1
module_hotfixes=true
EOF
yum list|grep nginx
yum install -y nginx
service nginx start
chkconfig nginx on

#编译安装
1.下载安装
yum install -y gcc-c++ gcc pcre-devel openssl-devel zlib-devel
tar xf nginx-1.16.1.tar.gz
cd nginx-1.16.1
./configure --prefix=/usr/local/nginx --user=www --group=www --with-http_ssl_module --with-http_stub_status_module
make && make install
2.使用systemd管理
cat>/usr/lib/systemd/system/nginx.service<<"EOF"
[Unit]
Description=nginx server daemon
Documentation=man:nginx(8)
After=network.target
[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl start nginx.service
systemctl enable nginx.service
systemctl status nginx.service
3.软链接
ln -s /usr/local/nginx/sbin/nginx  /usr/sbin/nginx

#nginx相关文件
[root@web1 nginx]# rpm -ql nginx
/etc/nginx/fastcgi_params					#反代php的配置
/etc/nginx/scgi_params						#反代
/etc/nginx/uwsgi_params						#反代python的配置文件
/usr/sbin/nginx										#命令行管理工具
/usr/sbin/nginx-debug							#终端调度工具
/etc/logrotate.d/nginx						#日志切割配置文件	
/etc/nginx/koi-utf								#nginx编码转换映射文件
/etc/nginx/koi-win								#nginx编码转换映射文件
/etc/nginx/win-utf								#nginx编码转换映射文件
/etc/nginx/mime.types							#配置http协议中的content-type与扩展名对应关系

#详解主配置文件
[root@web1 nginx]# vim nginx.conf 
user  nginx;																			#nginx进程使用的用户
worker_processes  1;															#worker进程数量，数量跟cpu核心数一致，一般用auto做cpu亲和
error_log  /var/log/nginx/error.log warn;					#错误日志存放路径
pid        /var/run/nginx.pid;										#服务运行后产生的pid进程号
events {
    worker_connections  1024;											#每个worker进程支持的最大连接数
}
http {
    include       /etc/nginx/mime.types;					#包含的资源类型文件
    default_type  application/octet-stream;				#如果用户请求的资源在mime.type中无法找到，则下载文件
    log_format  main   ...日志格式...							 #定义日志显示样式
    access_log  /var/log/nginx/access.log  main;	#访问日志路径
    sendfile        on;														#开启高效文件传输
    #tcp_nopush     on;														#配合sendfile使用
    keepalive_timeout  65;												#长连接超时时间65秒
    #gzip  on;																		#开启压缩
    include /etc/nginx/conf.d/*.conf;							#主配置文件中包含conf.d下所有以.conf结尾的文件
}

#虚拟机常规配置
每一个server{}代表一个网站，简称虚拟机。
如果用户访问的域名在所有.conf文件中都没有，服务器默认使用conf.d目录里最上的面.conf文件

vim /etc/nginx/conf.d/longines.cn.conf
server{
	listen 80;																#仅配置监听端口
	listen 10.0.20.51:80 default_server ssl;	#设定为默认虚拟主机，ssl是限制仅能通过ssl连接提供服务
	server_name *.longines.cn;								#支持通配符*表示任意长度的任意字符
	server_name ~^www\d+\.zlwm115\.com$				#支持~表示区分大小写
	root /code/wordpress;											#如果在此配置root，则未配置root的location默认使用这个
	index index.php;													#效果同上
	location {
			root /usr/share/nginx/html;
			index index.php index.html;
	}
	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
			root /usr/share/nginx/html;
	}
}

#配置多套业务
可在同一台nginx服务器上跑多套业务系统，可基于不同ip、不同端口、不同域名实现
1.基于不同域名
vim /etc/nginx/conf.d/invoice.longines.cn.conf 
server {
        listen 80;
        server_name invoice.longines.cn;
				root /data/www/site/invoice/public;
				index index.php;
}
vim /etc/nginx/conf.d/invoice.swatch.cn.conf 
server {
        listen 80;
        server_name invoice.swatch.cn;
				root /data/www/site/invoice/public;
				index index.php;
}

2.基于不同端口
vim /etc/nginx/conf.d/zwlw115.com_80.conf
server{
	listen 80;
	server_name localhost;
}
vim /etc/nginx/conf.d/zlwm115.com_8080.conf
server{
	listen 81;
	server_name localhost;
}
3.基于不同ip，一般不用
```

##### 日志配置

```nginx
在主配置文件nginx.conf的http层定义，可在http层、server层、location层、if in location层、limit_execpt层调用
一般在server层调用就够了，不要到location里再调用，否则日志会很乱

#定义日志格式
log_format  main  '$remote_addr - $request_time - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

log_format jonny  '$time_local return [$status] from_port:$remote_port '
									'to_port: $server_port user:$remote_user request_file: $request_filename '
         				  'real_ip:$http_x_forwarded_for  last_ip: $remote_addr ';
#调用或关闭日志
access_log /var/log/nginx/www.longines.cn_access.log main;
access_log off;

#日志常见内置变量
$time_local         	#记录通用的本地时间，格林威治时间+8
$remote_addr       		#记录上一跳如CDN或LB的ip
$http_x_forwarded_for	#客户端真实IP
$remote_port					#客户端端口
$remote_user        	#客户端用户名,登陆状况的会显示
$http_user_agent    	#客户端浏览器相关信息
$request            	#客户端请求的方法以及请求的http协议
$request_filename			#客户端请求的文件，由root或alias指令与URI请求生成
$request_length     	#客户端请求的长度，包括请求行，请求头和请求正文
$request_time       	#客户端请求花费的时间，单位为秒，精度毫秒
$http_referer       	#记录从哪个页面链接访问过来的
$status             	#服务器返回的状态码
$body_bytes_sent    	#发送给客户端的资源字节数，不包括响应头的大小
$bytes_sent         	#发送给客户端的总字节数
$msec               	#日志写入时间。单位为秒，精度是毫秒
$connection_requests	#TCP连接当前的请求数量 (1.3.8, 1.2.5)
$hostname							#web服务器的主机名
$server_addr					#web服务器端地址，需要注意的是：为了避免访问linux系统内核，应将ip地址提前设置在配置文件中
$server_name					#web服务器，当前访问的域名
$server_port					#web服务器端口80
```

##### 基础模块

```bash
vim /etc/nginx/conf.d/www.zlwm115.com.conf
#server优先级：nginx有多个相同server_name时，按在conf.d目录里上自而下的顺序来读取.conf文件
server{
        listen 80;
        server_name zlwm115.com;
        root /usr/share/nginx/html;
        index index.html;
1.连接限制模块        
				limit_conn max_conn 500;									#调用连接数限制，只有nginx读取了整个请求头才计算进连接数
				limit_req zone=max_req burst=5 nodelay;		#调用访问频率限制模块，延迟处理5个请求，其余请求无延迟
				limit_req_status 444;											#自定义由于访问频率限制产生的状态码，不定义则默认是503
				error_page 444 /err.html;									#接收到444状态码返回的错误页面,err.html存于站点目录下
				#注: 在server层只是调用，需要在http层定义
				limit_conn_zone $remote_addr zone=max_conn:1m;	
				limit_req_zone $binary_remote_addr zone=max_req:10m rate=10r/s;	#10M内存空间可存放16万个会话
				
				location / {
2.过滤模块，403
								deny 192.168.1.0/24;						#根据上一跳的ip来设置过滤规则，自上而下匹配，拒绝则报错403
								allow all;											#弊端：只管上一跳无法规避vpn或slb或代理服务器的影响
				}
        location /download {
                alias /code/mysoftware;					#alias路径是匹配到/download请求后，直接到/code下找文件
3.自动索引模块
								autoindex on;										#打开自动索引
                autoindex_exact_size off;				#不用kb为单位显示大小，默认on，改off后带单位是KB、MB、GB
                autoindex_localtime on;					#使用本地时间
                charset utf-8,gbk;							#解决中文乱码问题
3.访问验证模块，401
                auth_basic "your passwd please";#开启访问密码验证并设置提示语，密码错误则报401错误
                auth_basic off;									#关闭密码验证
                auth_basic_user_file /etc/nginx/htpasswd;	#指定密码文件，用htpasswd生成的密码才生效
								#注: 需要提交准备密码文件，在命令行操作如下
								yum install -y httpd-tools
                htpasswd -c -b /etc/nginx/htpasswd zhouxy zhou123		#-c创建密码文件 -b密码写在命令行
                htpasswd /etc/nginx/htpasswd zhouxy									#新建用户，交互式输入密码
                htpasswd -D /etc/nginx/htpasswd zhouxy							#删除用户和密码3
								htpasswd -d /etc/nginx/htpasswd zhouxy							#修改用户密码
        }
4.状态监控模块        
        location /status {
                stub_status;										#开启状态监控模块
                keepalive_timeout 0;						#关闭长连接。一次tcp连接可发起多次http请求
                keepalive_timeout 65;						#65秒没活动则断开连接
                allow 127.0.0.1;								#只允许本地通过回环口访问
                deny all;												#无法通过浏览器访问status
        				#注: 状态详解如下
								Active connections:2    表示Nginx当前活跃连接数，是writing+waiting的总和
								server									表示Nginx启动到现在共处理了16个连接
								accepts									表示Nginx启动到现在共成功创建16次握手。请求丢失数=握手数-连接数
								handled requests				表示总共处理了19次请求
								Reading     						Nginx读取到客户端的 Header 信息数
								Writing     						Nginx返回给客户端的 Header 信息数
								Waiting    							Nginx开启keep-alive长连接情况下, 既没有读也没有写, 建立连接情况
        }
5.location匹配规则        
        location =/50x.html {
        			字符完全匹配，优先级最高，适用于配置错误页面;
        			root /usr/share/nginx/html;
        }
        location ^~ /images {
        			字符前缀匹配，优先级第二，不检查正则;
        			default_type text/html;
        			return 200 "location ^~ /images";
        }
        location ~ /images* {
        			正则匹配，优先级低于字符匹配，启用正则必须以~*或~或开头，~*表示不区分大小写，~是区分大小写;
        }
        location ~* \.php$ {
        			正则匹配，匹配上之后会继续查找更精确的location;
        }
        location ~* \.(gif|png|jpg|css|js)$ {
        			正则匹配，最终会走与正则表示最精确的那个location;
        			expires 3d;
        }
        location /{
        			优先级最低，当正则与字符串都未匹配到时，都走本location;
        }
        
6.try_files的用法
        location / {
        			try_files $uri /index.php @php;	#逐个检查请求的文件，存在则返回第一个找到的，不存在则返回@php
        }
        location @php {												#必须定义last的内部重定向，否则产生内部错误
        			fastcgi_pass 127.0.0.1:9000;
        			fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        			include fastcgi_params;        			
        }
}
```

##### 正向代理

```bash
正向代理：代理客户端向服务端发起请求，例如：dns，vpn等。客户端使用正向代理访问nginx服务器可突破deny的访问限制

vim /etc/nginx/

```

##### 反向代理

```bash
反向代理：代理服务端，解决服务端处理不了的问题

1.nginx 代理 http			实现超文件传输协议
2.nginx 代理 https		实现https访问
3.nginx 代理 TCP			实现端口转发
4.nginx	代理 POP/IMAP	实现邮件收发
5.nginx	代理 RTMP			实现流媒体、直播、点播等
6.nginx	代理 GRPC			实现go语言远程过程调用

```

##### 动静分离

##### 七层负载

```bash

```

##### 四层负载

##### nginx高可用

##### 配置证书

##### 限制爬虫

##### 拒绝ip

## 三、php

## 四、tomcat





















