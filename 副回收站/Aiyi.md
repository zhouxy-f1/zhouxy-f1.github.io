```bash
#环境准备
jdk1.8、mysql、httpd、redis等

1.准备jdk1.8
su - SSLC
cp  jdk-8u261-linux-x64.tar.gz /home/SSLC/Soft
tar -zxvf jdk-8u261-linux-x64.tar.gz
cat>>~/.bashrc<<EOF
export JAVA_HOME=/home//jdk1.8.0_261
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
EOF
source ~/.bashrc


2. 部署httpd
yum install -y httpd
vim /etc/httpd/conf/httpd.conf
Listen 8002
User SSLC

cat>/etc/httpd/conf.d/v_host.conf<<EOF
<VirtualHost *:8002>
ProxyPreserveHost On
ServerName 172.28.16.211
DocumentRoot /home/SSLC/dist
RewriteEngine On

<Directory /home/SSLC/dist>
        Options -Indexes +FollowSymlinks
        AllowOverride All
        Require all granted
</Directory>

<Directory /home/SSLC/dist/static/>
        Order deny,allow
        Deny from all
        Allow from all
</Directory>

ErrorLog logs/sslc-error_log
CustomLog logs/sslc-access_log common
</VirtualHost>
EOF

systemctl start httpd

3.部署mysql

dnf install -y mysql-server
systemctl start mysqld
mysql -p
ALTER USER 'root'@'localhost' IDENTIFIED BY '123RLoot@AzQaz';
flush privileges;
create database longchuang549 charset utf8mb4 collate utf8mb4_general_ci;
source /root/longchuang549.sql
\q

4.部署redis
yum install -y redis
vim /etc/redis.conf
auth test


#先起后端的服务
mkdir /home/SSLC/jar/logs  -p
rz    (longchuang549-admin.jar和longchuang549-api.jar)     权限SSLC:SSLC
cat>/home/SSLC/start.sh<<EOF
echo 'call batch start job'
ps -ef | grep [l]ongchuang549 | grep -v grep | awk '{print $2}' | xargs kill -9;
nohup java -jar /home/SSLC/jar/longchuang549-admin.jar --spring.profiles.active=prod >/home/SSLC/jar/logs/longchuang549-admin-log.txt 2>&1 &
nohup java -jar /home/SSLC/jar/longchuang549-api.jar --spring.profiles.active=prod >/home/SSLC/jar/logs/longchuang549-api-log.txt 2>&1 &
EOF

chmod +x /home/SSLC/start.sh
su - SSLC
sh /home/SSLC/start.sh


#准备前端
mkdir /home/SSLC/dist
chown -R SSLC.SSLC /home/SSLC/dist


#发版
1.上传新的jar包（longchuang549-admin.jar和longchuang549-api.jar）
2.替换原来的jar包，路径在：/home/SSLC/jar，替换完记得更改属主属组改为SSLC
3.以SSLC的身份，重启服务，命令如下
su - SSLC -c 'sh /home/SSLC/start.sh'


```

