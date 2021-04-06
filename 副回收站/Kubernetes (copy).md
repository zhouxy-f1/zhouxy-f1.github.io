## 一、K8S基础

##### 基础常识

```bash
#云平台
IaaS(Infrastructure as a Service,基础设施即服务): 厂商只提供硬件设备，IaaS分为公有云、私有云、混合云。如阿里云、腾讯云、华为云等
PaaS(Platform-as-a-Service,平台即服务): 厂商提供软件研发的平台，供企业部署各类管理软件，如CRM、OA等，代表厂商新浪云、docker
SaaS(Software-as-a-Service,软件即服务): 厂商提供可定制的软件，按功能和时长收费，企业用户通过浏览器完成软件的创建修改等

#k8s的诞生
一旦使用docker部署集群时，解决了环境与配置的问题，但集群之间的连通和资源管理问题无法解决
Apache MESOS: 开源的分布式资源管理框架，只是资源管理框架，2019年5月推特公司弃用mesos，转向k8s
docker swarm: 由docker公司开发，功能不够丰富，如滚动更新等实现起来非常复杂,2019年7月阿里弃用docker swarm转向k8s
kubernetes: 是由谷歌公司开发，使用go语言重构了自家的资源管理工具borg，并更名为kuberneres。该项目于2014年立项，2015年发布1.0版本并加入NCNF基金会，2016年打败Mesos和docker swarm一统江湖，2018年从cncf基金会毕业，k8s正式上大规模落地生产

#k8s特性
1.轻量：go语言开发，执行效率跟C语言差不多，而且支持进程管理，消耗资源少
2.弹性伸缩：根据访问量自动扩容缩容
3.负载均衡：
```

##### k8s架构

![k8s架构图](https://tva1.sinaimg.cn/large/0081Kckwgy1gkwyvyu6idj316a0mu40a.jpg)

```bash
#k8s组件说明
高可用集群副本数最好大于等于3且为奇数个

#master节点上
apiserver:所有服务统一入口
controllerManager:维持副本期望数目
etcd：数据库，存储k8s集群状态。天然支持分布式集群，建议使用Etcd3.0，因为2.0是纯内存的，已经在k8sv1.11中弃用

#node节点上
kubelet: 跟docker引擎交互，创建删除容器
kube-proxy: 写入规则到iptalbes和ipvs

#其它插件
coredns:core公司的dns服务器，为svc创建hosts解析
dashboard:B/S架构，提供界面管理工具
ingress controller: 官方k8s使用4层负载，而ingress可实现七层代理（根据主机名或域名实现负载）
federation: 可以实现跨集群统一管理多套k8s集群
prometheus: 监控k8s集群
efk: 日志收集分析
```

##### 相关概念

```bash
#副本控制器
pod可分为自主式pod和由控制器管理的pod，副本控制器有：
1.ReplicationController：用来确保容器应用的副本数始终保持在用户定义的副本数，受系统资源限制
2.ReplicaSet:支持集合式selector，即通过标签管理pod。但RS不支持rolling-update，Deploy支持
3.Deployment:deploy不直接创建容器,通过创建RS由RS来创建,滚动更新时新建一个RS,每创建一个新pod就从旧RS中删一个旧pod
4.Horizontal pod autoscaling: 基于RS资源定义HPA对象，监控RS管理的pod的cpu使用率等来伸缩pod数量，hpa定义数量，v1alpha版本中还可根据内存和用户自定义的metric做扩缩容

#StatefullSet
deployments和replicaSets是为无状态服务设计的，而有状态服务如数据库等需要用statefullSet，功能有
1.稳定的持久化存储：pod重新调度后还能访问到原来的持久化数据，基于pvc来实现
2.稳定的网络标志：pod重新调度后，PodName和HostName不变，基于Headless Service（一种没有ClusterIP的服务）实现
3.有序部署：当新建pod需要顺序要求时，如先mysql再nginx再php的，可实现只有当mysql完成再启动别的，基于initC来实现
4.有序收缩：收缩时先删php再nginx再mysql

#DaemonSet
在node上打污点，打了污点的node上不运行指定的pod。当新建node时，给node节点新增一些通用的pod，如fluentd、logstash、glusterd、Prometheus Node Exporter等

#Job和cron job
job负责批处理仅执行一次的任务，保证批处理任务的一个或多个pod成功结束。cronjob管理基于时间的job在指定时间周期性执行
类似定时任务，运行在pod里，运行失败时重新运行

#服务发现
svc通过标签选择一组pod，用户通过访问svc的ip和端口来访问pod，pod间轮询处理请求

#网络通讯
k8s的网络模型假定了所有pod都在一个可以直接连接的扁平化网络空间，pod与pod间可直接通过ip访问
1.同一pod内部：ip是共用pause容器的，通过localhost(回环口)+端口访问不同容器
2.pod1与pod2间：如果在同一台主机，直接通过docker0转发；如果不在就要通过overlay network
3.pod与service之间：通过各节点的iptalbes规则
4.pod访问外网：查找路由表转发数据包到宿主机的网卡，宿主机上路由选择后由iptalbes执行masquerade把源ip改为宿主机ip

#overlay network
为了让集群中不同节点的主机创建的容器都具有全局唯一的虚拟ip，而且还能在这些ip间建立一个覆盖网络，这个覆盖网络就是overlay network，通过overlay network把数据包原封不同传递到目标容器内。覆盖网络有很多，如
1.Flannel: CoreOS公司为k8s设计的一个网络规划服务,flannel与etcd间交互包括a、存储管理flannel可分配的ip段资源，b、监控etcd中每个pod的实际ip，并在内存中建立维护pod节点的路由表

2.weave cni:
```

## 二、K8S集群部署

##### 系统初始化

```bash
#换阿里源并更新软件
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all && yum makecache fast
yum install -y conntrack ntpdate ntp ipvsadm ipset jq wget vim net-tools git

#升级内核（适用centos7）
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum list available --disablerepo="*" --enablerepo="elrepo-kernel"
yum install -y --enablerepo=elrepo-kernel kernel-lt
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
rpm -qa|grep kernel
package-cleanup --oldkernels

#关闭防火墙、selinux、swap、开启iptables并清空规则
systemctl stop firewalld && systemctl disable firewalld && systemctl status firewalld
yum -y install iptables-services && systemctl start iptables && systemctl enable iptables
iptables -F && iptables -Z && service iptables save
swapoff -a && sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab && free -h
setenforce 0 && sed -i '/^SELINUX/cSELINUX=disabled' /etc/selinux/config && getenforce

#将桥接的IPv4流量传递到iptables的链
cat>/etc/sysctl.d/k8s.conf<<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv6.conf.all.disable_ipv6=1
EOF
sysctl -p /etc/sysctl.d/k8s.conf

#kube-proxy开启ipvs的前置条件
modprobe br_netfilter
cat>/etc/sysconfig/modules/ipvs.modules<<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod +x /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
lsmod|egrep 'ip_vs|nf_conntrack_ipv4'

#做hosts解析
cat>>/etc/hosts<<EOF
172.16.1.133 server1
172.16.1.134 server2
172.16.1.135 server3
EOF

#日志使用journal
mkdir /var/log/journal
mkdir /etc/systemd/journald.conf.d
cat>/etc/systemd/journald.conf.d/99-prophet.conf<<EOF
[Journal]
Storage=persistent				#保存到磁盘
Compress=yes							#压缩历史日志
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000				
SystemMaxUse=10G					#最大占用10G空间
SystemMaxFileSize=200M		#单个日志200M
MaxRetentionSec=2week			#日志保存2周
ForwadToSyslog=no					#日志不转发到syslog
EOF
systemctl restart systemd-journald
```

##### 安装docker

```bash
#安装docker
yum install -y yum-utils device-mapper-persistent-data
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum remove -y docker docker-common docker-selinux docker-engine
yum list docker-ce --showduplicates 
yum install -y docker-ce-19.03.13
mkdir /etc/docker
cat>/etc/docker/daemon.json<<EOF
{
        "registry-mirrors": ["https://registry.docker-cn.com"],
        "exec-opts":["native.cgroupdriver=systemd"]
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload
systemctl start docker && systemctl enable docker && systemctl status docker

注意：
1.centos8无法直接安装containerd.io
yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/edge/Packages/containerd.io-1.2.13-3.1.el7.x86_64.rpm 

2.需要把docker设置为开机自启，否则后面执行kubeadm init时会报错

3.启动docker后报错：Your kernel does not support cgroup blkio weight_device
原因是kernel版本过高，建议使用长期维护的4.4版本内核
```

##### 安装k8s集群

```bash
建议采用kubeadm方式安装，因为二进制安装每个组件作为一个进程运行，没有自愈机制

#各节点安装k8s并把kubelet设为开机自启，不需启动，kubeadm init时要用
cat>/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg  http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubelet-1.18.10 kubeadm-1.18.10 kubectl-1.18.10
systemctl enable kubelet																		

#初始化主节点
kubeadm config print init-defaults>kubeadm-init.yml
==============
#kubeadm-init.yml文件需要修改自定义内容如下
localAPIEndpoint:
  advertiseAddress: 192.168.82.131													
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kubernetesVersion: v1.18.10
networking:
  serviceSubnet: 192.168.0.0/16
  podSubnet: 10.244.0.0/16
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
===========
kubeadm init --config=kubeadm-init.yml|tee /var/log/kubeadm-init.log
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#安装网络插件
k8s需要扁平化的网络，k8s支持的插件很多，如flannel、weave-net等，任选一种即可

1.flannel
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml

2.weave CNI
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

cp kubeadm-init.yml  weave-net.yml /usr/etc
#其它节点加入主节点(node节点只下载3个组件镜像：kube-proxy、pause、weave-kube)
kubeadm join 192.168.82.131:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash \ sha256:ea9b65186bed1f0d8346316d728f33a45045177b3eba8533f29c8f5722b0c3b1 
```

##### 部署harbor仓库

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

#下载harbor安装包并修改配置文件，处理
wget https://github.com/goharbor/harbor/releases/download/v2.0.1/harbor-offline-installer-v2.0.1.tgz
tar xf harbor-offline-installer-v2.0.1.tgz  -C /opt
cp /opt/harbor/harbor.yml.tmpl /opt/harbor/harbor.yml
sed -i '/^hostname/s#reg.mydomain.com#reg.longchuang.com#g' /opt/harbor/harbor.yml
sed -i 's#/your/certificate/path#/etc/docker/certs/reg.longchuang.com.crt#g' /opt/harbor/harbor.yml
sed -i 's#/your/private/key/path#/etc/docker/certs/reg.longchuang.com.key#g' /opt/harbor/harbor.yml
sed -i 's#Harbor12345#test#g' /opt/harbor/harbor.yml
cd /opt/harbor
./prepare
./install.sh

#用docker-compose编排工具，启动harbor服务并访问harbor站点
yum install -y docker-compose
docker-compose version
docker-compose -f /opt/harbor/docker-compose.yml up -d
http://reg.longchuang.com			用户名：admin				密码：test

---
其它操作：写入Harbor服务器证书文件
mkdir -p /etc/docker/certs.d/reg.longchuang.com
openssl s_client -showcerts -connect reg.longchuang.com:443 < /dev/null | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /etc/docker/certs.d/reg.longchuang.com/ca.crt
docker login 10.1.20.203:8088
```

## 三、Pod生命周期

##### Pod工作机制

```bash
启动pod时会立马启动一个pause容器和应用容器，应用容器共享pause容器的存储、网络及怎样运行这些容器的声明。apiServer接收到kubectl的指令后，把指令调度给kubelet，由kubelet操作CRI(container runtime interface)来让docker引擎完成容器生命周期管理，，过程包括：

1.容器环境初始化：通过InitC程序创建容器运行必要的文件或其它条件，多个initC串行执行，initC完成工作正常退出，退出码为0
2.start操作：在主容器运行开始运行时执行的第一条指令
3.运行主容器：每个pod可有一个或多个主容器
4.stop操作：在主容器准备退出前执行的指令，完成stop操作容器才能退出
5.readness：就绪检测，当容器建立后状态转为running(意思是可对外提供访问),但容器内进程未必全部加载成功，此时被用户访问就会报错，就绪检测是通过tcp或http探测容器内主程序状态，如果主程序ok，才把pod的状态变更为running，就绪检测可指定何时执行
6.liveness: 生存检测伴随容器生命周期，当容器内部不能正常对外提供服务时(如僵尸进程),生存检测根据容器状态重启或重建pod
```

##### Init容器

```bash
initC是Pod的初始化容器，用于在应用程序容器启动之前完成一些初始化操作，initC可有多个或没有也行，所有initC串行执行，执行完成正常退出，退出码为0；如果运行失败或非正常退出，会参考Pod的restartPolicy（默认always）来重启pod。pod重启时所有init容器必须重新串行执行。

1.运行一些实用工具，如sed、awk、pthon、dig等，多个应用程序都需要用这些工具，剥离出来可避免应用程序容器太大
2.init容器使用的是linux namespace，其文件系统实图不同于应用程序的，所以init容器可访问secret，举例来说，当mainC需要访问系统中某个目录下的文件，把整个目录的权限给mainC，则maniC可访问该目录下所有文件，不安全，此时把目录的权限给到init容器，mainC运行前由initC去该目录拿文件，再交给mainC即可
3.当pod里多个容器存在依赖关系，如mysql和nginx，可在nginx容器的initC中定义检测mysql是否正常，正常再起nginx
4.定义init容器不能用readness和liveness字段，因为init容器本身就是负责就就绪部分工作，且执行完就退出也不存活
5. initC的命名不能跟其它容器重名，且不可用特殊字符如-等
```

##### Pod示例

```bash

示例如下

#pod-init.yml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: mainc
    image: busybox
    command: ['sh', '-c','echo This app is running! && sleep 36000']
  initContainers:
  - name: initc1
    image: busybox
    command: ['sh','-c','until nslookup web1;do echo waiting for web1;sleep 4;done;']
  - name: initc2
    image: busybox
    command: ['sh','-c','until nslookup db1;do echo waiting for db1;sleep 4;done;']
#init-svc.yml
---
apiVersion: v1
kind: Service
metadata:
  name: web1
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
apiVersion: v1
kind: Service
metadata:
  name: db1
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9377
```

##### 探针

```bash
由应用程序容器的initC容器执行的就绪检测和存活检测存在局限性，因为initC在完成初始化后就退出了，无法持续，那么需要由应用程序容器来完成检测任务，具体是由各node节点的kubelet发起对容器的定期检测，由kubelet调用容器的handler执行探测方法

#探测方案
1.livenessProbe: 判断容器是否正常运行，伴随容器生命周期，循环检测，检测不成功则kubelet会杀死容器，再按重启策略执行
2.readnessProbe: 判断容器是否就绪，就绪指可以被外部访问，检测不成功则从对应的svc中删掉该pod的ip

#探测方法
1.ExecAction: 在容器内执行指定命令，返回码为0为正常，非0则不正常
2.TCPSockerAction: 对指定端口上容器的IP进行TCP检查，端口开的则正常
3.HTTPGetAction:对容器路径+端口发起httpGet请求，返回码大于等于200且小于400则正常

#就绪检测示例: readness.yml
apiVersion: v1
kind: Pod
metadata:
  name: readiness
spec:
  containers:
  - name: webapp
    image: docker.io/library/nginx:latest
    imagePullPolicy: IfNotPresent
    readinessProbe:
      httpGet:
        port: 80
        path: /index1.html
      initialDelaySeconds: 1
      periodSeconds: 3

绪论：      
kubectl create -f readness.yml 							#执行创建pod命令后状态为running，但READY状态为0/1，表示未就绪
kubectl describe pod readness								#查看pod详情，提示unhealth 404
kubectl exec -it readness -- /bin/bash			#进入容器
echo '123'>>/usr/share/nginx/html/index1.html #写入index1.html文件后，pod的READY状态转变为1/1
rm -fr /usr/share/nginx/html/index1.html		#再次删掉index1.html，此时pod的READY状态又变回0/1


#存活检测示例1： liveness-cmd.yml
apiVersion: v1
kind: Pod
metadata:
  name: liveness
spec:
  containers:
  - name: webapp2
    image: docker.io/library/nginx:latest
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh","-c","touch /tmp/live;sleep 20;rm -fr /tmp/live;sleep 360"]
    livenessProbe:
      exec:
        command: ["test","-e","/tmp/live"]
      initialDelaySeconds: 1
      periodSeconds: 3

#存活检测示例2： liveness-httpget.yml
apiVersion.spec.containers.livenessProbe
  httpGet:
    port: 80
    path: /index.html
  initialDelaySeconds: 1
  periodSeconds: 3
  timeoutSeconds: 10

#存活检测示例3： liveness-tcp.yml
apiVersion.spec.containers.livenessProbe
  initialDelaySeconds: 5
  tcpSocket:
    port: 8080
  periodSeconds: 3
  timeoutSeconds: 1
```

##### 启动和退出动作

```bash
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle
spec:
  containers:
  - name: app-lificycle
    image: docker.io/library/nginx:latest
    liftcycle:
      postStart:
        exec:
          command: ["/bin/sh","-c","echo postStart is ok >/usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","echo postStop is ok >/usr/share/message"]
```

##### pod的状态

```bash
Pending: 挂起，pod创建的任务已被k8s接受，但容器镜像尚未创建，等待调度pod、下载镜像等
Running: 运行，pod已经被调度到某个node，pod中所有容器都已创建好，且至少有一个容器正在启动或运行或重启
Succeeded: 执行成功并退出，退出状态码为0，一般出现在定时任务里
Failed: 执行失败并退出，退出状态码非0
Unknow: 无法获取pod状态，一般是因为master节点与pod所在节点失联导致
```

```bash
#查看帮助
kubectl explain pod.spec

#必须存在的属性
version:								#指定api的版本，基本是v1，可用kubectl api-versions查看所有版本		
kind:										#本yaml文件定义的资源类型，如pod、deployment、service等
metadata:								#元数据对象
  name:									#对象名称
  namespace:						#对象所属命名空间
spec:										#对象详情
  containers:						#对象的容器
    name:								#对象的容器名
    images:							#对象的容器用到的镜像
    imagesPullPolicy:		#拉取镜像策略，默认Always，还有Nerver、IfNotPresent
    command:						#指定容器启动命令，List型
    args:               #指定容器启动命令的参数 List型
    workingDir:	        #指定容器的工作目录
    volume:
      volumeMounts:			#
        name:						#被容器挂载的存储卷名称
        mountPath:			#被容器挂载的存储卷路径
        readOnly:				#该存储卷读写模式，默认false
    ports:							#配置容器需要用到的端口列表
      name:							#端口名
      containerPort:		#容器要监听的端口
      hostPort:					#容器所在主机需要监听的端口，默认跟containerPort一致。设置了本项，该容器就不能起多个副本
      protocol:					#端口协议，TCP或UDP
    env:
      name:
      value:
    resource:
      limits:
        cpu:
        memory:
        requests:
          cpu:
          memory:
  restartPolicy: 				#pod的重启策略，默认Always，只要容器挂了就拉起来，OnFailure以非零码挂了才拉起来
  nodeSelector:					#用键值对指定node的过滤标签
  imagePullSecrets:			#定义pull镜像时使用的secret名称
  hostNetwork:					#是否使用主机网络模式，默认false表示使用docker网络，true是使用主机网络
```

## 四、K8S资源控制器

##### 资源分类

```bash
#按命名空间分类
1.工作负载型：pod、ReplicaSet、Deployment、StatefulSet、DaemonSet、Job、CronJob
2.服务发现及负载型：Service、Ingress
3.存储型：Volume、CSI
4.特殊类型：ConfigMap、Secret、DownwardAPI

#按集群级别分类
不管在哪定义，其它命令空间里都能看得到
Namespace、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding

#元数据类型
提供指标，如HPA，按cpu指标实现平滑扩展
HPA、PodTemplate、LimitRange等
```

##### 概述

```bash
Deployment: 为RS和RC提供声明式定义方法，Deployment创建RS，让RS创建pod，可实现滚动升级、扩缩容、暂停或继续等操作
RS：功能跟RC一致，多一个selector功能（根据标签来选择pod）
DaemonSet：管理守护进程的pod，每个daemonSet只能在指定的节点上运行一个pod
Job：负责运行脚本，如果脚本执行完没有以0代码退出，则重新执行，有一定的纠错能力
Cronjob：负责周期性运行脚本
StatefullSet：部署有状态服务，pod重建后保持原来的网络、存储，并有序扩缩容（基于init容器实现）
Horizontal Pod Autoscaling : HPA，非控制器但可以通过管理控制器来实现自动扩缩pod

Deployment是申明式编程，最好用kubectl apply -f foo.yml来创建
RS是命令式编程，最好使用kubectl create -f foo.yml来创建
```

##### ReplicaSet

```bash
通过标签维持期望的副本数，当有pod的标签被改了，会用原来的标签再生成一个pod以维持该标签下pod的数量，当删掉所有pod时，rs管理的pod不会被删，只有删掉该rs时，rs管理的所有pod才会被删掉

#ReplicaSet
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-demo
spec:
  replicas: 3
  selector:
     matchLabels:
       tag: foo
  template:
    metadata:
      labels:
        tag: foo
    spec:
      containers:
      - name: nginx-1
        image: nginx

#相关命令
kubectl get pod --show-labels
kubectl label pod rs-demo-4lj8r tag=foo2 --overwrite=true
```

##### Deployment

```yml
相比于RS控制器，Deployment控制器可以实现升级与回滚操作，Deployment的工作机制是通过管理RS，让RS来操作Pod

#Deployment
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      tag: foo
  template:
    metadata:
      labels:
        tag: foo  
    spec:
      containers:
      - name: nginx
        image: nginx

kubectl edit deploy deploy-demo
kubectl scale deploy deploy-demo --replicas=5

#滚动升级与回滚
deployment可实现升级与回滚，通过新建一个RS资源，先在旧RS中删掉25%的pod，再到新RS里新建25%的pod，4次操作完成升级；
当有多个rollout并行时，比如10个pod正在执行升级，升级了4个的时候又执行了rollover命令，此时不会等剩余的6个pod完成操作，而是改变航向，直接删除已升级的4个pod，再执行rollover

kubectl create -f deploy1.yml --record											#创建时要添加参数 --record
kubectl set image deploy deploy1 nginx=docker.io/nginx:v2		#升级名为nginx的容器的镜像到v2版本
kubectl set image deploy deploy1 nginx=docker.io/nginx:v3		#再升级到v3版
kubectl rollout undo deploy deploy1													#回滚到上一版
kubectl rollout status deploy/deploy1												#查看是否回滚成功
kubectl rollout history deploy/deploy1							 				#查看回滚历史
kubectl rollout undo deploy deploy1 --ro-version=2					#回滚到指定版本
```

##### DaemonSet

```bash
每个DaemonSet确保指定的pod上运行且只运行一个pod副本，符合条件的node节点加入集群时自动运行该DaemonSet管理的pod，只有daemonSet移除时，其管理的pod全部移除，通常可用于运行守护进程，如glusterd、ceph、fluentd、logstash、prometheus的客户端

#DaemonSet
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-demo
spec:
  selector:
    matchLabels:
      tag: ds
  template:
    metadata:
      labels:
        tag: ds
    spec:
      containers:
      - name: web1
        image: nginx
```

##### Job与CronJob

```bash
Job资源可用于执行一些脚本

#Job
---
apiVersion: batch/v1
kind: Job
metadata:
  name: job-demo
spec:
  template:
    metadata:
      name: pai
    spec:
      containers:
      - name: pai
        image: perl
        commands: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
       restartPolicy: Never


#cronjob
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-demo
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cname
            image: busybox
            args:
            - /bin/sh
            -  -c
            - date;echo hello from k8s-cluster
          restartPolicy: OnFailure
```

## 五、Service服务发现

##### SVC作用与类型

```bash
控制器可以通过标签维持一组pod的副本数，但是想把这一组pod作为一个集群来轮询访问就需要借助SVC，因为每个pod都有自己的IP，通过Pod的IP来访问明显不现实。SVC的工作机制是通过标签选择Pod进服务队列，就算有Pod挂了重启IP变了，SVC只要检测到Pod的标签，就把把该Pod重新纳入服务队列，并更新SVC的记录。缺陷：SVC默认只提供四层负载均衡，即通过ip和端口做转发，不可通过域名和host转发

#SVC的4种类型
1.ClusterIP：默认型，给SVC分配一个虚拟IP，集群内其它应用程序可以访问该虚拟IP来轮询访问该SVC通过标签选择到的Pod
2.NodePort：为SVC指定一个端口如80，匹配node的某端口如3001，通过访问NodeIP:3001就能进入SVC再轮询
3.LoadBalancer：借助云供应商的负载均衡器如LSB，对不同node+端口做负载
4.ExternalName：k8s集群想访问外部的mysql集群，只需要在k8s集群里定义一个连接mysql的svc，php的pod都连接这个svc
```

##### IPVS工作机制

```bash
ipvs是1.14版本后默认的代理模式，不再使用iptalbes和userspace，因为userspace需要用kube-proxy代理转发流量，而iptalbes效率不如ipvs，IPVS工作机制如下：

#IPVS工作机制
1.管理员通过kubectl向apiServer发送创建SVC指令，apiServer把请求写入etcd数据库
2.Kube-proxy监控Etcd的变化，把得到的变化通过调用netlink接口，写入本Node节点的IPVS规则
3.IPVS使用NAT技术把虚拟IP的流量转至Endpoint中

注意：
1.kube-proxy启动时默认以为配置发IPVS模块，如果没检测到就降级使用iptalbes代理模式
2.IPVS负载均衡调度算法有：rr轮询、lc最小连接数、dh目标哈希、sh源哈希、sed最短期望延迟、nq不排队调度
```

##### SVC四种类型示例

```yml
#svc-clusterip.yml
--
apiVersion: v1
kind: Service
metadata:
  name: svc-clusterip
spec:
  type: ClusterIP
  selector:
    tag: foo
  ports:
  - name: http
    port: 80				#ClusterIP的端口
    targetPort: 80
    
#svc-headless.yml
如果ClustIP不需要轮询时，可在svc.yml中指定ClusterIP的值为none来创建Headless Service，headless服务不会创建ClusterIP，但是可以根据域名来访问，用于实现有状态服务

---
apiVersion: v1
kind: Service
metadata:
  name: svc-headless
spec:
  clusterIP: "None"
  selector:
    tag: foo
  ports:
  - name: http
    port: 80
    targetPort: 80

dig -t A svc-headless.default.svc.cluster.local @10.32.0.3
svc-headless.default.svc.cluster.local. 30 IN A 10.44.0.3
svc-headless.default.svc.cluster.local. 30 IN A 10.44.0.4
svc-headless.default.svc.cluster.local. 30 IN A 10.44.0.1

#svc-nodeport.yml
可通过node节点的IP加svc暴露的端口来轮询访问后端服务
---
apiVersion: v1
kind: Service
metadata:
  name: svc-nodeport
spec:
  type: NodePort
  selector:
    tag: foo
  ports:
  - name: http
    port: 80
    targetPort: 80

svc-nodeport   NodePort    10.107.70.170   <none>        80:32280/TCP   3m47s
TCP  172.16.1.132:32280 rr
  -> 10.44.0.1:80                 Masq    1      0          0         
  -> 10.44.0.3:80                 Masq    1      0          0         
  -> 10.44.0.4:80                 Masq    1      0          0  
curl 172.16.1.131:32280


#svc-externalname.yml
咋对应的？
当k8s集群想访问集群外部的服务时，可在集群内生成一个svc，这个SVC对应yml文件中externalName指定的IP或域名，集群内部的服务只需要访问这个SVC，就可以访问到外部的服务，如果外部服务发生变化，只需要修改SVC即可，而不需要改k8s集群内服务的配置

---
apiVersion: v1
kind: Service
metadata:
  name: svc-en
spec:
  type: ExternalName
  externalName: hub.longchuang.com
```

## 六、Ingress控制器

##### Ingress原理

```bash
SVC和Pod的IP仅在集群内部访问，当集群外部的请求想访问k8s集群内部的Pod就需要通过Ingress。请求打到ingress后转发到SVC，交由Kube-proxy通过IPVS转发到Pod。Ingress为进入集群的请求提供路由规则，Ingress可以给SVC提供集群外部访问的URL、负载均衡、SSL、HTTP路由等，其工作机制是：

1.安装好Ingress-nginx的官方插件后，在ingress-nginx命名空间下，运行一个Pod，名为ingress-nginx-controller，在这个Pod跑Nginx做负载均衡，用户请求这个Pod的Nginx，nginx根据规则分发请求到不同的SVC
2.Pod/ingress-nginx-controller中nginx分发规则是由在default空间下ingress对象提供，即ingress对象负责生成nginx配置文件，然后Pod/ingress-nginx-controller监听到ingress和SVC的变化后把ingress生成的配置文件写入Pod里并使之生效

```

##### ingress实现http转发

```bash
 #第一步：安装Ingress控制器
官网：https://kubernetes.github.io/ingress-nginx/deploy/
阿里镜像：registry.cn-shanghai.aliyuncs.com/nginx-ingress-controller/nginx-ingress-controller:0.30.0

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/baremetal/deploy.yaml
kubectl apply -f deploy.yaml

#第二步：准备两个SVC
#server1.yml 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy1
spec:
  replicas: 3
  selector:
    matchLabels:
      tag: v1
  template:
    metadata:
      labels:
        tag: v1
    spec:
      containers:
      - name: nginx
        image: wangyanglinux/myapp:v1
---
apiVersion: v1
kind: Service
metadata:
  name: svc1
spec:
  type: ClusterIP
  selector:
    tag: v1
  ports:
  - name: http
    port: 80
    targetPort: 80

#server2.yml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy2
spec:
  replicas: 3
  selector:
    matchLabels:
      tag: v2
  template:
    metadata:
      labels:
        tag: v2
    spec:
      containers:
      - name: nginx
        image: wangyanglinux/myapp:v2
---
apiVersion: v1
kind: Service
metadata:
  name: svc2
spec:
  type: ClusterIP
  selector:
    tag: v2
  ports:
  - name: http
    port: 80
    targetPort: 80
#第三步：创建Ingress规则
ingress-rules.yml 
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress1
spec:
  rules:
  - host: www1.abc.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc1
          servicePort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress2
spec:
  rules:
  - host: www2.abc.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc2
          servicePort: 80

#第四步：访问
通过访问ingress-nginx-controller的Pod所在节点IP和ingress-nginx-controller的SVC所暴露的端口就可以了
```

##### ingress实现https

```bash
1.自签证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=nginxsvc/0=nginxsvc"
kubectl create secret tls tls-secret --key tls.key --cert tls.crt

2.起server3
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy3
spec:
  replicas: 3
  selector:
    matchLabels:
      tag: v3
  template:
    metadata:
      labels:
        tag: v3
    spec:
      containers:
      - name: nginx
        image: wangyanglinux/myapp:v3
---
apiVersion: v1
kind: Service
metadata:
  name: svc3
spec:
  type: ClusterIP
  selector:
    tag: v3
  ports:
  - name: http
    port: 80
    targetPort: 80
    
3.配置ingress规则
ingress-rule-https.yml 
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress3
spec:
  tls:
  - hosts:
    - www3.abc.com
    secretName: tls-secret
  rules:
  - host: www3.abc.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc3
          servicePort: 80
          
4.访问
kubectl get pod -o wide -n ingress-nginx	#查看ingress所在机器的IP
kubectl get svc -n ingress-nginx					#查看443对应的端口
www3.abc.com:32060				#做好hosts解析后（172.10.30.187  www3.abc.com）187是node2的IP，通过浏览器访问
```

##### ingress实现auth认证

```bash
通过创建一个ingress-auth对象，来修改ingress-controller中nginx访问规则，这样可以实现用户认证

1.准备认证的用户
yum install -y httpd
htpasswd -c auth foo
kubectl create secret generic basic-auth --from-file=auth

2.生成ingress-auth对象
ingress-rule-auth.yml 
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-auth
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - foo' 
spec:
  rules:
  - host: www4.abc.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc2
          servicePort: 80
kubectl apply -f ingress-nginx-auth.yml
www4.abc.com:32435					#做好hosts解析【172.10.30.187 www4.abc.com】然后浏览器访问
```

##### ingress实现rewrite

```bash
#常用配置
nginx.ingress.kubernetes.io/rewrite-target: 新地址			#指定重定向的目录url
nginx.ingress.kubernetes.io/ssl-redirect:	true				#指未是否位置位置可访问SSL
nginx.ingress.kubernetes.io/force-ssl-redirect:	true	#是否强制重定向到https,即使ingress未启动TLS
nginx.ingress.kubernetes.io/app-root: 								#指定controller必须重定向的应用程序根，如果它在'/'上下文中
nginx.ingress.kubernetes.io/use-regex: true						#指定ingress上定义的路径是否使用正则表达式

#示例：ingress-rule-rewrite.yml 
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-rewrite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: https://www4.abc.com:32435
spec:
  rules:
  - host: www5.abc.com
    http:
      paths:
      - path: /
        backend:
          serviceName: svc2
          servicePort: 80
```

## 七、K8S存储

##### 存储类型及作用

```bash
ConfigMap: 存储配置文件
Secret: 存储加密信息，比如用户名、密码、密钥等
Volume: 共享存储
Persistent Volume: 持久卷
Persieten Volume Service: 
```

##### ConfigMap

```bash
ConfigMap在k8s1.2版本中引入的，用于存储配置文件。由于许多应用程序需要从配置文件、命令行或环境变量中读取配置信息，批量去修改pod配置显示不方便，ConfigMap可用于保存单个属性或整个配置文件甚至json格式的二进制大对象，再由ConfitMap API向容器中注入配置文件信息

#四种创建ConfigMap的方式
1. 通过目录创建，目录下文件名是key，文件内容是value
kubectl create configmap  cm-demo --from-file=/mnt/config		#创建
kubectl get cm cm-demo -o yaml															#查看
kubectl describe cm cm-demo																	#查看

2.通过文件创建，可多次使用--from-file来创建多个cm，效果跟用目录创建一样
kubectl create cm cm-file --from-file=/root/config/game.properties
kubectl describe cm cm-file 

3.使用字符串创建
kubectl create cm cm-person --from-literal=persion.name=zhouxy  --from-literal=persion.age=28
kubectl describe cm cm-person

4.通过yml文件创建
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm1
data:
  name: zhouxy
  age: "18"
  sex: male

#调用ConfigMap
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx
    image: wangyanglinux/myapp:v1
    command: [ "/bin/sh","-c","env" ]
    env:
    - name: NAME
      valueFrom:
        configMapKeyRef:
          name: cm1
          key: name
  restartPolicy: Never

#在数据卷里使用CM，以文件的形式挂载到数据卷。ConfigMap中的每个键值对组成一个文件，文件名为键名，文件内容为该键的值
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: nginx
    image: wangyanglinux/myapp:v1
    command: ["/bin/sh","-c","sleep 600s"]
    volumeMounts:
    - name: volume1
      mountPath: /mnt/config
  volumes:
  - name: volume1
    configMap:
      name: cm2
  restartPolicy: Never
  
  
#ConfigMap热更新：在deploy资源中挂载使用ConfigMap，完成后通过修改ConfigMap就可以修改挂载到Pod里/mnt/config/whoami的内容
注意：修改ConfigMap并不会触发Pod的滚动更新，此时可通过修改pod annotations的方式强制触发滚动更新，命令如下
kubectl patch deploy deploy1 --patch '{"spec":{"template":{"matadata":{"annotations:"{"version/config":"20201118"}}}}}'

#deploy1.yml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm1
data:
  whoami: zhouxy
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy1
spec:
  replicas: 1
  selector:
    matchLabels:
      tag: v1
  template:
    metadata:
      labels:
        tag: v1
    spec:
      containers:
      - name: cname
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
        volumeMounts:
        - name: volume1
          mountPath: /mnt/config
      volumes:
      - name: volume1
        configMap:
          name: cm1
kubectl exec deploy1-57bd694fb5-9522k -it -- cat /mnt/config/whoami
kubectl edit cm cm1				=>修改zhouxy 为 zhouxiaoyong后等10秒再通过上条命令查看
```

##### Secret

```sh
ConfigMap保存的信息都是明文的，通过查看cm就能看到，如果想保存密码、密钥、token等敏感配置信息就需要用Secret，Secret可用Volume或环境变量的方式使用，Secret有三种类型
1.ServiceAccount: 自动创建，挂载到/run/secrets/kubernetes.io/serviceaccount中，当Pod访问ApiServer时做验证
2.Opaque: 用base64编码加密，用于存储密码密钥，但通过命令echo 'abc'|base64 -d，就能解密，安全性不高
3.kubernetes.io/dockerconfigjson:用于存储私有docker registry的认证信息


#将Secret挂载到volume中使用，先生成secret1，再挂载使用secret1，生成pod后在该pod容器内部挂载目录下生成两个文件都是以secret1里的键名命令，文件内容是解密后的键值
---
apiVersion: v1
kind: Secret
metadata:
  name: secret1
type: Opaque
data:
  username: emhvdXh5Cg==
  password: WCFAb3kwbmcK
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx
    image: wangyanglinux/myapp:v1
    volumeMounts:
    - name: volume1
      mountPath: /etc/secret
      readOnly: yes
  volumes:
  - name: volume1
    secret:
      secretName: secret1
kubectl exec pod1 -it -- cat /etc/secret/password
kubectl exec pod1 -it -- cat /etc/secret/username
 

#将Secret导入到环境变量中使用,把secret中的key的值赋给env中的新定义的变量PASSWORD
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy1
spec:
  replicas: 2
  selector:
    matchLabels:
      tag: v1
  template:
    metadata:
      labels:
        tag: v1
    spec:
      containers:
      - name: nginx
        image: wangyanglinux/myapp:v1
        ports:
        - containerPort: 80
        env:
        - name: USERNAME
          valueFrom:
            secretKeyRef:
              name: secret1
              key: username
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: secret1
              key: password

#当从私有仓库如Harbor上下载镜像时，用户名和密码的认证信息需要写进Secret
kubectl create secret docker-registry logindocker  --docker-server=reg.longchuang.com --docker-username=zhouxy --docker-password=test --docker-email=abc@qq.com

---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: foo
    image: hub.longchuang.com
  imagePullSecrets:
  - name: logindocker
```

##### Volume

```sh
Pod生命周期短，一旦消亡重建，kubelet会重新执行InitC，让镜像还原到最初状态，容器之前产生的数据就会丢。Volume可用于做容器数据的持久化，具体是在定义Pod时挂载一个volume，这个volume跟pause容器绑定，Pod重启后pause容器找到绑定的卷，以实现数据持久化存储。volume的生命周期跟Pod相同，删除pod时volume也会被清除

#空卷：可让同一个Pod内的多个容器共享存储目录，具体实现方法是：在Pod内生成一个空卷，多个应用容器都挂载到这个空卷即可，一般空卷可用于保存管理容器需要用的文件等。如下示例，在Pod1中生成空卷volume1，应用容器nginx的/bar目录和tomcat容器的/foo目录都挂载到volume1，此时这两个目录的存储是共享的
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx
    image: wangyanglinux/myapp:v1
    volumeMounts:
    - mountPath: /bar
      name: volume1
  - name: tomcat
    image: tomcat:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /foo
      name: volume1
  volumes:
  - name: volume1
    emptyDir: {}

kubectl exec pod1 -c nginx --  mkdir /bar/zhouxy		#在nginx容器里新建目录
kubectl exec pod1 -c tomcat -- ls -l /foo						#在tomcat容器里能看到

#HostPath：把主机节点的文件或目录挂载到集群中，通过把集群内部的共享存储挂载到各个Node节点的某目录，然后用HostPath方式把容器内的Pod挂载到Pod所在节点的该目录，要注意如下事项：
1.每个Node节点都需要新建相同的挂载目录，不然pod消亡了调度到别的节点就会找不到挂载目录
2.Node节点上创建的挂载目录需要对kubelet开放读写权限
3.cronjob无法感知HostPath挂载的资源
示例
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: nginx
    image: wangyanglinux/myapp:v1
    volumeMounts:
    - name: volume2
      mountPath: /zhouxy
  volumes:
  - name: volume2
    hostPath:
      type: Directory
      path: /data
```

##### PV与PVC

```bash
Pod虽有volume实现数据持久化但仍有不足，比如Deploy管理的Pod滚动升级时，Node节点故障、多个Pod共享存储、对volume扩容缩容、快照等都无法实现。通过PV和PVC把存储作为一个对象，给Pod与volume的生命周期解耦

#持久化流程
配置底层文件系统 -> 创建PV资源 -> 创建PVC来给Pod跟数据卷做关联，通过PV和PVC可将Pod与数据卷解耦
PV（PersistentVolume）：PV是提供集群的网络存储资源，所有命名空间共用
PVC（PersistenVolumeClaim）：PVC请求存储资源，创建Pod时在yml文件里指定
持久卷声明的保护：当启用了PVC保护alpha功能时，当用户删除Pod正在使用的PVC，该PVC会等没有Pod使用该PVC时才自动删掉


#Volume生命周期
|Provisioning|Binding|Using|Releasing|Reclaiming|Deleting|
|	Available	 |<- Bound	-> |Realeased|<-   faild   		 ->|


#PV访问模式，PVC通过存储大小和访问模式来选择绑定的PV
ReadWriteOnce(RWO):是最基本的方式，可读可写，但只支持被单个节点挂载
ReadOnlyMany(ROX):可以用只读的方式被多个节点挂载 
ReadWriteMany(RWX):这种存储可以用读写的方式被多个节点共享


#PV的回收策略，即PVC释放卷的时候PV需要做什么操作
1.Retain: 不自动清理，保留Volume，需要手动清理
2.Recycle: 删除数据，相当于rm -fr /nfs/* ，只有NFS和hostPath支持这种策略
3.Delete: 删除存储卷






```

## 八、调度器

##### 调度策略

```bash
#调度过程
predicate:预选，过滤不满足条件的节点
priority: 排序，按优先级对节点排序
最后选择优先级最高的节点

#预选策略
PodFitsResources: 节点上剩余资源是否大小pod请求的资源
PodFitsHost: 如果Pod指定了NodeName，检查节点名称是否和NodeName匹配
PodFitsHostPorts: 节点上使用的Port是否和pod申请的port冲突
PodSelectorMatches: 过滤掉和pod指定的label不匹配的节点
NoDiskConflict: 已经mount的volume和pod指定的volume不冲突，除非它们都是只读

#排序策略
LeastRequestedPriority: 剩余资源越多，权重越高
BalancedResourceAllocation: cpu和内存使用率越接近，权重越高，此项目配合上面的一起用，不能单独用
ImageLocalityPriority: 节点内镜像总大小越大，权重越高
```

##### node亲和性

```bash


```

##### Pod亲和性

```bash


```

##### 污点和容忍

```bash


```

##### 固定调度

```bash


```

## 九、安全策略

##### g 

```bash

```

## 十、Helm

##### 原理

```bash
helm是官方提供的类似于YUM的包管理器，helm本质上是k8s应用管理工具，可动态生成k8s的资源清单，然后调用kubectl自动执行
资源的部署。
helm是C/S架构，客户端是HelmClent，服务端叫Tiller。客户端负责创建并管理chart及release并通过gRPC与Tiller交互，Tiller服务器运行在k8s集群中，负责解释helmClient的请求，交给kube-api



```

##### 安装部署

```bash
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
tar xf helm-v2.13.1-linux-amd64.tar.gz
mv linux-amd64 /opt/helm
cp /opt/helm/helm  /usr/local/bin/



```

























