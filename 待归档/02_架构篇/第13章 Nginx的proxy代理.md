## 理论基础

##### 正向代理与反向代理的区别

正向代理：对象是客户端，为个人PC服务，用于内部上网 （由代理解析DNS）

反向代理：代理的对象是服务端，为服务器服务，用于公司集群架构中 (客户端自己解析DNS)

##### nginx支持的代理协议

| Nginx代理模块           | 支持的协议                                 |
| ----------------------- | ------------------------------------------ |
| ngx_http_proxy_module   | http、websocket、https、tomcat、Java等程序 |
| ngx_http_fastcgi_module | php程序                                    |
| ngx_http_uwsgi_module   | python程序                                 |
| ngx_http_http_v2_module | grpc（Golang程序）                         |

## proxy代理配置方法

##### 1、配置后端的web01

```shell
[root@web01 conf.d]# vim proxy.123.com.conf 
server{
        listen 80;
        server_name proxy.123.com;

        location /{
                root /proxy;
                index index.html;
        }
}
[root@web01 /etc/nginx/conf.d]# mkdir /proxy
[root@web01 /etc/nginx/conf.d]# echo "this is web01-10.0.0.7">/proxy/index.html
[root@web01 /etc/nginx/conf.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@web01 /etc/nginx/conf.d]# nginx -s reload
[root@web01 /etc/nginx/conf.d]# 
```

##### 2、nginx代理配置

```shell
[root@lb01 conf.d]# vim proxy_web.conf
server{
        listen 80;
        server_name proxy.123.com;

        location /{
                proxy_pass http://10.0.0.7:80;
                proxy_set_header Host $http_host;
        }
}
```

##### 3、抓包分析

![image-20190909091753099](https://tva1.sinaimg.cn/large/006y8mN6gy1g6t021to0bj30p806at9z.jpg)

![image-20190909092244798](https://tva1.sinaimg.cn/large/006y8mN6gy1g6t073upt6j30pc0kk0wv.jpg)

## 优化配置

##### 1、代理相关的参数

```shell
[root@lb01 /etc/nginx]# vim proxy_params

proxy_http_version 1.1;																					#代理向后端请求使用http 1.1版本
proxy_set_header Host $http_host;																#代理向后端请求时携带域名
proxy_set_header X-Real-IP $remote_addr;												#获取真实IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;		#获取全链路IP

proxy_connect_timeout 30;		#代理与后端web服务器连接超时时间，30s内没连上报错502
proxy_read_timeout 60;			#代理连上后端web服务器，但服务器响应超时的时间
proxy_send_timeout 60;			#代理连上web，也响应了，但回传数据给代理的超时时间

#代理把web回传的内容先放缓冲区，边接收边回传给客户端，而不是等接收完所有数据再统一回传
proxy_buffering on;					#开启缓冲区
proxy_buffer_size 32k;			#代理保存用户头信息的缓冲区大小
proxy_buffers 4 128k;				#代理保存回传内容的缓冲区大小


无注释代码：
proxy_http_version 1.1;
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_connect_timeout 30;
proxy_read_timeout 60;
proxy_send_timeout 60;
proxy_buffering on;
proxy_buffer_size 32k;
proxy_buffers 4 128k;
```

##### 2、代理配置location时调用

```shell
#方便后续多个Location重复使用
location / {
    proxy_pass http://127.0.0.1:8080;
    include proxy_params;
}
```

## proxy代理的弊端

```shell
一个location仅能代理后端一台主机，解决办法是用负载均衡，详情见第14章
```













