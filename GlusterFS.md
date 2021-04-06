## GlusterFS

##### 工作原理

```bash
NFS用于做共享存储，但也存在无法扩容、单点故障、容量过大时效率低下等问题。分布式存储可以把多个独立的服务器组织成一个超大的存储空间，当需要扩容时只需要新增节点，通过数据均衡把原节点的数据迁移部分到新节点；
GlusterFS各节点组成的存储空间称为受信任存储池pool，池中节点称为成员peer，peer中导出的目录就是brick，是最小的单元，通过主机名:目录名来标识，指定一组brick可组成一个全局的命名空间称为volume。客户端只需要挂载这个volume到本机的目录就可以使用该volume，而不需要知道数据存取过程

#工作原理
通过统一管理inode来做分布式的叫有元数服务器，如GFS、hdfs、moosfs、fastdfs等，而GlusterFS是无元数服务器，无元数据如何解决索引问题？
创建volume时，通过Davies-Meyer算法来生成2^32次的哈希区间，平均分配给各brick，每个brick占用一段哈希范围，新建文件时，用文件名做运算得到哈希值，存储到对应的brick，取用文件时方法一样
```

##### 安装部署

```bash
#所有centos7节点都安装
yum install centos-release-gluster
yum install -y gluster-server
systemctl start glusterd
systemctl enable glusterd
systemctl status glusterd

#Linux8安装GlusterFS
yum install -y oracle-gluster-release-rl8.x86_64
yum install -y glusterfs-server
systemctl start glusterd
systemctl enable glusterd
systemctl status glusterd

#配置glusterd主目录（按磁盘可用空间配置，当/home为主存储时按下面操作）
mkdir /home/glusterd
sed -i '/working-directory/s#/var/lib/glusterd#/home/glusterd#g' /etc/glusterfs/glusterd.vol

#配置hosts
cat>>/etc/hosts<<EOF
172.16.1.136 server1
172.16.1.137 server2
172.16.1.138 server3
EOF
```

##### 使用卷

```bash
#配置受信任池pool
受信任池就相当于一个集群，在server1上探寻成员server2后组成受信任池，server1和server2都是受信任成员，相互发现，添加新成员需要受信任成员将其探寻入池，反之新服务器无法探寻池内成员来进池，如下所示：在server1上探寻server2、3入池即可
gluster peer probe server2	#从server1探寻server2后，彼此发现，不用再在server2上探寻server1
gluster peer probe server3	#server1是信任成员，可探寻新成员入池
gluster peer detach server2	#pool中在任意节点上都可以删除其它成员
gluster pool list						#查看池内成员及其uuid、hostname、state

#配置volume卷
卷是砖块的集合，在各个成员服务器上新建相同路径的目录作为砖块，再把这些砖块组成卷。卷有很多种，比如分布式卷、复制卷、分布式复制卷、条带卷、分布式条带卷等，功能各有不同，以复制卷为演示示例：
clush -ab 'mkdir /data'				#在所有节点上新建相同的目录
gluster
> volume create vol-k8s replica 3 server1:/home/gfsdata/vol-k8s-bk1 server2:/home/gfsdata/vol-k8s-bk2 server3:/home/gfsdata/vol-k8s-bk3
> volume start vol-k8s
> volume stop vol-k8s
> volume delete vol-k8s			
> volume list
> volume status vol-k8s				#查看volname卷的状态，提示该卷未启动或该卷的砖块有哪些
> q

#挂载卷即可使用
volume卷创建好并启动后，客户端使用这个卷里需要将任一成员的volume卷挂载到自己的指定目录即可，不用管卷里怎么存储文件
mount.gluster server1:vol-k8s  /mnt
for i in `seq -w 1 100`; do cp -rp /var/log/messages /mnt/copy-test-$i; done
```

##### volume卷类型

```bash
#分布式卷
创建卷时不指定卷类型，默认创建分布式卷，文件分散存到各brick中，没有数据冗余
gluster v create vol-ds server1:/ds server2:/ds server3:/ds force

#复制卷
复制卷被挂载后，每向volume里写入一个文件，都会在所有brick保存一个副本，所以当指定副本数为3时需要有3个brick来存储这些副本，副本数为2时需要有2个brick，即便有节点损坏也能保证数据不丢
gluster v create vol-rep replica 3 server1:/rep server2:/rep server3:/rep force
gluster v start vol-rep

#分布式复制卷
#分散卷
#分布式分散卷
```

##### 卷优化与管理

```bash
#卷优化
getfattr -d -m . -e hex /data
gluster volume quota vol-k8s enable													#开启指定volume的配额
gluster volume quota vol-k8s limit-usage / 1TB							#限制指定volume的配额
gluster volume set vol-k8s performance.cache-size 4GB				#设置cache大小,默认32MB
gluster volume set vol-k8s performance.io-thread-count 16		#设置IO线程,太大会导致进程崩溃
gluster volume set vol-k8s network.ping-timeout 10					#设置网络检测时间,默认42s
gluster volume set vol-k8s performance.write-behind-window-size 1024MB	#设置写缓冲区的大小, 默认1M

#扩容缩容
gluster volume add-brick volname2 10.1.20.14:/data4 10.1.20.15:/data5 force
gluster volume remove-brick volname2 force
```

##### K8S挂载GlusterFS

```yml
#创建gluster卷
mkdir /data/vol-k8s-bk1			#在server1上创建
mkdir /data/vol-k8s-bk2			#在server2上创建
mkdir /data/vol-k8s-bk3			#在server3上创建
gluster volume create vol-k8s replica 3 server1:/data/vol-k8s-bk1 server2:/data/vol-k8s-bk2 server3:/data/vol-k8s-bk3
gluster volume start vol-k8s
gluster volume list

#创建svc-glusterfs.yml
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
subsets:
- addresses:
  - ip: 172.16.1.136
  - ip: 172.16.1.137
  - ip: 172.16.1.138
  ports: 
  - port: 24007
    protocol: TCP

#创建PV和PVC
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol-k8s
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  glusterfs:
    endpoints: glusterfs
    path: vol-k8s
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-vol-k8s
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi

#deploy挂载使用PVC
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ss-nginx
spec:
  replicas: 3
  serviceName: webserver 
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
          claimName: pvc-vol-k8s
```

##### 卷命名

```bash
vol-nginx
pv-nginx
pvc-nginx
/data/vol-nginx-bk{1..3}
glusterfs
```

