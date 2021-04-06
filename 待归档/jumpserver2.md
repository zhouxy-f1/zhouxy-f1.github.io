## Jumpserver简介

##### 架构图及组件说明

```bash
Jumpserver: 管理后台,其核心组件是core，用Django Class Based View风格开发，支持Restful API
koko: 用来实现ssh和web terminal server，提供ssh和websocket接口，使用paramiko和flask开发
Luna: 是web terminal的前端，提供计划前端页面，负责后台渲染html
Lina: 
Guacamole：apache跳板机项目，该组件实现了jumpserver的RDP功能
Jumpserver-Python-SDK: 主要用于跟jumpserver API交互
```

## 极速安装

```bash
cd /opt
yum -y install wget git
git clone --depth=1 https://github.com/jumpserver/setuptools.git
cd setuptools
cp config_example.conf config.conf
vi config.conf
./jmsctl.sh -h
./jmsctl.sh install
./jmsctl.sh uninstall
./jmsctl.sh -h
```

## 正常安装部署

```bash
#准备环境
gzip /etc/yum.repos.d/CentOS-Base.repo
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache fast
yum update -y
cat /etc/redhat-release 									#CentOS Linux release 7.8.2003 (Core)
#提前下载或上传软件包到jumpserver服务器
wget https://github.com/jumpserver/jumpserver/releases/download/v2.0.2/jumpserver-v2.0.2.tar.gz
wget https://github.com/jumpserver/koko/releases/download/v2.0.2/koko-v2.0.2-linux-amd64.tar.gz
wget https://github.com/jumpserver/lina/releases/download/v2.0.2/lina-v2.0.2.tar.gz
wget https://github.com/jumpserver/luna/releases/download/v2.0.2/luna-v2.0.2.tar.gz
wget https://github.com/jumpserver/docker-guacamole
git clone --depth=1 https://github.com/jumpserver/docker-guacamole.git

1.安装并配置python3环境
yum install -y python36 python36-devel
python3.6 -m venv /opt/py3
source /opt/py3/bin/activate

2.安装配置mysql5.6
wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
rpm -Uvh mysql-community-release-el7-5.noarch.rpm
yum install -y mysql-community-server
/etc/init.d/mysqld start
mysql
> create database jumpserver default charset 'utf8' collate 'utf8_bin';
> grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by 'jumpserver';
>\q

3.安装配置redis
tar xf redis-5.0.8.tar.gz -C /usr/local/
cd /usr/local/redis-5.0.8/
make && make install
redis-server &

4.编译安装jumpserver
tar xf jumpserver-v2.0.2.tar.gz  -C /opt
mv /opt/jumpserver-v2.0.2/ /opt/jumpserver
yum install -y `cat /opt/jumpserver/requirements/rpm_requirements.txt`
yum install -y `cat /opt/jumpserver/requirements/requirements.txt`
pip install --upgrade pip
pip install wheel -i https://mirrors.aliyun.com/pypi/simple/
pip install -U pip setuptools==33.1.1 -i https://mirrors.aliyun.com/pypi/simple/
pip install -r requirements/requirements.txt -i https://mirrors.aliyun.com/pypi/simple/

注：安装依赖的时候如果有报错，就先把requirements.txt里的那个包先注释掉，把剩下的装完，一般会有 huaweicloud-sdk-python==1.0.21和 python-gssapi==0.6.4安装不了，解决办法：注释已安装行，本次仅安装这两项，命令如下
pip install setuptools==33.1.1
pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/

配置并启动jumpserver
cd /opt/jumpserver
cp config_example.yml  config.yml
vim config.yml
第4行： SECRET_KEY: <nLQLH]6!(_nUy~Ew4k2Z9s{!U$N>gWeMKs>M=zu`6A7Qj4LEq
第8行： BOOTSTRAP_TOKEN: q3oX_@*dpVE723Lg6'zq?heq
第12行：DEBUG: false
第16行：LOG_LEVEL: ERROR
第22行：SESSION_EXPIRE_AT_BROWSER_CLOSE: true
第38行：DB_PASSWORD: jumpserver
第128行：WINDOWS_SKIP_ALL_MANUAL_PASSWORD: True
./jms start -d

5.安装配置KoKo组件并启动
tar -xf koko-v2.0.2-linux-amd64.tar.gz  -C /opt
mv /opt/koko-v2.0.2-linux-amd64/ /opt/koko
cp /opt/koko/config_example.yml /opt/koko/config.yml
vim config.yml
第10行： BOOTSTRAP_TOKEN: q3oX_@*dpVE723Lg6zq?heqa
第29行： LOG_LEVEL: ERROR
./koko -d

6.安装配置Lina和Luna组件
tar -xf lina-v2.0.2.tar.gz -C /opt
tar xf luna-v2.0.2.tar.gz -C /opt
mv /opt/lina-v2.0.2 /opt/lina
mv /opt/luna-v2.0.2 /opt/luna
chown -R nginx:nginx lina													#等安装完nginx再执行

7.安装配置guacamole组件
unzip docker-guacamole-master.zip  -d /opt
cd /opt/docker-guacamole-master
tar xf guacamole-server-1.2.0.tar.gz
mv guacamole-server-1.2.0 guacamole-server
tar xf ssh-forward.tar.gz  -C guacamole-server/bin/
cd guacamole-server
yum install -y cairo-devel libjpeg-turbo-devel libjpeg-devel libpng-devel libtool uuid-devel ffmpeg-devel freerdp-devel pango-devel libssh2-devel libtelnet-devel libvncserver-devel libwebsockets-devel pulseaudio-libs-devel openssl-devel libvorbis-devel libwebp-devel
./configure --with-init-dir=/etc/init.d && make && make install
/etc/init.d/guacd start

8.编译安装tomcat9
yum install -y java-1.8.0-openjdk
mkdir -p /config/guacamole/{extensions,record,drive}
chown -R daemon.daemon /config/guacamole/{record,drive}

tar xf apache-tomcat-9.0.37-src.tar.gz  -C /config
mv apache-tomcat-9.0.37-src/ tomcat
sed -i 's/Connector port="8080"/Connector port="8081"/g' /config/tomcat/conf/server.xml
echo "java.util.logging.ConsoleHandler.encoding = UTF-8" >> /config/tomcat/conf/logging.properties

rm -fr /config/tomcat/webapps/*
ln -s /opt/guacamole/root/app/guacamole/guacamole.properties  /config/guacamole/guacamole.properties
ln -s /opt/guacamole/guacamole-auth-jumpserver-1.0.0.jar /config/guacamole/extensions/guacamole-auth-jumpserver-1.0.0.jar
ln -s /opt/guacamole/guacamole-1.0.0.war  /config/tomcat/webapps/ROOT.war


export JUMPSERVER_SERVER=http://127.0.0.1:8080
echo "export JUMPSERVER_SERVER=http://127.0.0.1:8080" >> ~/.bashrc
export BOOTSTRAP_TOKEN=q3oX_@*dpVE723Lg6zq?heqa
echo export BOOTSTRAP_TOKEN=q3oX_@*dpVE723Lg6zq?heqa >> ~/.bashrc
export JUMPSERVER_KEY_DIR=/config/guacamole/keys
echo "export JUMPSERVER_KEY_DIR=/config/guacamole/keys" >> ~/.bashrc
export GUACAMOLE_HOME=/config/guacamole
echo "export GUACAMOLE_HOME=/config/guacamole" >> ~/.bashrc
export GUACAMOLE_LOG_LEVEL=ERROR
echo "export GUACAMOLE_LOG_LEVEL=ERROR" >> ~/.bashrc
export JUMPSERVER_ENABLE_DRIVE=true
echo "export JUMPSERVER_ENABLE_DRIVE=true" >> ~/.bashrc
chmod +x /config/tomcat/bin/*
sh /config/tomcat/bin/startup.sh

#安装配置nginx1.18.0
yum install -y nginx
gzip /etc/nginx/conf.d/default.conf
vim /etc/nginx/conf.d/jumpserver.conf
server {
    listen 80;
    client_max_body_size 100m;  # 录像及文件上传大小限制
    
    location /ui/ {
        try_files $uri / /index.html;
        alias /opt/lina/;
    }
    location /luna/ {
        try_files $uri / /index.html;
        alias /opt/luna/;  # luna 路径, 如果修改安装目录, 此处需要修改
    }
    location /media/ {
        add_header Content-Encoding gzip;
        root /opt/jumpserver/data/;  # 录像位置, 如果修改安装目录, 此处需要修改
    }
    location /static/ {
        root /opt/jumpserver/data/;  # 静态资源, 如果修改安装目录, 此处需要修改
    }
    location /koko/ {
        proxy_pass       		http://localhost:5000;
        proxy_buffering 		off;
        proxy_http_version	1.1;
        proxy_set_header 		Upgrade $http_upgrade;
        proxy_set_header 		Connection "upgrade";
        proxy_set_header 		X-Real-IP $remote_addr;
        proxy_set_header 		Host $host;
        proxy_set_header 		X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }
    location /guacamole/ {
        proxy_pass       http://localhost:8081/;
        proxy_buffering off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        access_log off;
    }
    location /ws/ {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8070;
        proxy_http_version 1.1;
        proxy_buffering off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    location /core/ {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    location / {
        rewrite ^/(.*)$ /ui/$1 last;
    }
}
systemclt start nginx
systemctl enable nginx




5000
8081
3306

```

















