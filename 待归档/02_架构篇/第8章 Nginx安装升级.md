##YUM安装

##### 1.修改官方yum源

nginx官网说明：<http://nginx.org/en/linux_packages.html#RHEL-CentOS>

如果不更改yum源则默认从阿里epel源下载，阿里源的nginx版本较旧，是1.12.0版本

```shell
[root@web01 ~]# vim /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0	//是否验证密钥，默认1，改为0
enabled=1
```

##### 2.yum安装

```shell
[root@web01 ~]# yum install -y nginx

#安装其它后续需要用到的包
[root@web01 ~] $ yum install -y gcc gcc-c++ autoconf pcre pcre-devel make automake wget httpd-tools vim tree
```

##### 3.启动并开机自启

```shell
[root@web01 ~]# systemctl start nginx
[root@web01 ~]# systemctl enable nginx
```

##### 4.看是否成功启动

```shell
#方法一：查进程
[root@web01 ~]# ps -ef |grep [n]ginx
root      17603      1  0 13:51 ?        00:00:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
nginx     17604  17603  0 13:51 ?        00:00:00 nginx: worker process

#方法二：看端口
[root@web01 ~]# netstat -lntup |grep 80
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      17603/nginx: master 

#方法三：systemd
[root@web01 ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since 二 2019-08-13 11:44:09 CST; 3h 33min ago
     Docs: http://nginx.org/en/docs/
 Main PID: 17603 (nginx)
   CGroup: /system.slice/nginx.service
           ├─17603 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─17604 nginx: worker process

8月 13 11:44:09 web01 systemd[1]: Starting nginx - high performance web server...
8月 13 11:44:09 web01 systemd[1]: PID file /var/run/nginx.pid not readable (yet?) ...rt.
8月 13 11:44:09 web01 systemd[1]: Started nginx - high performance web server.
Hint: Some lines were ellipsized, use -l to show in full.

#方法四：浏览器输入IP或域名，能看到nginx欢迎页面
查站点目录
[root@web01 ~]# rpm -ql nginx |grep html
/usr/share/nginx/html
/usr/share/nginx/html/50x.html
/usr/share/nginx/html/index.html

#方法五：curl命令
[root@web01 ~]# curl 10.0.0.7
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

##### 5.启动时报错

```shell
[root@web01 ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

#查看详细信息，发现端口被占用
[root@web01 ~]# systemctl status nginx.service
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since 二 2019-08-13 15:25:01 CST; 5min ago
     Docs: http://nginx.org/en/docs/
  Process: 17693 ExecStop=/bin/kill -s TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 17715 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=1/FAILURE)
 Main PID: 17603 (code=exited, status=0/SUCCESS)

8月 13 15:24:58 web01 nginx[17715]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98...se)
8月 13 15:24:59 web01 nginx[17715]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98...se)
8月 13 15:24:59 web01 nginx[17715]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98...se)
8月 13 15:25:00 web01 nginx[17715]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98...se)
8月 13 15:25:00 web01 nginx[17715]: nginx: [emerg] bind() to 0.0.0.0:80 failed (98...se)
8月 13 15:25:01 web01 systemd[1]: nginx.service: control process exited, code=exit...s=1
8月 13 15:25:01 web01 nginx[17715]: nginx: [emerg] still could not bind()
8月 13 15:25:01 web01 systemd[1]: Failed to start nginx - high performance web server.
8月 13 15:25:01 web01 systemd[1]: Unit nginx.service entered failed state.
8月 13 15:25:01 web01 systemd[1]: nginx.service failed.
Hint: Some lines were ellipsized, use -l to show in full.

#查端口是被httpd占用
[root@web01 ~]# netstat -lntup |grep 80
tcp6       0      0 :::80                   :::*                    LISTEN      17703/httpd 
#停止httpd服务再启动nginx
[root@web01 ~]# systemctl stop httpd
[root@web01 ~]# systemctl start nginx
```

##### 6.查看版本信息

```shell
#简单版本
[root@web01 ~]# nginx -v
nginx version: nginx/1.16.0

#详细版本信息（含模块）
[root@web01 ~]# nginx -V
nginx version: nginx/1.16.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
```

## 源码安装Nginx

##### 1.从官网nginx.org下载源码包

```shell
[root@local ~]# wget http://nginx.org/download/nginx-1.16.1.tar.gz
```

##### 2.提前安装依赖包

```shell
[root@local ~]# yum install -y c gcc pcre-devel openssl-devel zlib-devel
gcc: c环境
pcre-devel:支持正则
openssl: 支持https
zlib: 压缩支持
```

##### 3.编译并安装

```shell
[root@local ~]# tar xf nginx-1.16.1.tar.gz 
[root@local ~]# cd nginx-1.16.1
[root@local ~]# ./configure --prefix=/usr/local/nginx --user=www --group=www --with-http_ssl_module --with-http_stub_status_module
[root@local ~]# make &&make install
```

##### 4.配置nginx

```shell
[root@local ~]# vim /usr/local/nginx/conf/nginx.conf
usr www;
http{
    include /usr/local/nginx/conf/conf.d/*.conf;
}
[root@local ~]# mkdir /usr/local/nginx/conf/conf.d
```

##### 5.使用systemd管理nginx

```shell
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
#systemctl daemon-reload
#systemctl start nginx.service
#systemctl enable nginx.service
#systemctl status nginx.service
EOF
```

##### 6.做软链接,使用nginx命令

```shell
[root@local ~]# ln -s /usr/local/nginx/sbin/nginx  /usr/sbin/nginx
```

## RPM打包

##### 1.在源码安装前保存依赖文件

```shell
[root@local ~]# vim /etc/yum.conf 
keepcache=1
#清空yum缓存的rpm软件
[root@local ~]# yum clean packages
```

##### 2.拷贝缓存的软件并打包

```shell
[root@local ~]# find /var/cache/yum -name "*.rpm" |xargs cp -t /usr/local/src/nginx-1.16.0
[root@local ~]# tar czf nginx-1.16.0.tgz /usr/local/src/nginx-1.16.0
```

##### 3.安装fpm工具

```shell
#解压
[root@oldboy ~]# tar xf fpm-1.3.3.x86_64.tar.gz 
#此时gem命令用不了，需要先安装ruby才能用gem
[root@oldboy ~]# yum install -y ruby rubygems ruby-devel
#更换gem命令下载源，默认国外源，速度相当慢，可用华为源
[root@oldboy ~]# gem sources --add https://mirrors.huaweicloud.com/repository/rubygems/ --remove https://rubygems.org/
#更换成功
[root@oldboy ~]# gem sources --list
*** CURRENT SOURCES ***
https://mirrors.huaweicloud.com/repository/rubygems/
#安装fpm工具
[root@oldboy ~]# gem install *.gem
```

##### 4.打包

打包前需要安装rpm-build包，否则报错 ：Need executable 'rpmbuild' to convert dir to rpm {:level=>:error}

```shell
#安装rpmbuild
[root@oldboy ~]# yum install -y rpm-build

#打包
[root@oldboy ~]# fpm -s dir -t rpm -n nginx -v 1.16.0 -d 'pcre-devel,openssl-devel' -f /app/nginx-1.16.0/

#完成后生成一个rpm包
[root@oldboy ~]# ll |grep ngin
drwxr-xr-x. 9 1001 1001   186 7月  11 11:30 nginx-1.16.0
-rw-r--r--. 1 root root  1.4M 7月  11 17:25 nginx-1.16.0-1.x86_64.rpm
-rw-r--r--. 1 root root 1009K 4月  23 21:58 nginx-1.16.0.tar.gz

```

```shell
#fpm打包参数详解
-s          指定源类型
-t          指定目标类型，即想要制作为什么包
-n          指定包的名字
-v          指定包的版本号
-C          指定打包的相对路径  Change directory to here before searching forfiles
-d          指定依赖于哪些包
-f          第二次打包时目录下如果有同名安装包存在，则覆盖它
-p          输出的安装包的目录，不想放在当前目录下就需要指定
--post-install      软件包安装完成之后所要运行的脚本；同--after-install
--pre-install       软件包安装完成之前所要运行的脚本；同--before-install
--post-uninstall    软件包卸载完成之后所要运行的脚本；同--after-remove
--pre-uninstall     软件包卸载完成之前所要运行的脚本；同--before-remove

```



##### 5.安装测试

```shell


```







## nginx升级



## nginx回滚



## nginx添加模块

















