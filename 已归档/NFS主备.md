## NFS共享存储

##### 安装配置nfs服务

```bash
以nfs1(10.0.20.51)为主，nfs2(10.0.20.52)为备，备通过定时任务拉取nfs1增量文件。若nfs1宕机，切换nfs2为主
备注：nfs1上共享目录为/opt/nfs/web,客户端静态文件目录为/var/www/share

#先在nfs1和nfs2上安装nfs服务,
yum install -y nfs-utils rpcbind									#rpcbind用于把nfs服务器的ip和端口发给客户端
vim /etc/exports																	#配置nfs服务端
/opt/nfs/web 10.0.20.11/32(rw,root_squash,anonuid=496,anongid=496,secure) 10.0.20.12/32(rw,root_squash,anonuid=496,anongid=496,secure) 
exportfs -a																				#无需重启让exports里配置生效
exportfs -u
cat /var/lib/nfs/etab															#检查配置文件是否生效
mkdir /opt/nfs/web -p															#创建共享目录
chown -R tomcat.tomcat /opt/nfs/web								#给共享目录授权		
systemctl start rpcbind nfs-server								#启动服务，先起底层的rpcbind服务
systemctl enable rpcbind nfs-server

#再把所有web节点的静态文件目录挂载到nfs1:/opt/nfs/web	，挂载前要保障数据一致
yum install -y nfs-utils rpcbind									#装nfs为使用showmount,rpcbind用于获取服务端可挂载目录
showmount -e 10.0.20.51														#查看服务端可挂载点
mount.nfs 10.0.20.51:/opt/nfs/web	/var/www/share	#把本机的share目录挂载到nfs1服务端的web目录
showmount -a																			#查看服务端已挂载了哪些远程主机
vim /etc/fastab																		#永久挂载
10.0.20.51:/opt/nfs/web /var/www/share defaults 0 0 

完成后：所有web节点上的/var/www/share目录内容与nfs1的/opt/nfs/web完全同步，在任一节点删文件，其它节点都会删
```

##### sersync

```bash
nfs1目前是单点，一旦宕机NFS服务就瘫痪了，需要实时把nfs1同步到nfs2，nfs1宕机就用nfs2顶替，实现方法有两种:
1.rsync+inotify:inotify只能记录有变化，不记录具体哪个目录和文件，rsync同步时要遍历扫描比对文件，效率太低
2.rsync+sersync:sersync基于inotify开发，可记录变化的目录或文件的名字，rsync同步时很快；多线程，适合大量小文件

sersync的原理是：实时监控/opt/nfs/web目录，并记录具体哪些文件或目录发生变化，一旦有变化，自动触发rsync，通过虚拟用户推送到nfs2,sersync推送时默认加了--delete参数，nfs1上删文件，nfs2上也会删


第1步：先在nfs2上配置好rsync服务(用root起rsync服务，rsync为虚拟用户，不用创建)
yum install -y rsync
cat >/etc/rsyncd.conf<<EOF				#配置nfs2上rsync服务，因为要在nfs2上拉nfs1的文件过来
uid=root													#用root用户起rsync服务
gid=root													#指定用户组
max connections=36000							#最大连接数
use chroot=no											#默认为true，修改为no，增加对目录文件软连接的备份 
log file=/var/log/rsyncd.log			#定义日志存放位置
ignore errors=yes									#忽略无关错误
read only=no	 										#设置rsync服务端文件为读写权限
auth users=rsync									#认证的用户名与系统帐户无关在认证文件做配置，如果没有这行则表明是匿名
secrets file =/etc/rsync.pass			#密码认证文件，格式(虚拟用户名:密码）
[nfs]															#这里是认证的模块名，在client端需要指定，可以设置多个模块和路径
comment=rsync nfs1 to nfs2				#自定义注释
path=/opt/nfs/web/								#同步到nfs2的文件存放的路径

echo "rsync:123456" >/etc/rsync.pass						#创建认证文件，可设多个认证用户，格式(用户名:密码)
chmod  600 /etc/rsync.pass /etc/rsyncd.conf			#设置所有者的读写权限
systemctl start rsyncd													#以守护进程方式启动rsync服务,也可用rsync --daemon	
																

第2步：在nfs1上安装inotify和sersync来监控/opt/nfs/web目录
yum install -y rsync inotify-tools														#安装基础文件
git clone https://github.com/wsgzao/sersync.git								#下载sersync
tar xf sersync/sersync2.5.4_64bit_binary_stable_final.tar.gz	#二进制包，解压即可用
mv GNU-Linux-x86/ /usr/local/sersync													#把sersync放到指定目录

vim /app/sersync/confxml.xml																	#配置sersync
第24行：<localpath watch="/opt/nfs/web">												#sersync监控的目录
第25行：<remote ip="172.16.1.52" name="nfs"/>									#备用nfs2地址和模块名
第31行：<auth start="true" users="rsync" passwordfile="/etc/rsync.pas"/> #nfs2认证信息
echo "123456">/etc/rsync.pas																	#将密码写入文件
chmod 600 /etc/rsync.pas																			#给读写权限

/app/sersync/sersync2 -rdo /app/sersync/confxml.xml	  				#启动sersync
-d 守护进程 -r 监控前先rsync推送一遍确保数据一致 -n 开启守护进程数量 默认10  -o 指定配置文件

启动后读配置文件，实际执行的命令如下
execute command: cd /opt/nfs/web && rsync -artuz -R --delete ./  --port=874 rsync@172.16.1.52::nfs --password-file=/etc/rsync.pas >/dev/null 2>&1
```

##### md5校验

```bash
[root@jonny-training ~]# md5sum 123.txt  >/opt/123.md5
[root@jonny-training ~]# cat /opt/123.md5 
86a20c1b92a2d831b50ba9d62e18ed86  123.txt
[root@jonny-training ~]# md5sum  -c /opt/123.md5 
123.txt: OK
[root@jonny-training ~]# echo "456456">>123.txt 
[root@jonny-training ~]# md5sum -c /opt/123.md5 
123.txt: FAILED
md5sum: WARNING: 1 computed checksum did NOT match
```



