##  一、Docker安装

##### docker理论基础

```bash
#什么是docker
Docker是一种进程虚拟化技术，本质上是一个进程，进程停止则容器销毁。docker通过namespace实现资源隔离，通过cgroup技术来限制进程使用多少cpu、内存、硬盘IO、网络IO等资源，隔离的环境拥有自己的系统文件、IP地址、主机名等。实际生产中，开发者把jar包给运维来部署，可能会因环境不同导致部署不成功，那么开发把jar包连同环境一起打包成docker镜像，可实现一次封装，在任何软件任何环境都可运行

#docker相关概念
镜像：是一个定义容器模板的文件，用于创建容器，容器也是分层的，区别是容器的最上层可读可写，镜像只读
容器：是用镜像创建的运行实例，该实例是一个极简的linux环境+服务，一个容器运行一种服务，容器本质上是一个进程，可以启动、停止、删除等
仓库：是用于存放镜像文件的地方，可通过push上传镜像到仓库，也可通过pull把镜像从仓库拉取到本地
命名空间：用于做资源隔离（比如两个子脚本里有重名的函数，脚本运行时无法区别调用的是哪个脚本的函数，可在函数前加空间名以区分）
cgroups：用于做资源限制（只能限制一个进程使用的资源，比如cpu、内存、硬盘IO等）

#docker底层原理
docker是一种虚拟化技术，传统虚拟化如kvm、vmware等都是在宿主机的操作系统上通过Hypervisor来模拟硬件，在此基础上安装各种完整的操作系统，再来运行软件。而docker是C/S架构，共用宿主机内核，通过docker引擎直接调用本机的硬件资源。由此可见，kvm虚拟化需要硬件能支持虚拟化，而且完全模拟硬件（比如硬盘，kvm需要写入虚拟机硬盘再转换存入宿主机硬盘，docker写入150m/s，KVM写入30m/s），镜像庞大，内存io等消耗大，可移植性差

每次启动容器都会挂载3个文件 
/etc/hosts  /etc/hostname  /etc/resolv.conf
这样可以让容器有不同的主机名、DNS等，让容器能上网
```

![image-20201003152206963](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjc6990m14j30j30a5my2.jpg)

##### docker安装部署 

```bash
docker-ce是社区版（Community Edition），docker-ee是企业版（Enterprise Edition）

#配源
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#安装
yum remove -y docker docker-common docker-selinux docker-engine
yum list docker-ce --showduplicates |sort -hr
yum install -y docker-ce-[version]


#配置镜像加速
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'

cat>/etc/docker/daemon.json<<EOF
{
    "graph": "/home/lib/docker",
    "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"],
    "exec-opts":["native.cgroupdriver=systemd"]
}
EOF
systemctl restart docker && systemctl status docker



#启动docker
systemctl daemon-reload
systemctl start docker
systemctl enable docker
docker version
docker info
docker status
dockck run hello-world

注意：
1.docker只能运行在CentOS6.5及以上的系统，要求内核版本在2.6.32-431以上，查看系统版本命令：lsb_release -a
2.centos8系统需要安装containerd.io
yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/edge/Packages/containerd.io-1.2.13-3.1.el7.x86_64.rpm


#处理报错
1.Your kernel does not support cgroup blkio weight_device
2.
```

##### docker管理命令

```bash
#帮助命令
docker version													#查看版本信息
docker info															#查看详细docker服务端状态，如容器数量、当前镜像站、网络类型、驱动器、插件、内核版本等
docker [run] --help											#帮助命令

#容器命令
docker ps																#查看当前存活的容器
docker ps -a|-l|-n3											#不论存亡地查看容器，-a是所有，-l是最后一个 -n3是最后3个
docker ps -qa --no-trunc								#-q是静默输出只显示容器id，--no-trunc是不截取字段，显示完整的容器id和命令
docker create --name web1 nginx					#根据镜像名或镜像id来创建容器，但不启动容器，本地若没有nginx镜像则先自动pull到本地
docker start|restart web1								#启动或重启指定的容器，按容器名或容器id
docker stop|kill web1										#停掉容器，stop相当于按关机，kill相当于按电源
docker run -it centos  /bin/bash 				#-i是交互-t是返回终端，无-i则进入终端后无法输入命令，无-t则不返回终端，终端解释器是容器内定义的
docker run -d centos /bin/bash -c "while true;do echo good;sleep 3;done" #后台运行的命令，没有-c的话启动后立马自动退出
注：run是创建并启动容器，容器无论在前台或后台运行，都需要执行一个能一直挂起的命令，否则容器引擎会认为该容器没事做，会自动退出，退出不是消亡
ctrl+p+q																#不停止容器方式退出终端，exit或ctrl+d是停止并退出容器
docker attach -it nginx /bin/bash				#进入容器，所有用户使用同一个终端，同步操作退出同一终端
docker exec -it nginx /bin/bash					#进入容器，并返回交互终端
docker exec -t nginx ls -l /tmp					#不进入容器，没有-i是不交互，直接从宿主机上传送命令到容器内执行
docker logs -ft --tail 2 147ae4dbc6a0		#查容器日志 -t是加时间戳 -f是追加输出 --tail是只显示最后多少行
docker top 147ae4dbc6a0									#查容器内部运行的进程
docker inspect bcc8424c1d4d							#查看容器详情，包括id、state、image、config、network等
docker rm 9a1fdaf81fd1	21125d5149b0		#删除1个或多个容器,不可删除正在运行的容器
docker rm -f $(docker ps -qa)						#删除所有容器 -f是强制删除正在运行的容器

#容器启动示例
docker run -w /tmp centos pwd;ls -l `pwd`
docker run -d -p 80:80 --name nginx -v /opt:/usr/share/nginx -e PWD=test /bin/bash
docker run -it centos bash

-d				    			#放后台运行 --detach
-e PWD=pwd123				#指定环境变量
-it									#返回一个交互式终端 -interactive --tty 
-p									#映射端口 宿主机端口:容器端口
-v 									#目录或文件映射，宿主机目录:容器目录
-h jenkins					#给容器指定hostname为server1
-w /tmp 						#指定进入容器时的工作目录
--name 			  			#给容器指定名称，不指定则系统随时生成，格式xxx_xxx
--restart=always		#重启docker时拉起容器，默认是no
--network 					#指定容器使用的网络
--ip								#指定IP
/bin/bash						#启动容器后执行的第一个命令
docker run -d -p 10.0.0.13:8080:80								#多个容器想使用宿主机的同一个端口
```

##### docker容器编排

```bash
docker-compose可通过docker-compose.yml来批量管理容器，运行镜像也不用添加大量参数

#下载安装
wget https://github.com/docker/compose/releases/download/1.27.4/docker-compose-Linux-x86_64
chmod 777 docker-compose-Linux-x86_64
mv docker-compose-Linux-x86_64  /usr/local/bin/docker-compose
export PATH=/usr/local/bin:$PATH

#编写docker-compose.yml
version: '3.1'
services:
  mysql:																					#服务名
    restart: always																#启动docker时自动拉起本容器
    images: daocloud.io/library/mysql:5.7.4				#镜像路径
    container_name: mysql													#指定容器名
    ports:
      - 3306:3306
    environment:
      MYSQL_ROOT_PWD: root
      TZ: Asia/Shanghai
    volumes:
      - /opt/docker/mysql_data:/var/lib/mysql

vim docker-compose.yml
version: '3'
services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress
   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     volumes:
       - web_data:/var/www/html
     ports:
       - "80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
volumes:
    db_data:
    web_data:
#使用docker-compose,需要在yml文件所在目录执行如下命令
docker-compose ps										#查看docker-compose管理的容器
docker-compose up -d								#启动所有管理的容器
docker-compose down									#删除所有管理的容器
docker-compose start|stop|restart		#操纵所有管理的容器
docker-compose logs -f							#查看所有管理的容器日志，一般会在yml文件里定义各自的日志
```

## 二、Docker镜像

![image-20201003121838656](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjc0ydijuhj30pf0cz3z9.jpg)

##### docker镜像原理

```bash
#镜像定义
镜像是一种轻量级、可执行的独立软件包，包含运行环境和基于环境开发的软件，如代码、库、环境变量、配置文件等。镜像是分层结构的，基于分层结构可以实现资源共享，docker分层可复用，比如CentOS基础镜像只加载一份，所有软件共用，相同的内容也只用加载一份到内存

#镜像底层原理
镜像是由一层层的文件系统组成，这种层级的文件系统叫UnionFS，即联合文件系统。其支持对文件系统的修改作为一次提交来层层叠加
最底层是bootfs(boot file system)，包含内核和内核加载器(bootloader)
第二层是rootfs(root file system)，包含不同的操作系统，rootfs可以很小，只需要包括最基本的命令、工具和程序库就行了
第三层是软件，软件之上dockerfile的每个变化是一层，如FROM、RUN、CMD等
以上统称为镜像层，镜像层只能读，当容器启动时，会加载一个可写层到镜像层顶部，这个可写层就是容器层
```

##### docker镜像命令

```bash
#基础命令
docker search -s 100 --no-trunc nginx								#-s是过滤收藏数大于等于100的镜像，--no-trunc是显示完整描述信息
docker pull nginx																		#从配置的仓库（阿里）拉镜像，是指定版本则默认最新，即tomcat:latest
docker pull nginx:1.18.0														#拉取指定版本的镜像
docker images -qa																		#列出本地镜像，-a是所有镜像，包含中间镜像层，-q是只显示镜像id
docker images -a --digests --no-trunc 							#--digests是显示镜像摘要，即完整hash值，--no-trunc是显示完整的镜像id
docker history nginx:1.18.0													#查看镜像每层的构建命令，可查看首行判断从该镜像创建容器后是否能挂起
docker inspect fe139cca9e1b													#查看镜像详情
docker image prune																	#清除构建失败的镜像
docker save centos:7.8 -o /tmp/centos:7.8.tgz				#把本地镜像导出为压缩包,使用save +镜像id时，再导入就没有tag了
docker load -i /tmp/centos:7.8.tgz									#把压缩包导入到本地镜像
？？docker images|awk -F '[/ ]+' 'NR>1{print "docker save "$5" -o "$3".tgz"}'|bash
docker images|awk -F ''
for i in `ls -1`;do docker load -i $i;done
docker commit -m "blog" -a "zxy" nginx wp:v1				#把容器nginx提交成镜像
docker tag metro-sms 192.168.0.148/metro-sms:v1.0		#本地镜像打标签，复制一份并重命名，类似硬链接
docker push 192.168.0.148/metro-sms:v1.0						#把本地镜像上传到148仓库
docker rmi -f tomcat	nginx													#删除多个镜像，-f是强制，可删中间层，同样不指定版本默认删标签为latest的
docker rmi -f `docker images -qa`										#删除所有本地镜像
docker rmi `docker images|awk '\nginx\{print $3}'`	#批量删除nginx镜像

#补充
1.查看镜像有哪些可用版本
yum install -y jq
curl -s https://registry.hub.docker.com/v1/repositories/centos/tags|jq
docker pull centos:7.8.2003

2.批量导出镜像为压缩包
docker images|awk 'NR>1{print "docker save "$3" -o /tmp/images_"$1":"$2".tgz"}'|bash
```

##### dockerfile镜像

```bash
Dockerfile是用于构建docker镜像的脚本文件，由一系列的命令和参数构成，定义了进程需要的一切东西，包括执行代码或文件、环境变量、依赖包、运行时环境、动态链接库、操作系统的发行版、服务进程、内核进程、命令空间等。其构建过程是：依据FROM拉取基础镜像，再从上到下逐步执行指令，每条指令都会创建一个新的镜像层，并对镜像做一次提交给下一层用，直到完成所有指令。保留字可用小写，大写更规范，保留字后不可为空，用井号做注释

#优化Dockerfile
1.尽可能选择小体积的linux,如alpine
2.尽可能把RUN指令合并，减少分层
3.清除无用的文件，比如yum缓存、源码包等
4.使用ADD复制文件时，在项目目录中新建.dockerignore文件中添加名称，减少不必要的文件ADD
5.修改dockerfile时，把变化的内容尽可能放在结尾处，可走缓存，提高构建镜像速度

#保留字指令
FROM					#指定基础镜像
MAINTAINER		#标注作者及邮箱，例：zhouxy<zhou012@yeah.net>
RUN						#执行命令，每个RUN都会起一个临时容器，执行完命令后缓存给下一个RUN用，尽可能合并，减少分层，例：yum install -y vim && .. /
EXPOSE				#指定容器暴露的端口，此时启动容器的命令中-P参数才能起作用 例： EPXORT 80 22
WORKDIR				#从终端进入容器时的工作目录，默认是\
ENV						#指定环境变量,如果要定义多个变量，则多写几行ENV，此变量可被命令行覆盖 例：ENV ROOT_PWD 123
ARG						#编译镜像时可通过docker build --build-arg=PROFILE=develop -t reg.hailian.com/herelink-demo:v1 . 来传入参数
COPY					#拷贝Dockerfile所在目录中的文件或目录到容器的指定路径，不指定路径则默认拷贝到WORKDIR下，在指定WORKDIR之前都拷贝到根目录下
ADD						#比COPY多自动处理URL和解压tar包的功能，跟COPY一样，文件名支持正则，例：ADD maven /maven
VOLUME				#指定容器中的数据卷，宿主机的数据卷是随机的，--volume-from跟某个已经存在的容器挂载相同的卷。例: ["volume1","volume2"]
CMD						#指定容器启动时的初始命令，可有多个但只有最后一个生效，且生效的CMD整个命令还会被docker run 后的命令覆盖掉 
ENTRYPOINT		#功能跟CMD一样，区别是docker run后的命令作为entrypoint命令的参数，追加到后面，例：CMD ["程序" "参数1" "参数2"]
ONBUILD				#触发器，当构建一个被继承的dockerfile时，就会触发ONBUILD的指令

#生成镜像
docker build -t nginx:v1.0 .	#.是指把哪个目录打包成镜像,Dockerfile目录下执行build命令且文件名为Dockerfile时，可省略-f,-t指定tag
docker build -f /data/Dockerfile_foo  -t nginx:v1.0 /data
```

##### harbor仓库

```bash
当需要大规模使用docker hub时，大家都从官方仓库拉取镜像，非常占带宽，此时就需要私有仓库（registry），registry本就是一个服务，直接起用registry容器即可。但是通过registry配置私有仓库，功能弱，权限控制简单，删除镜像麻烦等问题，临时应急才用，一般选用harbor，harbor是把官方仓库加以拓展，功能性更强。harbor仓库只需要在服务器6上安装即可


#自签证书
1.生成RSA私钥，需要创建访问私钥的密码
2.通过RSA私钥生成CSR证书签名请求，需要依次输入私钥的密码、国家代码、省会、城市、组织、公司名、域名等信息
3.删除私钥中的密码
4.通过RSR生成crt证书
5.把证书拷贝到指定目录
openssl genrsa -des3 -out ca.key 2048
openssl req -new -key ca.key -out ca.csr
openssl rsa -in ca.key -out ca.key
openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
openssl x509 -in ca.crt -out ca.pem -outform PEM
mkdir /etc/docker/certs/ -p
mv ~/ca.key /etc/docker/certs/reg.longchuang.com.key
mv ~/ca.crt /etc/docker/certs/reg.longchuang.com.crt
mv ~/ca.csr /etc/docker/certs/

#下载harbor安装包并修改配置文件
wget https://github.com/goharbor/harbor/releases/download/v2.0.1/harbor-offline-installer-v2.0.1.tgz
tar xf harbor-offline-installer-v2.0.1.tgz  -C /opt
cp /opt/harbor/harbor.yml.tmpl /opt/harbor/harbor.yml
sed -i '/^hostname/s#reg.mydomain.com#reg.hailian.com#g' /opt/harbor/harbor.yml
sed -i 's#/your/certificate/path#/etc/docker/certs/ca.crt#g' /opt/harbor/harbor.yml
sed -i 's#/your/private/key/path#/etc/docker/certs/ca.key#g' /opt/harbor/harbor.yml
sed -i 's#Harbor12345#hailian@2021#g' /opt/harbor/harbor.yml
cd /opt/harbor
./prepare
./install.sh

#下载docker-compose并启动harbor
yum install -y docker-compose 
chmod +x /usr/local/bin/docker-compose
docker-compose version
docker-compose -f /opt/harbor/docker-compose.yml up -d

注：若没有找到源可用如下命令
curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

#docker和k8s连接harbor
1. k8s集群连接harbor
需要通过命令或yaml文件来创建registry-harbor的Secret对象，存入harbor的账号和密码，然后创启微服务时在deploy.yml中引用这个secret对象
kubectl create secret docker-registry registry-harbor  --docker-server=reg.longchuang.com --docker-username=admin --docker-password=test --docker-email=abc@qq.com

2.docker连接harbor，由于是自签证书不被docker信任，只需要先配置hosts解析再把harbor服务器上的证书拷贝到本地指定目录即可
方法1：直接生成证书
mkdir -p /etc/docker/certs.d/reg.hailian.com
openssl s_client -showcerts -connect reg.hailian.com:443 < /dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /etc/docker/certs.d/reg.hailian.com/ca.crt

方法2：拷贝证书
mkdir -p /etc/docker/certs.d/reg.hailian.com
scp harbor:/etc/docker/certs/ca.crt /etc/docker/certs.d/reg.hailian.com/ca.crt

#在命令行或网页上使用harbor
docker login -uadmin reg.hailian.com
http://reg.longchuang.com			用户名：admin				密码：在harbor.yml中定义 (默认是Harbor12345)
```

##### registry仓库

```bash
如果只想临时使用仓库，可在本机用docker起一个仓库服务

1.处理base认证再起容器
yum install httpd-tools -y
mkdir /opt/registry-var/auth/ -p
htpasswd  -Bbn oldboy 123456  >> /opt/registry-var/auth/htpasswd
docker run -d -p 5000:5000 --restart=always -v /opt/registry-var/auth/:/auth/   -v /opt/myregistry:/var/lib/registry -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e  "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" registry 

2.处理https问题
vim /etc/docker/daemon.json
 	{
 	"registry-mirrors": ["https://registry.docker-cn.com"],
 	"insecure-registries":["10.0.0.11:5000"]
	}
systemctl restart docker

3.上传镜像
docker tag nginx:latest 10.0.0.11:5000/nginx:latest
docker push 10.0.0.11:5000/nginx:latest
```

## 三、Docker数据卷

##### 数据持久化

```bash
卷是目录或文件，存在于一个或多个容器中，由docker挂载到容器，但不属于联合文件系统，绕过UnionFS实现数据持久化或容器间共享数据，卷中数据的更改会直接生效，其更改不包含在镜像的更新中

#docker数据卷命令
docker volume create wp				#创建数据卷，默认路径是 /var/lib/docker/volumes/wp/_data
docker volume inspect wp			#查看数据卷详情，如仓库时间、名称、路径等
docker volume ls							#查看所有数据卷
docker volume rm wp						#删除数据卷
docker volume prune						#只清理当前未被容器挂载使用的数据卷。当删除容器，挂载卷默认不会自动删除，可手动清理

#挂载数据卷
1.匿名挂载
docker run -d -P -v /usr/share/nginx/html nginx

挂载时仅需指明容器内的路径，如果该路径不存在则自动创建，宿主机上对应的目录也会自动创建，默认路径是/var/lib/docker/volumes/*/_data
如果容器内的路径存在，则在宿主机上自动创建映射目录，且会把容器内目录的文件复制到宿主机，容器内目录权限默认为rw，不可改
查看路径：docker inspect xxx|grep Source
查看挂载卷：docker volume ls

2.具名挂载
docker run -d -v dirname:/usr/share/nginx/html:ro  nginx

挂载时仅指定宿主机内的目录名，跟匿名挂载一样会自动创建目录也会拷贝内容到宿主机，区别是路径为/var/lib/docker/volumes/dirname/_data，
另外，此时还可以给容器内目录设置读写权限为ro，默认rw

3.指定路径挂载
docker run nginx -d -v /data:/usr/share/nginx/html/ 

如果停掉容器时在宿主机目录写入新内容，那么容器启动后自动加载更新的内容，此外还可以给容器的目录设置权限如ro，设置ro后容器内的映射目录只能读不能写，可实现宿主机到容器的单向写入操作。
宿主机目录不存在 容器目录不存在   				自动创建宿主机目录和容器目录
宿主机目录不存在 而容器目录存在					自动创建宿主机目录 清空容器目录
宿主机目录不存在 容器目录存在且为只读			自动创建宿主机目录 清空容器目录，容器目录只读
宿主机目录存在	  且容器目录存在					 宿主机目录内容完整覆盖容器目录
宿主机目录存在   容器目录不存在  				 自动创建容器目录，复制宿主机目录的文件到容器目录

4.容器间数据共享
docker run -d --name docker1 -v /volume1:/mnt/docker nginx
docker run -d --name docker2 --volumes-from docker1 nginx
docker run -d --name docker3 --volumes-from docker2 nginx
VOLUME ["volume1","volume2"]

宿主机目录与容器间只需要做一次挂载即可实现容器间数据共享，如上示例 docker1为数据卷容器，docker2通过--volumes-from docker1来挂载与docker1相同的目录，此时docker1与docker2之间数据是共享的，docker3挂载到docker1或docker2上都能实现3个docker之间的数据共享，哪怕docker1容器被删掉了，数据卷容器的生命周期一直持续到没有容器使用为止
```

## 四、Docker网络

##### 网络原理

```bash
docker0连接eth0：docker服务启动时，生成网卡docker0，与eth0通过NAT连接
docker0连接各容器：通过evth-pair连接，evth-apir是一对虚拟的设备接口
容器连接容器：容器1 -> 通过evth连接docker0 -> docker0路由转发到容器2
容器连接外网：容器  -> docker0 -> eth0 ->外网

#NAT需要开启内核转发并依赖iptables规则
sysctl net.ipv4.ip_forward=1
iptables -t nat -nL

#docker的网络类型
1.Bridge：docker默认类型
2.Container：与另一个运行中的容器共享Network Namespace，--net=container:containerID（K8S
3.Host：与宿主机共享Network Namespace，--network=host  性能最高 
4.None：不为容器配置任何网络功能，--net=none
```

##### 容器互连

```sh
#docker容器互连
采用bridge模式，容器间可以通过ip连接，但无法通过主机名连接，由于docker容器重启后ip会变，不利于应用间的连接，解决办法如下：

方法1：通过--link连接
原理：在启动web3容器时，用--link指定连接到web1，创建web3容器时会自动在其hosts文件里加入web2的解析记录，这种是单向的
docker run -d --name web3 --link web1 tomcat
docker exec web3 ping web1	=> 能通
docker exec web1 ping web3	=> 不通

方法2：通过自定义网络
docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 net_web
docker run -d --name web1 --net net_web tomcat
docker run -d --name web2 --net net_web tomcat
docker exec web1 ping web2 => 能通
docker exec web2 ping web1 => 能通

#不同网段间连接
当做集群连接时，web集群需要连接mysql集群，由于是通过自定义网络，两个集群不在同一网段，无法通讯，那么可以把指定的容器加入另一个集群
原理：通过执行connect命令，把web1加入net_mysql，实际是在web1再虚拟了一块网卡，其ip跟net_mysql在同一网段，再把这个ip加入到mysql集群。
这样web1就可以跟任何一台mysql连接，包括后来才加入集群的mysql
docker network connect net_mysql web1
```

## 五、Docker监控

##### cadvisor

```bash
监控架构：cadvisor(获取数据)+influxdb(存入数据库)+grafna(图形展示和警报)
cadvison由谷歌公司开发，解决了docker stats取值动态变化的问题

docker load -i docker_monitor.tar.gz 
#先启动并配置influxdb
docker run -itd -p 8083:8083 -p 8086:8086 --name influxdb tutum/influxdb
访问：10.0.0.12:8083
建库：Query Tempates-Create Database-cadvisor
用户：Query Tempates-Create Admin User-root root
#再启cadvisor
docker run -itd --name cadvisor -p 8080:8080 --link influxdb:influxdb --mount type=bind,src=/,dst=/rootfs,ro --mount type=bind,src=/var/run,dst=/var/run --mount type=bind,src=/sys,dst=/sys,ro --mount type=bind,src=/var/lib/docker/,dst=/var/lib/docker,ro google/cadvisor -storage_driver=influxdb -storage_driver_db=cadvisor -storage_driver_user=root -storage_driver_password=root -storage_driver_host=influxdb:8083
访问 http://10.0.0.12:8080

#influxdb拿到数据就启grafna
docker run -itd --name grafana  -p 3000:3000 grafana/grafana
访问：10.0.0.12:3000
账户：admin 密码：admin
```

##### cgroup

```bash
cgroup 是 Control Groups 的缩写，是 Linux 内核提供的一种可以限制、记录、隔离进程组所使用的物理资源(如 cpu、memory、磁盘IO等等) 的机制，被 LXC、docker 等很多项目用于实现进程资源控制。cgroup 将任意进程进行分组化管理的 Linux 内核功能。cgroup 本身是提供将进程进行分组化管理的功能和接口的基础结构，I/O 或内存的分配控制等具体的资源管理功能是通过这个功能来实现的。这些具体的资源管理功能称为 cgroup 子系统，有以下几大子系统

blkio：设置限制每个块设备的输入输出控制。例如:磁盘，光盘以及 usb 等等。
cpu：使用调度程序为 cgroup 任务提供 cpu 的访问。
cpuacct：产生 cgroup 任务的 cpu 资源报告。
cpuset：如果是多核心的 cpu，这个子系统会为 cgroup 任务分配单独的 cpu 和内存。
devices：允许或拒绝 cgroup 任务对设备的访问。
freezer：暂停和恢复 cgroup 任务。
memory：设置每个 cgroup 的内存限制以及产生内存资源报告。
net_cls：标记每个网络包以供 cgroup 方便使用。
ns：命名空间子系统。
perf_event：增加了对每 group 的监测跟踪的能力，可以监测属于某个特定的 group 的所有线程以及运行在特定CPU上的线程。
```

