## 制作yum仓库

为规范管理软件的各文件目录，需要编译安装软件，再把rpm包连同依赖文件一起打包成一个rpm

##### 准备rpm包

```bash
#下载依赖包和源码包
sed -i 's#keepcache=0#keepcache=1#g' /etc/yum.conf							#打开yum缓存
wget http://nginx.org/download/nginx-1.16.0.tar.gz							#下载源码包
yum install -y gcc gcc-c++ pcre-devel zlib-devel openssl-devel	#下载依赖包
find /var/cache/yum -name "*.rpm"|xargs cp -t /usr/local/src		

#源码安装nginx
tar xf nginx-1.16.0.tar.gz
cd nginx-1.16.0
./configure --prefix=/app/nginx-1.16.0													#生成makefile文件
make																														#生成二进制命令
make install																										#把生成的命令拷贝/usr/bin
```

##### 用fpm打包rpm

```bash
#安装fpm工具
tar xf fpm-1.3.3.x86_64.tar.gz 
yum install -y ruby rubygems ruby-devel													#安装ruby才能用gem
gem sources --add https://mirrors.huaweicloud.com/repository/rubygems/ --remove https://rubygems.org/																						#更换gem命令的下载源为华为源
gem sources --list																							#查看源是否换成功
gem install /fpm/*.gem																					#用gem命令安装fpm工具
yum install -y rpm-build																				#打包前需要安装rpm-build包

#打包
fpm -s dir -t rpm -n nginx -v 1.16.0 -d 'pcre-devel,openssl-devel' -f /app/nginx-1.16.0/

fpm打包参数详解
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

##### 制作yum仓库

```bash
安装vsftpd服务，共享目录在/var/ftp/pub，在/opt目录下新建base仓库和其它仓库，再挂载进pub即可

yum install -y vsftpd																#安装vsftpd服务
systemctl start vsftpd															#启动服务
mkdir /var/ftp/{base,epel} -p												#装备yum仓库的目录
cp nginx-1.16.0-1.x86_64.rpm /var/ftp/epel					#把各rpm包归到对应的仓库
createrepo /var/ftp/base														#生成base仓库
createrepo /var/ftp/epel														#生成epel仓库
cat >/etc/yum.repos.d/base.repo<<EOF								#准备各软件repo文件，推到各主机即可用yum从本地仓库安装
[base]
name=This is local opt
baseurl=ftp://10.0.0.99/pub/base/										#在浏览器上复制过来
enabled=1
gpgcheck=0
EOF
yum install -y nginx																#yum安装会走本地源
```

