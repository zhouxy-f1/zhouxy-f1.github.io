## 一、K8S基础

##### k8s诞生背景

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

#SpringBoot与微服务间关系
微服务: 是一种架构设计风格，不取决于实现的技术方案。演化过程是从单体架构  -> 分布式架构  -> 微服务架构
SpringBoot: 是一种技术解决方案，使得构建和开发更快捷，微服务架构如果使用SpringBoot会更方便
SpringCloud: 技术解决方案全家桶，解决微服务架构问题，比如负载均衡、服务发现、网关服务等
业内：实现微服务架构，常用SpringBoot快速搭建web应用，用SpringCloud来解决架构其它问题
```

##### 组件图解说明

![image-20210220145910111](https://tva1.sinaimg.cn/large/008eGmZEgy1gnu0clyirdj30qe0e640j.jpg)

```bash
#概要
在Master节点上有ApiServer、controller manager、scheduler、etcd等组件，在Node节点上有kubelet、kube-proxy、docker引擎等组件。
高可用集群副本数最好大于等于3且为奇数个

1.ApiServer
ApiServer是各组件通信的中介，主要负责接收、校验并响应所有的rest请求，把结果状态存储到Etcd中。apiServer接收到kubectl的指令后，把指令调度给kubelet，由kubelet操作CRI(container runtime interface)来让docker引擎完成容器生命周期管理

2.Controller manager
控制器的管理器，主要负责管理各种控制器，如RC、RS、EP、NS、SVC等，以确保资源处于预期状态

3.Scheduler
调度器主要负责把Pod调度到最优的Node节点来运行，也可配置节点亲和、Pod亲和、固定节点调度等

4.Etcd
数据库，用于存储集群配置信息及各资源状态，当数据发生变化时，通过推送变更到api，由api通知客户端


5.Kube-proxy工作原理
kube-proxy主要用于实现负载均衡，实现过程是：当管理员通过kubectl创建SVC，apiServer接收到请求后写入etcd，kube-proxy监测etcd看到有本节点的请求，再根据请求写入本节点的IPVS或iptables规则，由IPVS来把来自该SVC的请求转发到后端Pod，完成负载均衡

6.kubelet
kubelet是node的agent，当Scheduler确定在某个Node上运行Pod后，会将Pod的具体配置信息（image、volume等）发送给该节点的kubelet，kubelet会根据这些信息创建和运行容器，并向master报告运行状态。


#其它插件
coredns:core公司的dns服务器，为svc创建hosts解析
dashboard:B/S架构，提供界面管理工具
ingress controller: 官方k8s使用4层负载，而ingress可实现七层代理（根据主机名或域名实现负载）
federation: 可以实现跨集群统一管理多套k8s集群
prometheus: 监控k8s集群
efk: 日志收集分析
```

##### k8s网络

```bash
#网络通讯
k8s的网络模型假定了所有pod都在一个可以直接连接的扁平化网络空间，pod与pod间可直接通过ip访问
1.同一pod内部：ip是共用pause容器的，通过localhost(回环口)+端口访问不同容器
2.pod1与pod2间：如果在同一台主机，直接通过docker0转发；如果不在就要通过overlay network
3.pod与service之间：通过各节点的iptalbes规则
4.pod访问外网：查找路由表转发数据包到宿主机的网卡，宿主机上路由选择后由iptalbes执行masquerade把源ip改为宿主机ip


#网络插件：overlay network
为了让集群中不同节点的主机创建的容器都具有全局唯一的虚拟ip，而且还能在这些ip间建立一个覆盖网络，这个覆盖网络就是overlay network，通过overlay network把数据包原封不同传递到目标容器内。覆盖网络有很多，如
1.Calico(最优选择，对linux8友好)
2.Flannel
CoreOS公司为k8s设计的一个网络规划服务,flannel与etcd间交互包括，存储管理flannel可分配的ip段资源；监控etcd中每个pod的实际ip，并在内存中建立维护pod节点的路由表等，由于其性能较差，一般不用
3.Weave net

```

##### k8s资源分类

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

#其它
DaemonSet：管理守护进程的pod，每个daemonSet只能在指定的节点上运行一个pod
StatefullSet：部署有状态服务，pod重建后保持原来的网络、存储，并有序扩缩容（基于init容器实现）
```

## 二、K8S集群部署

##### 系统初始化

```bash
#换阿里源并更新软件
`//linux7
rm -fr /etc/yum.repos.d/*
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all && yum makecache fast
yum install -y conntrack ntpdate ntp ipvsadm ipset jq wget vim net-tools git
`//linux8
gzip /etc/yum.repos.d/{oracle-linux-ol8.repo,uek-ol8.repo}
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-8.repo
sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
yum install -y https://mirrors.aliyun.com/epel/epel-release-latest-8.noarch.rpm
sed -i 's|^#baseurl=https://download.fedoraproject.org/pub|baseurl=https://mirrors.aliyun.com|' /etc/yum.repos.d/epel*
sed -i 's|^metalink|#metalink|' /etc/yum.repos.d/epel*
yum clean all && yum makecache

#做hosts解析
cat>>/etc/hosts<<EOF
192.168.1.11 server1
192.168.1.12 server2
192.168.1.13 server3
192.168.1.14 server4
192.168.1.15 server5
EOF

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
systemctl stop firewalld  && systemctl disable firewalld	&& systemctl status firewalld 
yum -y install iptables-services && systemctl start iptables && systemctl enable iptables && systemctl status iptables 
iptables -F && iptables -Z && service iptables save
swapoff -a && sed -i '/swap/s/^\(.*\)$/#\1/g' /etc/fstab && sysctl -w vm.swappiness=0 && free -h
setenforce 0 && sed -i '/^SELINUX/cSELINUX=disabled' /etc/selinux/config && getenforce

#加载CRI所需内核模块并设置开机自动加载
cat>/etc/modules-load.d/cri.conf<<EOF	
overlay
br_netfilter
EOF
modprobe br_netfilter
modprobe overlay
lsmod|egrep 'overlay|br_netfilter'

#修改内核参数，使桥接的IPv4流量传递到iptables的链
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system 2&>/dev/null
或者sysctl -p /etc/sysctl.d/99-kubernetes-cri.conf

#如果需要使用ip_vs则加载这些模块
modprobe ip_vs ip_vs_rr ip_vs_sh nf_conntrack_ipv4
cat<<EOF|tee /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_rr
ip_vs_sh 
nf_conntrack_ipv4
EOF
lsmod|grep 'ip_vs'

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

##### 部署docker

```bash
#安装docker
yum install -y yum-utils device-mapper-persistent-data
wget -O /etc/yum.repos.d/docker-ce.repo https://repo.huaweicloud.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+repo.huaweicloud.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
yum remove -y docker docker-common docker-selinux docker-engine
yum list docker-ce --showduplicates 
yum install -y docker-ce-18.09.9		
mkdir /etc/docker
cat>/etc/docker/daemon.json<<EOF
{
        "registry-mirrors": ["https://registry.docker-cn.com"],
        "exec-opts":["native.cgroupdriver=systemd"],
        "insecure-registries": ["reg.lianyu.com"]
}
EOF
systemctl daemon-reload
systemctl start docker && systemctl enable docker && systemctl status docker

注1:需要把docker设置为开机自启，否则后面执行kubeadm init时会报错

#Linux8安装containerd
containerd是容器环境，在centos7里，安装docker时会将其作为依赖程序直接安装上，但Linux8系统由于docer-ce的yum源中缺少contained，需要提前安装contained.io-1.4.3，再配置cgroup使用systemd来做资源隔离

yum localinstall https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.4.3-3.el7.x86_64.rpm
containerd config default > /etc/containerd/config.toml
sed -i '/systemd_cgroup/s#false#true#g' /etc/containerd/config.toml
systemctl restart containerd
yum install -y https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/edge/Packages/docker-ce-18.09.9-3.el7.x86_64.rpm
```

##### 部署k8s集群

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
yum install -y kubelet-1.18.13 kubeadm-1.18.13 kubectl-1.18.13
systemctl enable kubelet																		

#配置kubelet的cgroup驱动使用systemd,在第3行末尾引号外添加 --cgroup-driver=systemd
sed -i '3s#$# --cgroup-driver=systemd#g' /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl daemon-reload
systemctl restart kubelet

#初始化主节点
kubeadm config print init-defaults>kubeadm-init.yml

====== kubeadm-init.yml文件需要修改自定义内容如下
localAPIEndpoint:
  advertiseAddress: 192.168.82.131													
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kubernetesVersion: v1.18.13
networking:
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
======

kubeadm init --config=kubeadm-init.yml|tee /var/log/kubeadm-init.log
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
ll ~/.kube

#安装网络插件
k8s需要扁平化的网络，k8s支持的插件很多，如flannel、weave-net、calico等，任选一种即可
1.flannel
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml

2.weave CNI
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
cp kubeadm-init.yml  weave-net.yml /usr/etc

3.calico插件	（注意：calico默认给svc的网段是192.168.0.0/16，需要根据kubeadm-init.yml调整）
wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml
sed -i 's#192.168.0.0/16#10.96.0.0/12#g' calico.yaml 	
kubectl apply -f calico.yaml

排错：Readiness probe failed: calico/node is not ready:BIRD is not ready: BGP not established ...
原因：集群起了容器后无法通过curl ip:80，容器间也无法ping通，查得calico日志组件在未就绪状态。原因是官方提供的yaml文件中，ip识别策略（IPDETECTMETHOD）没有配置，即默认为first-found，这会导致一个网络异常的ip作为nodeIP被注册，从而影响node-to-node mesh。我们可以修改成can-reach或者interface的策略，尝试连接某一个Ready的node的IP，以此选择出正确的IP。
解决：在calico.yml中第606行，Cluster type to identify the deployment type下添加字段如下
            - name: IP_AUTODETECTION_METHOD
              value: "interface=ens.*"  # ens 根据实际网卡开头配置

#把节点加入master组建集群
主节点初始化完成后，安装好网络插件就可以通过token把node加入master来组建集群，初始化后token自动生成，命令写在日志的最后；但是此token有效期为24小时，失效后需要重新生成，命令如下:

kubeadm token create --print-join-command
kubeadm join 192.168.82.131:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash \ sha256:ea9b65186bed1f0d8346316d728f33a45045177b3eba8533f29c8f5722b0c3b1 

# 绑定权限，让部署的微服务能够调用Kubernetes的API接口
kubectl create clusterrolebinding gitlab-cluster-admin --clusterrole=cluster-admin --group=system:serviceaccounts --namespace=default


其它操作说明
1.生成node节点加入集群的token，也可以这样操作
kubeadm token create				#在master节点生成token
kubeadm token list					#查看token
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'			 #计算hash值

2.当需要清理集群时
kubeadm reset
kubeadm reset cleanup-node
etcdctl del "" --prefix
```

##### 命令补齐

```bash
cat>>~/.bash_profile<<EOF
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k
EOF
echo "alias k='kubectl'">>~/.bashrc
source ~/.bash_profile
source ~/.bashrc

#或者
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

##### 部署Dashbord

```bash
#deploy
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
kubectl apply -f recommended.yaml
#建用户
cat>admin-user.yml<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
kubectl apply -f admin-user.yml

#生成密钥
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"

#打开站点

查token
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

## 三、Pod详解

##### Pod-demo.yml

```yml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
spec:
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "foo.local"
    - "bar.local"
  hostNetwork: false
  initContainers:
  - name: init1
    image: busybox
    command: ['sh','-c','echo first initC']
  containers:
  - name: nginx
    image: nginx:latest
    imagePullPolicy: IfNotPresent
    lifecycle:
      postStart:
        exec:
          command: ["sh","-c","touch /usr/share/nginx/html/index.html"]
      preStop:
        httpGet:
          path: /
          port: 80
    ports:
    - name: http
      containerPort: 80
      hostPort: 80
      protocol: TCP
    volumeMounts:
    - name: time
      mountPath: /etc/localtime
      readOnly: true
    env:
    - name: PROFILES
      value: "test"
    - name: POD_NAME
    	valueFrom:
    	  filedRef:
    	    filedPath: metadata.name
    resources:
      requests:
        cpu: "50m"
        memory: "700Mi"
      limits:
        cpu: "2"
        memory: "700Mi"
    readinessProbe:
      tcpSocket:
        port: 80
      periodSeconds: 3
      timeoutSeconds: 8
    livenessProbe:
      httpGet:
        port: 80
        path: /
      initialDelaySeconds: 5
      periodSeconds: 3
      timeoutSeconds: 10
  restartPolicy: Always
  imagePullSecrets:
  - name: registry-harbor
  volumes:
  - name: time
    hostPath:
      path: /etc/localtime							#挂载本地时区目录，让pod的时区跟宿主机一致
      type: ""
```

##### Pod详解

```yml
#Pod工作机制
Pod的中文是豆荚，意指每个Pod由一个或多个容器组成。Pod是k8s对象中的最小单位，其工作机制是：apiServer接收到kubectl传来的创建Pod指令后，将指令写入etcd，kubelet从etcd里调取指令，并操作CRI来让docker引擎完成Pod的操作，操作过程是：首先创建Pause基础容器，对Pod的挂载卷和网络栈做初始化，各应用容器共用Pause的存储和网络；然后进入容器生命周期，即：
	init容器	->  postStart  -> 运行主容器 -> preStop  -> readiness -> liveness  

#Pause容器
kubelet每创建一个Pod，都会先创建一个pause基础容器，业务容器共享pause容器的网络栈和挂载卷。pause容器提供如下功能：
1.PID命名空间：Pod中的不同应用程序可以看到其他应用程序的进程ID
2.网络命名空间：Pod中的多个容器能够访问同一个IP和端口范围
3.IPC命名空间：Pod中的多个容器能够使用SystemVIPC或POSIX消息队列进行通信
4.UTS命名空间：Pod中的多个容器共用一个主机名，共用一个IP、共用Volumes（共享存储卷），容器间通过localhost+端口通讯

#Pod生命周期
InitC: 			通过InitC程序创建容器运行必要的文件或其它条件，多个initC串行执行，initC完成工作正常退出，退出码为0
postStart:	在主容器运行开始运行时执行的第一条指令
mainC:			每个pod可有一个或多个主容器
preStop:		在主容器准备退出前执行的指令，完成stop操作容器才能退出
readness:		就绪检测，在指定的时间通过Tcp或http探测容器内主程序状态，ok就running，不ok就重启Pod
liveness:	  生存检测，伴随容器生命周期，若容器不能对外提供服务(如僵尸进程),生存检测根据容器状态重启或重建pod

#Pod状态
Pending:   挂起，pod创建的任务已被k8s接受，但容器镜像尚未创建，等待调度pod、下载镜像等
Running:   运行，pod已经被调度到某个node，pod中所有容器都已创建好，且至少有一个容器正在启动或运行或重启
Succeeded: 执行成功并退出，退出状态码为0，一般出现在定时任务里
Failed:    执行失败并退出，退出状态码非0
Unknow:    无法获取pod状态，一般是因为master节点与pod所在节点失联导致

#Init容器
init容器是Pod的初始化容器，用于在应用程序容器启动之前完成一些初始化操作，如果有多个initC容器，必做串行，前一个执行完，并以0状态码退出后才执行下一个，如果执行失败就参考Pod的restartPolicy来重启Pod，重启后initC从头再来。initC执行完就退出，不可用readiness和liveness字段，常见用法有：
1.加载实用工具如sed、awk、dig等，避免应用程序容器过大
2.处理Pod内多个容器间依赖关系，比如pod内有mysql和nginx，可在nginx容器的initC中检测mysql是否正常，正常则启动nginx
3.解决应用容器对系统中权限不足问题，init容器使用的是linux namespace，权限高，当应用容器需要拿文件，可由init容器拿好交给应用容器
```

##### 探针

```bash
由应用程序容器的initC容器执行的就绪检测和存活检测存在局限性，因为initC在完成初始化后就退出了，无法持续，那么需要由应用程序容器来完成检测任务，具体是由各node节点的kubelet发起对容器的定期检测，由kubelet调用容器的handler执行探测方法

#探测方案
livenessProbe:   判断容器是否存活，伴随容器生命周期，循环检测，检测不成功则kubelet会杀死容器，再按重启策略执行
readnessProbe:   判断容器是否就绪，就绪指可以被外部访问，检测不成功则从对应的svc中删掉该pod的ip

#探测方法
Exec:      在容器内执行指定命令，返回码为0为正常，非0则不正常
TCPSocker: 对指定端口上容器的IP进行TCP检查，端口开的则正常
HTTPGet:   对容器路径+端口发起httpGet请求，返回码大于等于200且小于400则正常

#示例
spec:
  containers:
    readinessProbe:
      httpGet:
        port: 80
        path: /index1.html
    livenessProbe: 
      httpGet:
        port: 80
        path: /index.html
      initialDelaySeconds: 10		#容器启动10秒后再开始探测，默认10秒
      periodSeconds: 3					#每3秒探测一次，默认1秒
      timeoutSeconds: 5	   			#10秒没探测到结果就算超时，默认1秒
      successThreshold: 1				#自上次失败后，成功1次就算成功，默认值为1次
      failureThreshold: 3				#自上次成功后，失败3次才确认失败，默认值为3次
    livenessProbe:
      exec:
        command: ["test","-e","/tmp/live"]
    livenessProbe:
      tcpSocket:
        port: 8080
```

## 四、Pod编排

```sh
K8S中有很多编排相关的控制器资源，编排无状态应用有Deployment、RC、RS，编排有状态应用有statefulset，编排守护进程有daemonset，编排离线业务有job、cronjob，以下逐个分析
```

##### 升级回滚

```bash
#滚动升级与回滚
deployment可实现升级与回滚，通过新建一个RS资源，先在旧RS中删掉25%的pod，再到新RS里新建25%的pod，4次操作完成升级；
当有多个rollout并行时，比如10个pod正在执行升级，升级了4个的时候又执行了rollover命令，此时不会等剩余的6个pod完成操作，而是改变航向，直接删除已升级的4个pod，再执行rollout

#操作命令如下
kubectl apply -f nginx.yml --record										#--record用于记录CHANGE-CAUSE，从同一个yml创建的资源，升级回滚历史是一样的
kubectl set image deploy nginx nginx=nginx:1.18				#第1个nginx是deploy资源名，第2个nginx是容器的名字，第3个是镜像名
kubectl set image deploy nginx nginx=nginx:1.19				#再升级到1.19
kubectl rollout undo deploy nginx											#回滚到上一版
kubectl rollout undo deploy nginx --to-revision=2			#回滚到指定版本
kubectl rollout status deploy nginx										#查看是否回滚成功
kubectl rollout history deploy nginx							 		#查看回滚历史
```

##### 标签命令

```bash
#标签相关命令
kubectl get pod --show-labels													#查看标签
kubectl label pod nginx-nshgq app=nginx								#增加标签
kubectl label pod nginx-nshgq app-										#删除app标签
kubectl label pod nginx-6z7ll app=tomcat --overwrite	#修改标签
kubectl label pod --all app-													#删除所有pod的status标签
```

##### ipvs工作机制

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

##### ReplicaSet

```yml
RS是副本集，跟RC相比RS支持集合式selector，RS是命令式编程，最好使用kubectl create -f foo.yml来创建。RS是通过标签来选择一组Pod来统一管理，创建RS后，当有pod的标签被改了或消亡了，此时Pod的数量少于RS的期望数，RS会以selector中定义的标签再生成一个pod以维持该标签下pod的数量，当把其它Pod标签改成本RS的，此时Pod数多于期望值会把最年轻的pod删掉，只有删了RS时，才能删掉其管理的Pod

#rs-demo.yml
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
      - name: nginx
        image: nginx
        livenessProbe:
          httpGet:
            port: 80
            path: /
        volumeMounts:
        - name: time
          mountPath: /etc/localtime 
          readOnly: true
      volumes:
      - name: time
        hostPath:
          path: /etc/localtime
          type: ""
```

##### Deployment

```yml
Deployment可以为RS和RC提供声明式定义方法，Deployment创建RS，让RS创建pod，可实现滚动升级、扩缩容、暂停或继续等操作；相比于RS控制器，Deployment控制器可以实现升级与回滚操作，Deployment的工作机制是通过管理RS，让RS来操作Pod。Deployment是申明式编程，最好用kubectl apply -f foo.yml来创建

1. kubectl 发起一个创建 deployment 的请求
2. apiserver 接收到创建 deployment 请求，将相关资源写入 etcd；之后所有组件与 apiserver/etcd 的交互都是类似的
3. deployment controller list/watch 资源变化并发起创建 replicaSet 请求
4. replicaSet controller list/watch 资源变化并发起创建 pod 请求
5. scheduler 检测到未绑定的Pod资源，通过一系列匹配以及过滤选择合适的Node进行绑定
6. kubelet 发现自己Node上需创建新Pod，负责Pod的创建及后续生命周期管理
7. kube-proxy 负责初始化service相关的资源，包括服务发现、负载均衡等网络规则

#示例
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: longchuang-front-smartfarm-vue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: longchuang-front-smartfarm-vue
  template:
    metadata:
      labels:
        app: longchuang-front-smartfarm-vue
    spec:
      containers:
      - name: longchuang-front-smartfarm-vue
        image: reg.longchuang.com/uat/longchuang-front-smartfarm-vue:v1.1
        imagePullPolicy: Always
        env:
          - name: SPRING_PROFILES_ACTIVE
            value: "test"
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: 700Mi
          limits:
            cpu: "2"
            memory: 700Mi
        livenessProbe:
          httpGet:
            scheme: HTTP
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 20
          successThreshold: 1
        readinessProbe:
          httpGet:
            scheme: HTTP
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
          successThreshold: 1
      imagePullSecrets:
      - name: registry-harbor
```

##### DaemonSet

```yml
每个DaemonSet确保指定的pod上运行且只运行一个pod副本，符合条件的node节点加入集群时自动运行该DaemonSet管理的pod，只有daemonSet移除时，其管理的pod全部移除，通常可用于运行守护进程，如glusterd、ceph、fluentd、logstash、prometheus的客户端

#ds-demo.yml，ds里不可设置副本数，因为默认一个节点上只能起一个pod
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-demo
spec:
  selector:
    matchLabels:
      tag: foo
  template:
    metadata:
      labels:
        tag: foo
    spec:
      containers:
      - name: webserver
        image: wangyanglinux/myapp:v1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            port: 80
            path: /index.html
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: time
          mountPath: /etc/localtime
          readOnly: true
      volumes:
      - name: time
        hostPath:
          path: /etc/localtime
```

##### Job与CronJob

```yml
#Job用于实现仅执行一次的任务，比如数据库迁移等，JobController 负责根据 Job Spec 创建 Pod，并持续监控 Pod 的状态，直至其成功结束。如果失败，则根据 restartPolicy（只支持 OnFailure 和 Never，不支持 Always）决定是否创建新的Pod 再次重试任务。
---
apiVersion: batch/v1
kind: Job
metadata:
  name: job-demo
spec:
  completions: 4									#需要完成的任务数量，默认1
  parallelism: 3									#允许同时创建3个pod来并行处理任务，默认1
  activeDeadlineSeconds: 50				#超时50秒后，没完成任务的pod直接退出
  template:
    metadata:
      name: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["echo","hello"]
      restartPolicy: OnFailure

#CronJob 即定时任务，用于周期性执行job，到了指定的时间就新建一个job来执行任务
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

##### Service

```yml
控制器可以通过标签维持一组pod的副本数，SVC用于为这一组Pod提供一个统一的访问入口，供集群内部的服务来访问。SVC只通过标签来选择后端服务的Pod，当Pod挂了重启变了IP，SVC检测到Pod标签就会把该Pod重新纳入服务队列，哪怕是通过RS后来才新建的Pod打上SVC的标签也会被轮询到。目前SVC只能提供四层负载，由kube-proxy监控etcd变化，把变化通过调用netlink接口写入本Node节点的IPVS规则，IPVS使用NAT技术把虚拟IP的流量转发到Endpoint中。如果需要通过域名做负载需要用Ingress组件

#Endpoint
endpoint是k8s集群中的资源对象，存储在etcd中，当新建SVC时指定了selector的话，ep控制器会自动生成ep资源，记录该SVC对应的所有Pod的IP和端口


#ClusterIP类型：默认类型，给SVC自动分配一个仅集群内部可访问的虚拟IP，通过该虚拟IP和SVC里定义的port来访问后端服务。svc的定义可以通过yml或json，如下面的例子，定义一个名为svc-demo的svc，将svc的980端口的请求转发到后端带有tag=foo标签的pod的80端口。当服务需要多个端口时，每个端口要设置不同的名字，如http、https等。
---
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
    protocol: TCP
    port: 980				#ClusterIP的端口
    targetPort: 80	#Pod的端口
  - name: https
    protocol: TCP
    port: 9443
    targetPort: 443

#NodePort: 通过ClusterIP只能在集群内部访问，如果要暴露给公网可以用NodePort或LoadBalancer。NodePort是在ClusterIP的基础上再在宿主机上暴露一个nodePort，把来自nodeIP:nodePort的请求转发到ClusterIP:port，最后转发到后端服务Pod:targetPort
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
    targetPort: 80					#通过PodIP:80访问，80是pod的端口
    port: 8000							#用SVC的clusterIP:8000访问
    nodePort: 31002					#在宿主机上开的端口，如果不指定就会用5位数的随机端口，用节点的ip:31002访问Pod，转发到port

#ExternalName：当k8s集群想访问集群外部的服务时，可在集群内生成一个svc，这个SVC对应yml文件中externalName指定的IP或域名，集群内部的服务只需要访问这个SVC，就可以访问到外部的服务，如果外部服务发生变化，只需要修改SVC即可，而不需要改k8s集群内服务的配置
咋对应的？
---
apiVersion: v1
kind: Service
metadata:
  name: svc-en
spec:
  type: ExternalName
  externalName: hub.longchuang.com

#LoadBalancer：只适合运行在云平台上，在NodePort的基础上，借助LSB创建一个外部的负载均衡器，并将请求转发到 <NodeIP>:NodePort

#headless：用于创建有状态的服务。如果ClustIP不需要轮询时，可在svc.yml中指定ClusterIP的值为none来创建Headless Service，headless服务不会创建ClusterIP，但是可以根据域名来访问，用于实现有状态服务
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
```

##### Ingress

```yml
#Ingress原理
SVC和Pod的IP仅在集群内部访问，当集群外部的请求想访问k8s集群内部的Pod就需要通过Ingress。请求打到ingress后转发到SVC，交由Kube-proxy通过IPVS转发到Pod。Ingress为进入集群的请求提供路由规则，Ingress可以给SVC提供集群外部访问的URL、负载均衡、SSL、HTTP路由等，其工作机制是：

1.安装好Ingress-nginx的官方插件后，在ingress-nginx命名空间下，运行一个Pod，名为ingress-nginx-controller，在这个Pod跑Nginx做负载均衡，用户请求这个Pod的Nginx，nginx根据规则分发请求到不同的SVC
2.Pod/ingress-nginx-controller中nginx分发规则是由在default空间下ingress对象提供，即ingress对象负责生成nginx配置文件，然后Pod/ingress-nginx-controller监听到ingress和SVC的变化后把ingress生成的配置文件写入Pod里并使之生效

#安装Ingress控制器
官网：https://kubernetes.github.io/ingress-nginx/deploy/
阿里镜像：registry.cn-shanghai.aliyuncs.com/nginx-ingress-controller/nginx-ingress-controller:0.30.0

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/baremetal/deploy.yaml
kubectl apply -f deploy.yaml

#ingress实现http转发
1.先准备两个SVC
server1.yml 
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

server2.yml
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
    
2.创建Ingress规则
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

3.访问
通过访问ingress-nginx-controller的Pod所在节点IP和ingress-nginx-controller的SVC所暴露的端口就可以了

#ingress实现https
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

#ingress实现auth认证
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
www4.abc.com:32435					

4.访问，做好hosts解析【172.10.30.187 www4.abc.com】然后浏览器访问

#ingress实现rewrite
示例：ingress-rule-rewrite.yml 
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
常用配置
nginx.ingress.kubernetes.io/rewrite-target: 新地址			#指定重定向的目录url
nginx.ingress.kubernetes.io/ssl-redirect:	true				#指未是否位置位置可访问SSL
nginx.ingress.kubernetes.io/force-ssl-redirect:	true	#是否强制重定向到https,即使ingress未启动TLS
nginx.ingress.kubernetes.io/app-root: 								#指定controller必须重定向的应用程序根，如果它在'/'上下文中
nginx.ingress.kubernetes.io/use-regex: true						#指定ingress上定义的路径是否使用正则表达式
```

##### statefulSet

```yml
Deployment和ReplicaSet是部署的是无状态服务，而StatefulSet是为解决有状态服务的问题，通过把SVC的ClusterIP设置为None，不给SVC分配IP地址，用户的请求直接到Pod，可通过命令kubectl api-resources|grep statefulsets来查看sts资源简写，其应用场景包括

1.稳定的持久化存储: 即Pod重新调度后还是能访问到相同的持久化数据，基于PVC来实现
2.稳定的网络标志: 即Pod重新调度后其PodName和HostName不变，基于Headless Service（即没有ClusterIP的Service）来实现
3.有序部署，有序扩展: 即Pod是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依序进行（从0到N-1，在下一个 Pod 运行之前所有之前的Pod必须都是Running 和 Ready 状态），基于init containers来实现
4.有序收缩，有序删除: （即从 N-1 到 0）

#最佳实践
以挂载GlusterFS为例，首先需要创建glusterfs的卷，命名为vol-nginx;然后创建访问该卷的SVC和Endpoint服务，再根据EP和volume来创建PV和PVC，最后挂载使用PVC


1.创建gluster卷，brick不需要同名
mkdir /data/vol-nginx-bk1			#在server1上创建
mkdir /data/vol-nginx-bk2			#在server2上创建
mkdir /data/vol-nginx-bk3			#在server3上创建
gluster volume create vol-nginx replica 3 server1:/data/vol-nginx-bk1 server2:/data/vol-nginx-bk2 server3:/data/vol-nginx-bk3
gluster volume start vol-nginx
gluster volume list

2.创建SVC和EP，该SVC不会自动创建EP，需要手动创建
---
apiVersion: v1
kind: Service
metadata:
  name: glusterfs
spec:
  ports:
  - port: 24007
---
apiVersion: v1
kind: Endpoints
metadata:
  name: glusterfs
subsets:									#集合是ip和端口的笛卡尔积
- addresses:
  - ip: 172.16.1.136
  - ip: 172.16.1.137
  - ip: 172.16.1.138
  ports: 
  - port: 24007						#ep的端口
    protocol: TCP

3.创建PV和PVC
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nginx
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  glusterfs:
    endpoints: glusterfs
    path: vol-nginx
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nginx
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi

4、创建statefulset和svc，并挂载使用PVC
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  replicas: 3
  serviceName: webserver 			#负责管理pod的网络标识
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
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: aa
          mountPath: /usr/share/nginx/html
      volumes:
      - name: aa
        persistentVolumeClaim:
          claimName: pvc-nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: None
  selector:
    tag: foo
  ports:
  - name: http
    port: 80
    targetPort: 80
```

## 五、K8S存储

##### Volume

```yml
K8S提供了volume机制和丰富的插件来解决容器里的数据持久化以及容器间数据共享。跟docker不同的是，k8s里volume的生命周期跟Pod是绑定的，具体体现在当容器挂了重启后，volume的数据依然还在，只有删除Pod时，volume才会被删。目前K8S支持很多volume类型,常见如emptyDir、hostPath、glusterfs、local、secret、glusterfs、cephfs等等

#emptyDir: 当设置volume类型为emptyDir时，当Pod被分配到Node上了就会在Pod里创建一个空卷，Pod内应用容器都挂载到这个空卷即可实现Pod内的多个容器共享存储目录，容器内挂载目录不存在时会自动创建。emptyDir多用于保存管理容器需要的文件等
---
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: emptydir
      mountPath: /bar
  - name: tomcat
    image: tomcat:latest
    volumeMounts:
    - name: emptydir
      mountPath: /foo
  volumes:
  - name: emptydir
    emptyDir: {}
    sizeLimit: 64Mi
kubectl exec pod1 -c nginx -- mkdir /bar/zhouxy		#在nginx容器里新建目录
kubectl exec pod1 -c tomcat -- ls -l /foo					#在tomcat容器里能看到

#HostPath：挂载到主机，是把Pod的指定目录挂载到Pod所在节点的目录，需要在所有节点都创建相同的目录，否则pod调度到其它节点没有目录可挂载，而且Node节点上创建的挂载目录需要对kubelet开放读写权限；缺点：cronjob无法感知HostPath挂载资源
---
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

#nfs: 把pod里指定的目录直接挂载到nfs服务器的指定目录，这样就不需要在每个节点都新建相同的目录了，当pod重启后又会重新挂载到nfs服务端，数据不丢
---
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: suibian
        mountPath: /usr/share/nginx/html
      volumes:
      - name: suibian
        nfs:
          server: 172.16.1.138								#这是nfs服务器的地址
          path: /website											#nfs的挂载目录，需要提前新建并配置/etc/exports

#pvc挂载: 通过创建PV存储资源后，PVC需求匹配容量大小和访问方式来绑定PV，生成deploy资源时直接挂载使用PVC就行了
---
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: website
          mountPath: /usr/share/nginx/html
      volumes:
      - name: website
        persistentVolumeClaim:
          claimName: pvc-1

#glusterfs挂载:
---
        - containerPort: 80
        volumeMounts:
        - name: gluster-uat-data-volume
          mountPath: "/data"
      volumes:
      - name: gluster-uat-data-volume
        persistentVolumeClaim:
          claimName: glusterfs-uat-data
```

##### PV与PVC

```yml
Pod可通过Volume实现数据持久化但仍有不足，比如Deploy管理的Pod滚动升级、Node节点故障、多个Pod共享存储、对volume扩容缩容、快照等都无法实现。
K8S的做法是把存储做为一个对象，跟Pod分离开，流程如下：
首先: 创建PV资源来处理底层文件系统、创建数据卷等操作，从而提供网络存储资源；
然后: 创建PVC资源通过匹配存储的大小和访问模式来绑定PV；
最后: 创建deploy资源时挂载PVC资源就行了。
持久卷声明的保护:当启用了PVC保护alpha功能时，当用户删除Pod正在使用的PVC，该PVC会等没有Pod使用该PVC时才自动删掉

#PV示例
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 172.16.1.138
    path: /data
#PVC示例
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi

#accessModes: 指PV的访问模式，有3种
ReadWriteOnce（RWO）：是最基本的方式，可读可写，但只支持被单个节点挂载
ReadOnlyMany（ROX） ：可以用只读的方式被多个节点挂载
ReadWriteMany（RWX）：这种存储可以以读写的方式被多个节点共享
#persistentVolumeReclaimPolicy: 指PVC释放卷的时候PV的回收策略，有3种
Retain  ：不清理, 保留 Volume（需要手动清理）
Recycle ：删除数据，即 rm -rf /thevolume/* （只有 NFS 和 HostPath 支持） 
Delete  ：删除存储资源，比如删除 AWS EBS 卷（只有 AWS EBS, GCE PD, Azure Disk 和 Cinder 支持）
#volume生命周期
Provisioning：正在创建PV
Binding			：将PV分配给PVC
Using				：Pod通过PVC正在使用PV
Releasing		：Pod释放volume并删除PVC
Reclaiming	：回收PV
Deleting		：删除PV
#PV状态
Available ：可用
Bound     ：已分配给PVC
Released  ：跟PVC解绑但未执行回收策略
Failed    ：发生错误

#可将PVC写在deploy资源里
  template:
  volumeClaimTemplates:
  - metadata:
      name: pvc1
    spec:
      accessModes:
      - ReadWriteMany
      resources:
        requests:
          storage: 4Gi
```

##### ConfigMap

```bash
ConfigMap在K8S-1.2版本中引入的，用于存储配置文件。由于许多应用程序需要从配置文件、命令行或环境变量中读取配置信息，批量去修改pod配置显示不方便，ConfigMap可用于保存单个属性或整个配置文件甚至json格式的二进制大对象，再由ConfigMap API向容器中注入配置文件信息

#四种创建ConfigMap的方式
1. 通过目录创建，目录下文件名是key，文件内容是value
kubectl create configmap  cm-dir --from-file=/mnt/config		#创建
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

#deploy-demo.yml
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
  name: deploy-demo
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
        - name: volume-demo
          mountPath: /mnt/config
      volumes:
      - name: volume-demo
        configMap:
          name: cm-demo
kubectl exec deploy1-57bd694fb5-9522k -it -- cat /mnt/config/whoami
kubectl edit cm cm-demo				=>修改zhouxy 为 zhouxiaoyong后等10秒再通过上条命令查看
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
  name: secret-demo
type: Opaque
data:
  username: emhvdXh5Cg==
  password: WCFAb3kwbmcK
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
spec:
  containers:
  - name: nginx
    image: wangyanglinux/myapp:v1
    volumeMounts:
    - name: volume-demo
      mountPath: /etc/secret
      readOnly: yes
  volumes:
  - name: volume-demo
    secret:
      secretName: secret-demo
kubectl exec pod-demo -it -- cat /etc/secret/password
kubectl exec pod-demo -it -- cat /etc/secret/username
 

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
kubectl create secret docker-registry registry-harbor  --docker-server=reg.longchuang.com --docker-username=admin --docker-password=test --docker-email=abc@qq.com
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
  - name: registry-harbor
```



## 六、调度器

##### Sheduler原理

```bash
sheduler是k8s集群的调度器，运行在master节点上，一直监听ApiServer，获取Pod.Spec.NodeName为空的pod，通过创建binding把pod调度到指定的Node上运行，调度过程分三部，先过滤不满足条件的节点；再按优先级对节点排序，最后选择最优节点

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

##### node亲和

```bash
#节点亲和性，硬亲和是不满足条件就挂起，软亲和是表达倾向意愿，用权重表达意愿度。软硬亲和可同时使用
containers:
- name: nginx
  image: nginx
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: NotIn
          values: 
          - server1
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 8
      preference:
        matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - server2
    - weight: 2
      preference:
        matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - server3

#operator：键值运算关系（ Exists某个label存在；DoesNotExist某个labels不存在；In在列表；NotIn不在列表 Gt大于 Lt小于）

注意：
nodeSelectorTerms下有多个选项的话，只需要满足其一即可；matchExpressions下有多个选项需要全部满足才行
```

##### Pod亲和

```bash
#pod软硬亲和，operator键值运算关系有（ Exists某个label存在；DoesNotExist某个labels不存在；In在列表；NotIn不在列表 ）
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: NotIn
          values:
          - tomcat
      topologyKey: kubernetes.io/hostname
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - tomcat1
        topologyKey: kubernetes.io/hostname
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - tomcat2
        topologyKey: kubernetes.io/hostname
  podAntiAffinity: 
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 1
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - tomcat3
        topologyKey: kubernetes.io/hostname
```

##### 污点和容忍

```bash
节点亲和与pod亲和是把pod吸引到某类节点上运行，Taint则是让节点排斥指定的pod

#污点：通过用标签在node上打污点，表示包含该标签的pod不调度到本node节点，甚至驱逐出本节点
kubectl taint node server2 app=nginx:NoSchedule		#server2上打污点，让带有app=nginx标签的pod不能调度到server2
kubectl taint node server2 app=nginx:NoSchedule-	#删除server2上的污点
kubectl describe node server2|grep Taint					#查看server2上的污点

NoSchedule:表示在node打完污点后，带有污点对应标签的pod不会再调度到该node
PreferNoSchedule: 表示尽量不调度
NoExecute: 表示不再调度，且会把已经存在的pod驱逐走，即便是已经有pod在server2上，后来才在server2上打上污点，pod也会被驱逐走

#容忍：在创建deploy资源时，pod模板里配置该deploy资源对节点上污点的容忍，让pod可以调度到打了污点的节点上
根据operator的选项，可以有两种配置方案，

方案一：
tolerations:
- operator: Equal				  #当为Equal时，key、value和effect的值必须要跟node节点上的污点配置保持一致，表示容忍node上带有该key的污点；
  key: app
  value: nginx
  effect: NoExecute
  tolerationSeconds: 10		#只有当effect为NoExecute时才能添加此选项，作用是让将要被驱逐pod在当前节点保持运行10秒钟

方案二：
tolerations:
- operator: Exists			  #当operator指定为Exists时，表示容忍一类污点，如果只有这一项目没有下面的配置则表示容忍所有污点
  key: app|~							#可不写，或指定为空。指定时表示容忍key为app的污点，不写或指定为空时表示容忍所有key的污点
  value: ~								#当operator为Exists时，不可以指定value的值，可不写或指定为空
  effect: ~								#再比如不指定effect时表示容忍所有effect
  
  
#当有多个master时，最好做如下配置以免浪费资源
kubectl taint node server1 node-role.kubernetes.io/master=:PreferNoSchedule
```

##### 固定节点调度

```bash
1.当需要把pod调度到指定的节点时，只需要在pod模板里添加nodeName字段，这样会跳过调度器，强制匹配，只能把所有pod都调度到一个节点
spec:
  nodeName: server2 
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - operator: Exists
  
2.还有一种实现方式是通过在一个或多个node节点让打标签，然后在deploy资源里用nodeSelector来选择带此标签的node节点，可把pod调度到一组节点
kubectl label node server2 app=edit-tools
kubectl get node --show-labels
spec:
  nodeSelector:
    app: edit-tools
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - operator: Exists
```

## 七、安全认证

##### 认证

```bash
ApiServer是各组件通信的中介，k8s的安全机制是围绕保护APIServer来设计，通过认证（Authentication）、鉴权（Authorization）、准入控制（Admission Control）来保障其安全

#认证
需要跟ApiServer交互的组件有kubectl、scheduler、controller manager、kube-proxy、kubelet、pod(比如dashboard)等
1.ControlManager、Scheduler由于跟API在同一台主机，可以直接通过非安全端口访问 --insecure-bind-address=127.0.0.1
2.Kubectl、Kubelet、Kube-proxy都需要用证书进行https双向认证
3.Pod访问Api需要用ServiceAccount，因为Pod经常创建销毁，通过https来认证的话，每次创建都需要申请证书，资源浪费太多

#认证方式
用于在ApiServer和合法用户之间做认证，ApiServer识别合法用户，用户识别自己对应的ApiServer，认证方法有三种
1.Http Token
用户发起调用API请求时，在http请求头携带一个token，每个token对应一个用户名，token存储在API能访问到的文件中，实现单向认证
2.Http Base
通过用户名+密码形式，用户先用base64做加密得到哈希值，发起请求时把哈希值放在请求头的Heather Authorization域，服务端收到请求后通过echo '哈希值'|base64 -d来解码从而得到用户名和密码
3.Https
k8s采用http协议实现C/S结构的开发，天然适合通过https来通讯。具体是通过私有的CA中心来签发证书，各组件及部分Pod与API都向CA申请各自的证书，通过各自证书做双向认证后用随机字符制作对称密钥来通讯


#ServiceAccount
SA用于让pod中的容器访问ApiServer，如dashboard等容器。默认每个namespace下都会有一个SA,如果创建pod时没有指定namespace，就会使用pod所属的namaspace中的SA。SA中包含3个部分，分别是：
1.token: 是使用ApiServer私钥签名的JWT（json web token，是一种基于json开放标准RFC7519的token，该token安全紧凑，常用于分布式站点间传递被认证的用户身份信息）；
2.ca.crt: 是根证书，用于client端验证apiserver发送的证书
3.namespace: 用于标识service-account-token的作用域，service-account-token是secret资源对象


#其它
1.自动签发证书：kubelet首次访问APiServer时使用token认证，kubeadm token create --print-join-command生成token，node节点通过认证后controller manager会给kubelet生成一个证书，以后kubelet访问ApiServer就用证书来认证

2./root/.kube/config: 文件包含集群访问方式和认证信息，如ca.crt证书、apiserver地址、集群名、命令空间、用户名等还有客户端参数如证书私钥等

```

##### 鉴权

```bash
认证可以让通信双方确认对方是可信的，而鉴权是确定请求方有访问哪些资源的权限，ApiServer目前支持如下授权策略


#授权策略
1.AlwayDeny: 拒绝所有请求用于测试
2.AlwayAllow: 允许所有请求
3.ABAC(Attribute-Based Access Control): 基于属性访问权限，定义一个属性的访问类型，用户包含该属性就能访问对应的资源，修改后重启api生效
4.Webbook: 通过调用外部REST服务对用户进行授权；以上都不在生产中使用
5.RBAC(Role-Based Access Control): 基于角色访问控制，是1.5版本后的默认策略，RBAC对资源和非资源如pod状态等均有完整的覆盖，可通过kubecctl对RBAC操作，且在运行时调整无需重启API

#RBAC
基于角色的访问控制，可完整覆盖集群中的资源和非资源如pod状态，可用kubectl或api进行操作，无需重启ApiServer
API资源对象：

Role:	给角色赋予权限，再把Role有的权限通过RoleBinding赋予某用户，赋权时要指定命名空间
ClusterRole: 集群角色是没有命名空间的限制，可以把权限赋予给某个用户，用户就有该角色在所有命名空间下的权限
RoleBinding: 有命名空间的限制
ClusterRoleBinding: 无命名空间限制
```

##### 准入控制

```bash
#准入控制
准入控制是用一些插件，用于实现

```



![image-20201212231221309](https://tva1.sinaimg.cn/large/0081Kckwgy1gllh82mi9fj30m004ajtg.jpg)

## 八、Helm

##### 原理

```bash
helm是官方提供的类似于YUM的包管理器，helm本质上是k8s应用管理工具，可动态生成k8s的资源清单，然后调用kubectl自动执行
资源的部署。
helm是C/S架构，客户端是HelmClent，服务端叫Tiller。客户端负责创建并管理chart及release并通过gRPC与Tiller交互，Tiller服务器运行在k8s集群中，负责解释helmClient的请求，交给kube-api
```

##### 安装部署

```bash
wget https://mirrors.huaweicloud.com/helm/v2.15.0/helm-v2.15.0-linux-amd64.tar.gz
tar xf helm-v2.15.0-linux-amd64.tar.gz -C /opt
mv /opt/linux-amd64 /opt/helm
cp /opt/helm/helm  /usr/local/bin/
chmod +x /usr/local/bin/helm

cat>rbac-config.yml<<'EOF'
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF
kubectl create -f rbac-config.yml
helm init --service-account tiller --skip-refresh

#helm使用
helm version
```

## 九、K8S实现CICD

##### 准备maven项目

```bash
#在herelink-demo项目下新建文件controller/UserController，内容如下

package com.herelink.herelinkdemo.controller;
import org.springframework.web.bind.annotation.*;
@RestController
@RequestMapping(value = "/user",method = {RequestMethod.GET,RequestMethod.POST})
public class UserController {
    @RequestMapping("/hello")
    @ResponseBody
    public String hello(){
        return "Spring Boot Demo";
    }
}
```

##### springboot配置

```bash


#在启动jar包时指定
1.在Dockerfile中用
CMD java  -Xms700m -Xmx980m -Dspring.profiles.active="ENV_PROFILE" -jar /maven/herelink-demo.jar来指定

2.在Jenkinsfile里用sed替换
sed -i s#ENV_PROFILE#$BRANCH_NAME#g Dockerfile

#在deploy.yml中指定
1.在deploy.yml中用env传参
env:
  - name: SPRING_PROFILES_ACTIVE
    value: release

2.在配置中引用参数
application.yml如下
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE}
    
#docker启动时指定
docker run -di --name demo -p 80:80 --spring.profiles.active=xx  reg.hailian.com/herelink/herelink-demo:v1
```















