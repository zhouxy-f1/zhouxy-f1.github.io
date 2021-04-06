## 语法示例

##### meta

```shell
[root@m01 playbook]# cat /root/roles/wordpress/meta/main.yml
dependencies:
  - { role: nginx }
  - { role: php-fpm }
```

## 示例1：用Roles配置nfs

##### 1.创目录

```shell
#创建目录
[root@m01 ~/roles]# mkdir nfs/{tasks,templates,handlers} -pv
[root@m01 ~/roles/nfs]# tree
├── nfs
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   ├── config.yml
│   │   ├── createdata.yml
│   │   ├── install.yml
│   │   ├── main.yml
│   │   ├── mount.yml
│   │   └── start.yml
│   └── templates
│       └── exports.j2
└── site.yml

4 directories, 9 files
```

##### 2.写roles功能

```shell
#拆开写，再组合进main.yml
[root@m01 ~/roles]# cat nfs/tasks/install.yml 
- name: Install nfs-utils Server
  yum: name=nfs-utils state=present
[root@m01 ~/roles]# cat nfs/tasks/config.yml        
- name: Configure NFS Server
  template: src=exports.j2 dest=/etc/exports
  notify: Restart NFS Server
[root@m01 ~/roles]# cat nfs/tasks/start.yml           
- name: Start NFS Server
  systemd: name=nfs-server state=started enabled=yes
[root@m01 ~/roles]# cat nfs/tasks/createdata.yml 
- name: Create Data Directory
  file: path=/data/wordpress state=directory owner=www group=www
[root@m01 ~/roles]# cat nfs/tasks/mount.yml 
- name: Mount web to nfs
  mount:
    path: /code/wordpress/wp-content/uploads/
    src: 172.16.1.31:/data/wordpress
    fstype: nfs
    state: mounted
[root@m01 ~/roles]# cat nfs/tasks/main.yml 
- include_tasks: install.yml 
- include_tasks: config.yml
- include_tasks: start.yml
- include_tasks: createdata.yml
- include_tasks: mount.yml

#准备配置文件
[root@m01 ~/roles]# cat nfs/templates/exports.j2 
{{ nfs_dir }} 172.16.1.0/24(rw,sync,all_squash,anonuid=666,anongid=666)
[root@m01 ~/roles]# cat nfs/handlers/main.yml 
- name: Restart NFS Server
  systemd: name=nfs state=restarted
```

##### 3.playbook中引用

```shell
#最后写playbook
[root@m01 ~/roles]# cat site.yml 
- hosts: web01
  roles:
    - nfs
```

##### 4.变量处理

建议在site.yml同级新建group_vars和host_vars目录，这样的话不仅nfs能用，别的服务也能用这些变量

```shell
[root@m01 ~/roles]# ll
total 4.0K
drwxr-xr-x. 2 root root 17 Sep 26 11:57 group_vars
drwxr-xr-x. 5 root root 52 Sep 26 12:03 nfs
-rw-r--r--. 1 root root 34 Sep 26 11:59 site.yml

[root@m01 ~/roles]# vim group_vars/all 
nfs_dir: /data/wordpress
```

## 示例2：用roles安装memcached

##### 1.新建roles目录

```shell
[root@m01 ~/roles/memcached]# tree
.
├── handlers
│   └── main.yml
├── tasks
│   ├── config.yml
│   ├── install.yml
│   ├── main.yml
│   └── start.yml
└── templates
    └── memcached.j2

3 directories, 6 files
```

##### 2.写tasks功能

```shell
[root@m01 ~/roles/memcached]# vim tasks/install.yml 
- name: Install Memcached Server
  yum: name=memcached state=present
[root@m01 ~/roles/memcached]# vim tasks/config.yml 
- name: Scp Memcache configure
  template: src=memcached.j2 dest=/etc/sysconfig/memcached
  notify: Restart Memcached Server
[root@m01 ~/roles/memcached]# vim tasks/start.yml 
- name: Start Memcached Server
  systemd: name=memcached state=started
[root@m01 ~/roles/memcached]# vim tasks/main.yml 
- include_tasks: install.yml
- include_tasks: config.yml
- include_tasks: start.yml

[root@m01 ~/roles/memcached]# vim handlers/main.yml 
- name: Restart Memcached Server
  systemd: name=memcached state=restarted
[root@m01 ~/roles/memcached]# vim templates/memcached.j2 
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="{{ ansible_memtotal_mb //2 }}"
OPTIONS=""
```

##### 3.playbook中引用

```shell
[root@m01 ~/roles]# vim mem.yml 
- hosts: web01
  roles:
    - memcached
```

## 示例3：用roles安装rsync

##### 1.新建Roles目录

```shell 
[root@m01 ~/roles]# mkdir rsync/{tasks,templates,handlers} -p  
[root@m01 ~/roles]# tree rsync/
rsync/
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    ├── rsyncd.conf.j2
    └── rsync.passwd.j2
3 directories, 4 files
```

##### 2.写tasks功能

```shell
#tasks
[root@m01 ~/roles/rsync]# cat tasks/main.yml 
- name: Install Rsync Server
  yum: name=rsync state=present

- name: Configure Rsync Server
  template: src={{item.src}} dest={{item.dest}}  mode={{item.mode}}
  with_items:
    - { src: 'rsyncd.conf.j2',dest: '/etc/sysconfig/rsyncd',mode: '0644'}
    - { src: 'rsync.passwd.j2',dest: '/etc/rsync.passwd',mode: '0600'}
  notify: Restart Rsync Server

- name: Start Rsync Server
  systemd: name=rsyncd state=started

#handlers
[root@m01 ~/roles/rsync]# cat handlers/main.yml 
- name: Restart Rsync Server
  systemd: name=rsyncd state=restarted

#templates
[root@m01 ~/roles/rsync]# cat templates/rsyncd.conf.j2 
uid = www
gid = www
port = 873
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore errors
read only = false
list = false
auth users = rsync_backup
secrets file = /etc/rsync.passwd
log file = /var/log/rsyncd.log
[nfs]
comment = backup the directory named share in nfs-server !
path = /backup_share
[everyday]
comment = backup the important data in all hostcomputer everyday!
path = /backup_everyday
[root@m01 ~/roles/rsync]# cat templates/rsync.passwd.j2 
rsync_backup:123
```

##### 3.playbook中引用

```shell
[root@m01 ~/roles]# cat site.yml 
- hosts: web02
  roles:
    - nfs
  tags: nfs-server

- hosts: web01
  roles:
    - memcached
    - rsync
  tags:
    - memcached-server
    - rsync-server
[root@m01 ~/roles]# ansible-playbook site.yml  -t rsync-server
```

## ansible-galaxy

##### 相关命令

```shell
[root@m01 ~]# ansible-galaxy init nginx
[root@m01 ~]# ansible-galaxy search nginx
[root@m01 ~]# ansible-galaxy install geerlingguy.nginx
```
