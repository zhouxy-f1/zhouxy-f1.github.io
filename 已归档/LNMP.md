## 一、LNMP环境搭建

`LNMP=linux+nginx+mysql+php`

##### 安装启动php

```bash
#安装
rpm -ivh https://rpms.remirepo.net/enterprise/remi-release-7.rpm
vim /etc/yum.repos.d/remi-php73.repo		==>enable
yum install -y php.x86_64
php -v
php -m
yum install -y php.x86_64 php-fpm php-bcmath.x86_64 php-gd.x86_64 php-pecl-imagick-devel.x86_64 php-pecl-imagick.x86_64 php-intl.x86_64 php-mbstring.x86_64 php-pecl-zip.x86_64 php-xml.x86_64 php-mysqlnd.x86_64 php-xml.x86_64 php-soap.x86_64 php-pecl-zip.x86_64 php-opcache.x86_64 php-pecl-redis5.x86_64 php-imap.x86_64 php-json.x86_64 php-pecl-mcrypt.x86_64  mhash-devel

#启动
systemctl start php-fpm
systemctl enable php-fpm

注意：centos6安装指定版本php，最高到php5.3
```

##### 安装配置nginx

```bash
#安装nginx
http://nginx.org/en/linux_packages.html#RHEL-CentOS		#根据要求安装指定版本的nginx，从官网找对应的repo
yum install -y nginx
nginx -v
systemctl start nginx
systemctl enable nginx
浏览器输入web的公网ip：http://101.200.162.214/	看到主页就行了

#配置nginx反代php
vim /etc/nginx/conf.d/www.123.com.conf
server {
        listen 80;
        server_name _;
        root /code;

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

#测试nginx反代php
vim /code/index.php
<?php
		phpinfo();
?>
浏览器输入：http://101.200.162.214/ 看到php详细信息则反代成功
```

##### 安装mysql

```bash
#安装
wget http://repo.mysql.com/mysql-community-release-el6-5.noarch.rpm
rpm -ivh mysql-community-release-el6-5.noarch.rpm
yum install -y mysql-community-server
/etc/init.d/mysqld start

或者使用mariadb代替
yum install -y mariadb-server
systemctl start mariadb
systemctl enable mariadb

mysql
>set password=password('123') 
>create database wordpress;
>grant all on wordpress.* to wp@'172.16.1.%' identified by '111';
>exit

#在web上测试wp用户远程连接mysql
mysql -uwp -h 172.16.1.186 -p

#测试php连mysql
vim /code/mysql.php
<?php
        $servername="172.16.1.51";
        $username="wp";
        $password="111";
        //创建连接
        $conn = mysqli_connect($servername,$username,$password);
        //检测连接
        if (!$conn) {
                die ("Connection failed:".mysqli_connect_error());
        }
        echo "php连接mysql数据库成功";
?>

浏览器输入：http://101.200.162.214/mysql.php  看到连接成功
```

##### wordpress优化

```bash
#更新主题时限制上传文件大小为2M
1.nginx上server层默认为1m,改为100m
[root@web01 /etc/nginx/conf.d]# vim blog.123.com.conf 
server{
	client_max_body_size 100m;
}

2.php.ini的配置
[root@web01 /code/wordpress]# vim /etc/php.ini 
upload_max_filesize = 200M
post_max_size = 200M

#下载插件或主题
vim /code/wordpress/wp-config.php
文末添加如下代码：
define("FS_METHOD","direct");
define("FS_CHMOD_DIR",0777);
define("FS_CHMOD_FILE",0777);

```

## 二、nginx进阶

##### 基础模块

```nginx
#自动索引
location /download {
        alias /autoindex;								#指定目录列表的根目录
        autoindex on;										#开启目录列表
        autoindex_exact_size off;				#on是显示字节，off是带单位显示
        autoindex_localtime on;					#on是本地时间，off是伦敦时间
}
        charset utf-8,gbk;							#放在server里，解决中文件乱码问题

#状态监控
 location /status{
        stub_status;
        allow 127.0.0.1;
        deny all;
}

#密码登陆认证
location /download {
        auth_basic "username and password please! ";
        auth_basic_user_file /etc/nginx/htpwd;
}
yum install -y httpd-tools
htpasswd  -c -b /etc/nginx/htpwd zhouxy 123
cat /etc/nginx/htpwd

#限制用户请求频率
limit_req_zone $binary_remote_addr zone=max_req:10m rate=1r/s;		#先在nginx.conf里http层定义
limit_conn max_conn 1;																						#再去server或location层调用
limit_req zone=max_req burst=5 nodelay;		#允许延迟处理5个请求；nodelay是其它全部无延迟拒绝
limit_req_status 444;											#自字义状态码
error_page 444 /err_444.html;							#接收到444，后返回444错误页面给用户，返回时重新加载

#location模块匹配优先级
= 高于 ^*或*$  高于 ~或~* 高于 !~或!~* 高于 / #带*号的都不区分大小写 ~表示区分大小写，加*就不区分了 
location =/50x.html
location ~ \.php$
location ~ .*\.(jpg|gif|png|js|css)$
location ~* "\.(sql|bak|tgz|tar.gz|git)$"
```

##### 七层负载

```nginx
upstream longines{
		server 172.16.1.7:80 weight=3;
  	server 172.16.1.8:80 weight=3;
  	server 172.16.1.9:80 down;
}
server{
  	listen 80;
  	server_name www.123.com;
}
location /{
  proxy_pass http://logines;
  proxy_set_header HOST $http_host;
  proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
}
```

##### 四层负载

```nginx
vim /etc/nginx/conf.c/lb4.conf
stream{
  upstream www.123.com{
    	server 172.16.1.5:80 weight=5 max_fails=3 fail_timeout=30s;
    	server 172.16.1.6:80 weight=5 max_fails=3 fail_timeout=30s;
  }
  server{
    	
  }
}
```

##### 会话共享

```bash
#安装redis
yum install -y redis 
vim /etc/redis.conf
bind 127.0.0.1 172.16.1.51

#配置php连redis
vim /etc/php.ini
session.save_handler = redis
session.save_path = "tcp://172.16.1.51:6379"
session.auto_start = 1
vim /etc/php-fpm.d/www.conf
;php_value[session.save_handler] = files
;php_value[session.save_path]    = /var/lib/php/session
```

##### rewrite

```nginx
location /2018{
  rewrite (.*) /x/y/z/index.html redirect;
}
location /test{
  rewrite ^(.*)$ www.baidu.com redirect;
}
```

##### 配置https

```bash
#nginx配置
vim /etc/nginx/conf.d/www.zlwm115.com.conf
server {
        listen 80;
        server_name www.zlwm115.com;
        return 301 https://$server_name$request_uri;		
        rewrite ^(.*) https://$server_name$1 permanent;	#作用同上，只需要用一个
}
server {
        listen 443 ssl;
        server_name www.zlwm115.com;
        root /code/wordpress;
        ssl_certificate ssl/www.zlwm115.com.crt;
        ssl_certificate_key ssl/www.zlwm115.com.key;
				ssl_session_cache shared:SSL:10m;							#首次连接后10分钟内不需要再连接
				ssl_session_timeout 1440m;										#连接断开后的超时时间
				ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
				ssl_protocols TLSv1 TLSv1.1 TLSv1.2;					#可用TLS版本
				ssl_prefer_server_ciphers on;									#由nginx决定跟浏览器的通讯协议

        location / {
                index index.php index.html;
        }
        location ~ \.php$ {
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                fastcgi_param HTTPS on;
                include fastcgi_params;
        }
}

#查看证书
openssl x509 -in xxx.crt -noout -dates					#x509是给CA颁发的证书专用 -in指定证书 -noout不输出
-dates				查看证书过期时间
-text					查看证书内容
-serial				查看证书序列号
-subject			查看拥有者名字
-fingerprint	查看md5指纹

#判断证书与私钥是否匹配
openssl x509 -pubkey -noout -in www.zlwm115.com.crt
openssl rsa -pubout -in www.zlwm115.com.key
```

## 三、php进阶

##### php安装及模块拓展

```bash





```

##### php调试

```bash
-a               以交互式shell模式运行
-c <path>|<file> 指定php.ini文件所在的目录
-n               指定不使用php.ini文件
-d foo[=bar]     定义一个INI实体，key为foo，value为'bar'
-e               为调试和分析生成扩展信息
-f <file>        解释和执行文件<file>.
-h               打印帮助
-i               显示PHP的基本信息
-l               进行语法检查 (lint)
-m               显示编译到内核的模块
-r <code>        运行PHP代码<code>，不需要使用标签 <?..?>
-B <begin_code>  在处理输入之前先执行PHP代码<begin_code>
-R <code>        对输入的没一行作为PHP代码<code>运行
-F <file>        Parse and execute <file> for every input line
-E <end_code>    Run PHP <end_code> after processing all input lines
-H               Hide any passed arguments from external tools.
-S <addr>:<port> 运行内建的web服务器.
-t <docroot>     指定用于内建web服务器的文档根目录<docroot>
-s               输出HTML语法高亮的源码
-v               输出PHP的版本号
-w               输出去掉注释和空格的源码
-z <file>        载入Zend扩展文件 <file>.
args...          传递给要运行的脚本的参数. 当第一个参数以-开始或者是脚本是从标准输入读取的时候，使用--参数
--ini            显示PHP的配置文件名
--rf <name>      显示关于函数 <name> 的信息.
--rc <name>      显示关于类 <name> 的信息.
--re <name>      显示关于扩展 <name> 的信息.
--rz <name>      显示关于Zend扩展 <name> 的信息.
--ri <name>      显示扩展 <name> 的配置信息.


常用命令：
php -v								#查看版本
php -l index.php			#检查语法
php --ini							#查找php配置文件
php -i								#查看php信息，等同于用php --info来调用phpinfo()函数
php -m								#查看php已安装哪些扩展模块
php --ri mysqlnd			#查看redis模块配置信息,如果没有安装则提示该模块未安装
php -f index.php			#解释并执行php文件
php -S localhost:8000	#指定内建的web服务器
```

##### 查php配置

```bash
ps auxf |grep [p]hp							#查看该进程的父子进程
grep include /etc/opt/remi/php71/php-fpm.conf
ll /etc/opt/remi/php71/php-fpm.d/*.conf
grep log /etc/opt/remi/php71/php-fpm.d/longines_cn_www.conf
grep '^[a-Z 0-9]' /etc/opt/remi/php71/php-fpm.d/longines_cn_www.conf
/var/log/php-fpm/www-slow.log
```



















