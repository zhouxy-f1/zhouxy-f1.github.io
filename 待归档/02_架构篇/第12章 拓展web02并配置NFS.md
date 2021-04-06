## 扩展多台web

##### 1.统一环境

```shell
0）准备对应的www用户
[root@web02 ~]# groupadd -g666 www
[root@web02 ~]# useradd -u666 -g666 www

1）拷贝web01上面的yum仓库
[root@web02 ~]# scp root@172.16.1.7:/etc/yum.repos.d/*.repo /etc/yum.repos.d/
	
2）安装nginx和php
[root@web02 ~]# yum -y install nginx php71w php71w-cli php71w-common php71w-devel php71w-embedded php71w-gd php71w-mcrypt php71w-mbstring php71w-pdo php71w-xml php71w-fpm php71w-mysqlnd php71w-opcache php71w-pecl-memcached php71w-pecl-redis php71w-pecl-mongodb
```
##### 2.统一配置

```shell
1）同步nginx
[root@web02 ~]# rsync  -avz --delete root@172.16.1.7:/etc/nginx/ /etc/nginx/
[root@web02 ~]# nginx -t
[root@web02 ~]# systemctl enable nginx
[root@web02 ~]# systemctl start nginx

2）同步php（/etc/php-fpm.conf /etc/php-fpm.d  /etc/php.ini）
[root@web02 ~]# rsync  -avz --delete root@172.16.1.7:/etc/php* /etc/
[root@web02 ~]# systemctl enable php-fpm
[root@web02 ~]# systemctl start php-fpm
```

##### 3.统一代码

```shell
	[root@web01 ~]# tar czf code.tar.gz /code							#在web01上打包站点
	[root@web01 ~]# scp code.tar.gz root@172.16.1.8:/tmp	#在web01上将打包好的代码发送给web02
	[root@web02 ~]# tar xf /tmp/code.tar.gz -C /					#在web02上进行解压，并解压到/目录下

```

##### 4.配置解析地址再访问

此时在web02上传的图片是存放于web02服务器本地，web01上无法访问

![image-20190908163930287](https://tva1.sinaimg.cn/large/006y8mN6gy1g6s77b2vs9j313z0dzmza.jpg)



## 共享多台web的静态资源

##### 1.配置nfs服务端

```shell
#准备172.16.1.31共享存储服务器，规划目录，配置好权限
0）创建用户
[root@nfs ~]# groupadd -g666 www
[root@nfs ~]# useradd -u666 -g666 www	

1）安装
[root@nfs ~]# yum install nfs-utils -y
	
2）配置
[root@nfs ~]# cat /etc/exports
/data/wordpress 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)

3）根据配置，创建目录，准备用户，授权等等
[root@nfs ~]# mkdir /data/wordpress -p
[root@nfs ~]# chown -R www.www /data/

4）启动
[root@nfs ~]# systemctl enable nfs-utils 
[root@nfs ~]# systemctl restart nfs-utils
```

##### 2.同步web图片

```shell
#在挂载前要把web01和web02上已有的静态资源推到nfs对应目录
[root@web01 ~]#rsync -avz /code/wordpress/wp-content/uploads/ 172.16.1.31:/data/wordpress
[root@web02 ~]#rsync -avz /code/wordpress/wp-content/uploads/ 172.16.1.31:/data/wordpress

注意1：在web01上用rsync推，会用web01全覆盖nfs，在web01上拉的话是用nfs全覆盖web01
注意2：用scp推送的需要上nfs服务器上进行重新的递归授权，否则会出现无法上传文件的错误，用rsync的不用
[root@nfs ~]# chown -R www.www /data/
```

##### 3.挂载

```shell
[root@web01 ~]# mount.nfs /code/wordpress/wp-content/uploads/ root@172.16.1.31:/data/wordpress
[root@web02 ~]# mount.nfs /code/wordpress/wp-content/uploads/ root@172.16.1.31:/data/wordpress
```

4、效果展示

![image-20190910114543030](https://tva1.sinaimg.cn/large/006y8mN6gy1g6u9y5tavsj30qw0k3jvc.jpg)

## 弊端

```shell
此时通过域名访问网站，只能手动切换域名解析
```

