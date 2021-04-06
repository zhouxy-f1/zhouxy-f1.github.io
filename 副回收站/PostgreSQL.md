## 环境说明

```bash
k*W$FP2&YAQh
2}<sR&Uy6g^N

#部署到新环境需要把下面的ip替换为新的ip
primary服务器:	192.168.1.13
standby服务器:	192.168.1.14
服务器用户postgres密码：p5s\RxVf;e+t
数据库用户postgres密码：D8D[*{33>mE<
replication密码:	ieu9387LIIUWFJ763&!
postgres密码:	test
pcp密码：test
虚拟IP： 192.168.1.99
```

## 部署PostgreSQL11

##### 安装

```bash
部署postgresql的所有操作都需要在primary和standby都执行

#Linux7
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
yum install -y postgresql11-server postgresql11 postgresql11-contrib

#Linux8
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql11-server postgresql11 postgresql11-contrib
```

##### 配置

```bash
#配置用户和目录
useradd postgres
echo 'p5s\RxVf;e+t'|passwd --stdin postgres
mkdir -p /home/pgdata/pgsql/{data,log,archivedir}
chown -R postgres.postgres /home/pgdata/pgsql /usr/pgsql-11/
cat >>/etc/hosts << EOF
192.168.1.13  pg-test-1
192.168.1.14  pg-test-2
EOF

#数据库初始化
先修改初始化脚本和启动的配置文件里的默认目录，最后初始化数据库
sed -i '/^PREVDATADIR/cPREVDATADIR=/home/pgdata/pgsql' /usr/pgsql-11/bin/postgresql-11-setup
sed -i 's#/var/lib/pgsql/11#/home/pgdata/pgsql#g' /usr/lib/systemd/system/postgresql-11.service
systemctl daemon-reload
/usr/pgsql-11/bin/postgresql-11-setup initdb

#配置数据库访问控制
cat>/home/pgdata/pgsql/data/pg_hba.conf<<'EOF'
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     peer
host    all             all             127.0.0.1/32            ident
host    all             all             ::1/128                 ident
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            ident
host    replication     all             ::1/128                 ident
host    replication     replication     192.168.1.13/32         md5
host    replication     replication     192.168.1.14/32         md5
host    all             postgres        192.168.1.13/32         trust
host    all             postgres        192.168.1.14/32         trust
host    all             repl            192.168.1.13/32         trust
host    all             repl            192.168.1.14/32         trust
EOF

#修改配置文件
cat>/home/pgdata/pgsql/data/postgresql.conf<<'EOF'
listen_addresses = '*'
port = 5433
max_connections = 500
shared_buffers = 128MB
max_prepared_transactions = 900
dynamic_shared_memory_type = posix
wal_level = hot_standby
max_wal_size = 1GB
min_wal_size = 80MB
archive_mode = on
archive_command = 'cp "%p" "/home/pgdata/pgsql/archivedir/%f"'
max_wal_senders = 3
wal_keep_segments = 32
max_replication_slots = 3
hot_standby = on
hot_standby_feedback = on
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%a.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_line_prefix = '%m [%p] '
log_timezone = 'Asia/Shanghai'
datestyle = 'iso, mdy'
timezone = 'Asia/Shanghai'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'
pgpool.pg_ctl = '/usr/pgsql-11/bin/pg_ctl'
EOF
```

##### 启动

```bash
systemctl restart postgresql-11
systemctl enable postgresql-11
systemctl status postgresql-11
su - postgres -c 'psql -p5433'
```

## 部署pgpool-II

##### 安装

```bash
#Linux7
yum install -y http://www.pgpool.net/yum/rpms/4.0/redhat/rhel-7-x86_64/pgpool-II-release-4.0-1.noarch.rpm
yum install -y pgpool-II-pg11* postgresql-devel

#Linux8
yum install -y http://www.pgpool.net/yum/rpms/4.0/redhat/rhel-8-x86_64/pgpool-II-release-4.0-1.noarch.rpm
yum install -y pgpool-II-pg11* postgresql-devel
```

##### 配置pgpool.conf

```bash
#Primary上的配置
vim /etc/pgpool-II/pgpool.conf

listen_addresses = '*'
port = 5432
socket_dir = '/var/run/postgresql'
listen_backlog_multiplier = 2
serialize_accept = off
pcp_listen_addresses = '*'
pcp_port = 9898
pcp_socket_dir = '/var/run/postgresql'
backend_hostname0 = 'pg-test-1'
backend_port0 = 5433
backend_weight0 = 1
backend_data_directory0 = '/home/pgdata/pgsql/data'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = 'pg-test-2'
backend_port1 = 5433
backend_weight1 = 1
backend_data_directory1 = '/home/pgdata/pgsql/data'
backend_flag1 = 'ALLOW_TO_FAILOVER'
enable_pool_hba = on																	#开启客户端认证
pool_passwd = 'pool_passwd'														#指定用于md5认证的文件名
authentication_timeout = 60
allow_clear_text_frontend_auth = off
ssl = off
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = off
num_init_children = 400
max_pool = 10
child_life_time = 300
child_max_connections = 0
connection_life_time = 0
client_idle_limit = 0
log_destination = 'syslog'
log_line_prefix = '%t: pid %p: '
log_connections = off
log_hostname = off
log_statement = off
log_per_node_statement = off
log_client_messages = off
log_standby_delay = 'none'
syslog_facility = 'LOCAL1'
syslog_ident = 'pgpool'
pid_file_name = '/var/run/pgpool/pgpool.pid'
logdir = '/var/log/pgpool'
connection_cache = on
reset_query_list = 'ABORT; DISCARD ALL'
replication_mode = off
replicate_select = off
insert_lock = on
lobj_lock_table = ''
replication_stop_on_mismatch = off
failover_if_affected_tuples_mismatch = off
load_balance_mode = on
ignore_leading_white_space = on
white_function_list = ''
black_function_list = 'currval,lastval,nextval,setval'
black_query_pattern_list = ''
database_redirect_preference_list = ''
app_name_redirect_preference_list = ''
allow_sql_comments = off
disable_load_balance_on_write = 'transaction'
master_slave_mode = on															#开启主备模式
master_slave_sub_mode = 'stream'										#开启基于流复制
sr_check_period = 5
sr_check_user = 'postgres'													#执行流复制的用户
sr_check_password = 'test'
sr_check_database = 'postgres'											#执行流复制检测的数据库
delay_threshold = 0
follow_master_command = ''
health_check_period = 5
health_check_timeout = 30
health_check_user = 'postgres'
health_check_password = 'test'
health_check_database = ''
health_check_max_retries = 3
health_check_retry_delay = 1
connect_timeout = 10000
failover_command = '/etc/pgpool-II/failover.sh %d %h %p %D %m %H %M %P %r %R'
failback_command = ''
failover_on_backend_error = on
detach_false_primary = off
search_primary_node_timeout = 300
recovery_user = 'postgres'
recovery_password = 'test'
recovery_1st_stage_command = 'recovery_1st_stage.sh'
recovery_2nd_stage_command = ''
recovery_timeout = 90
client_idle_limit_in_recovery = 0
use_watchdog = on														#激活看门狗
trusted_servers = 'pg-test-1,pg-test-2'
ping_path = '/bin'
wd_hostname = 'pg-test-1'										#本机看门狗的地址，primary服务器上配置为pg-test-1，standby上配pg-test-2
wd_port = 9000
wd_priority = 2															#配置本地看门狗在选举时优先级，值越大越优先
wd_authkey = ''
wd_ipc_socket_dir = '/var/run/postgresql'
delegate_IP = '192.168.1.50'								#这里配置VIP，供外部访问
if_cmd_path = '/bin'
if_up_cmd = 'ip_w addr add $_IP_$/24 dev eth0 label eth0:0'
if_down_cmd = 'ip_w addr del $_IP_$/24 dev eth0'
arping_path = '/usr/bin'
arping_cmd = 'arping_w -U $_IP_$ -w 1'
clear_memqcache_on_escalation = on
wd_escalation_command = ''
wd_de_escalation_command = ''
failover_when_quorum_exists = on
failover_require_consensus = on
allow_multiple_failover_requests_from_node = off
wd_monitoring_interfaces_list = ''
wd_lifecheck_method = 'heartbeat'
wd_interval = 3
wd_heartbeat_port = 9694
wd_heartbeat_keepalive = 2
wd_heartbeat_deadtime = 30
heartbeat_destination0 = 'pg-test-2'		#检测对方心跳
heartbeat_destination_port0 = 9694 
heartbeat_device0 = ''
wd_life_point = 3
wd_lifecheck_query = 'SELECT 1'
wd_lifecheck_dbname = 'template1'
wd_lifecheck_user = 'nobody'
wd_lifecheck_password = ''
other_pgpool_hostname0 = 'pg-test-2'		#配置对方的pgpool
other_pgpool_port0 = 5432
other_wd_port0 = 9000
relcache_expire = 0
relcache_size = 256
check_temp_table = on
check_unlogged_table = on
memory_cache_enabled = off
memqcache_method = 'shmem'
memqcache_memcached_host = 'localhost'
memqcache_memcached_port = 11211
memqcache_total_size = 67108864
memqcache_max_num_cache = 1000000
memqcache_expire = 0
memqcache_auto_cache_invalidation = on
memqcache_maxcache = 409600
memqcache_cache_block_size = 1048576
memqcache_oiddir = '/var/log/pgpool/oiddir'
white_memqcache_table_list = ''
black_memqcache_table_list = ''
log_timezone = 'Asia/Shanghai'
timezone = 'Asia/Shanghai'
lc_messages = 'zh_CN.UTF-8'
lc_monetary = 'zh_CN.UTF-8'
lc_numeric = 'zh_CN.UTF-8'
lc_time = 'zh_CN.UTF-8'

#将上面的配置复制到standby服务器上，再执行如下命令
sed -i '/wd_hostname/s#pg-test-1#pg-test-2#g' /etc/pgpool-II/pgpool.conf
sed -i '/heartbeat_destination0/s#pg-test-2#pg-test-1#g' /etc/pgpool-II/pgpool.conf
sed -i '/other_pgpool_hostname0/s#pg-test-2#pg-test-1#g' /etc/pgpool-II/pgpool.conf
sed -i '/wd_priority/s#2#1#g' /etc/pgpool-II/pgpool.conf
```

##### 配置pcp.conf

```bash
pcp.conf是pgpool管理器自己的用户名和密码，用于管理集群

#先生成配置
pg_md5 test			=> 得到 098f6bcd4621d373cade4e832627b4f6
echo 'postgres:098f6bcd4621d373cade4e832627b4f6'>>/etc/pgpool-II/pcp.conf

#再根据pcp.conf配置生成md5认证的文件pool_passwd，文件内容是用户名和密码
pg_md5 -f /etc/pgpool-II/pgpool.conf -m -u postgres test
```

##### 修改pgpool访问控制

```bash
cat>>/etc/pgpool-II/pool_hba.conf<<EOF
host    all         all         0.0.0.0/0             md5
host    all         all         0/0                   md5
EOF
```

##### 集群免密

```bash
#设置集群的服务器之间免密码访问，需要2台机器都执行，执行时切换到postgres用户

su - postgres
ssh-keygen -t rsa -f .ssh/id_rsa_pgpool
ssh-copy-id -i .ssh/id_rsa_pgpool.pub postgres@pg-test-1
ssh-copy-id -i .ssh/id_rsa_pgpool.pub postgres@pg-test-2
```

## 根据配置文件准备环境

##### 创建恢复方法

```bash
#安装sql脚本
cd  /var/lib/pgsql/
curl -O https://www.pgpool.net/mediawiki/images/pgpool-II-4.0.6.tar.gz
tar -xzvf pgpool-II-4.0.6.tar.gz
curl -O https://www.pgpool.net/mediawiki/images/pgpool-II-3.5.2.tar.gz
tar -xzvf pgpool-II-3.5.2.tar.gz
mkdir /home/pgdata/pgsql/data/sql
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/insert_lock.sql /home/pgdata/pgsql/data/sql/
cp /var/lib/pgsql/pgpool-II-3.5.2/src/sql/pgpool_adm/pgpool_adm.sql.in /home/pgdata/pgsql/data/sql/pgpool_adm.sql
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-recovery/pgpool-recovery.sql.in /home/pgdata/pgsql/data/sql/pgpool-recovery.sql
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-recovery/uninstall_pgpool-recovery.sql /home/pgdata/pgsql/data/sql/
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-regclass/pgpool-regclass.sql.in /home/pgdata/pgsql/data/sql/pgpool-regclass.sql
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-regclass/uninstall_pgpool-regclass.sql /home/pgdata/pgsql/data/sql/
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool_adm/pgpool_adm.control /etc/pgpool-II/
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool_adm/pgpool_adm--1.0.sql /etc/pgpool-II/
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-recovery/pgpool_recovery.control /etc/pgpool-II/
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-recovery/pgpool_recovery--1.1.sql /etc/pgpool-II/
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-regclass/pgpool_regclass.control /etc/pgpool-II/
cp /var/lib/pgsql/pgpool-II-4.0.6/src/sql/pgpool-regclass/pgpool_regclass--1.0.sql /etc/pgpool-II/
rm -fr /var/lib/pgsql/pgpool-II-4.0.6
rm -f /var/lib/pgsql/pgpool-II-4.0.6.tar.gz
rm -fr /var/lib/pgsql/pgpool-II-3.5.2
rm -f /var/lib/pgsql/pgpool-II-3.5.2.tar.gz
sed -i "s/MODULE_PATHNAME/\/usr\/pgsql-11\/lib\/pgpool_adm/g" /etc/pgpool-II/pgpool_adm--1.0.sql
sed -i "s/MODULE_PATHNAME/\/usr\/pgsql-11\/lib\/pgpool_adm/g" /home/pgdata/pgsql/data/sql/pgpool_adm.sql
sed -i "s/MODULE_PATHNAME/\/usr\/pgsql-11\/lib\/pgpool-recovery/g" /etc/pgpool-II/pgpool_recovery--1.1.sql
sed -i "s/MODULE_PATHNAME/\/usr\/pgsql-11\/lib\/pgpool-recovery/g" /home/pgdata/pgsql/data/sql/pgpool-recovery.sql
chown postgres:postgres -R /home/pgdata/pgsql/data/sql

#根据上面的sql脚本来创建Recover和Admin方法
cd /home/pgdata/pgsql/data/sql
sudo -u postgres psql -p 5433 -f pgpool-recovery.sql template1
vi /home/pgdata/pgsql/data/sql/pgpool_adm.sql
修改第30行，方法的返回类型为Integer,如下所示
CREATE FUNCTION pcp_node_count(text, integer, text, text, OUT node_count integer)
RETURNS integer
CREATE FUNCTION pcp_node_count(text, OUT node_count integer)
RETURNS integer
sudo -u postgres psql -p 5433  -f pgpool_adm.sql template1
```

##### 创建恢复配置文件

```bash
#Primary上执行
cat>/home/pgdata/pgsql/data/recovery.done<<EOF
standby_mode = 'on'
primary_conninfo = 'user=replication password=''ieu9387LIIUWFJ763&!'' host=192.168.1.14 port=5433 sslmode=prefer sslcompression=0 krbsrvname=postgres target_session_attrs=any'
EOF

#Standby上执行的
cat>/home/pgdata/pgsql/data/recovery.done<<EOF
standby_mode = 'on'
primary_conninfo = 'user=replication password=''ieu9387LIIUWFJ763&!'' host=192.168.1.13 port=5433 sslmode=prefer sslcompression=0 krbsrvname=postgres target_session_attrs=any'
EOF
```

##### 配置恢复用户

```bash
#库里新建用户
su - postgres
psql -p5433
CREATE ROLE replication WITH REPLICATION PASSWORD ' test' LOGIN;
CREATE ROLE repl WITH REPLICATION PASSWORD 'test' LOGIN;
ALTER USER postgres WITH PASSWORD 'test';
\q
logout
```

##### 配置日志输出

```bash
mkdir /var/log/pgpool-II
touch /var/log/pgpool-II/pgpool.log

sed -i 's#cron.none#&;LOCAL1.none#g' /etc/rsyslog.conf 
sed -i '$a\\n#By:zhouxy		At:2021-1-18 \nLOCAL1.*    		/var/log/pgpool-II/pgpool.log' /etc/rsyslog.conf 
sed -i '1i/var/log/pgpool-II/pgpool.log' /etc/logrotate.d/syslog
systemctl restart rsyslog
```

##### recovery_1st_stage.sh

```bash
vi /home/pgdata/pgsql/data/recovery_1st_stage.sh
chmod +x /home/pgdata/pgsql/data/recovery_1st_stage.sh
chown -R postgres.postgres /home/pgdata/pgsql/data/recovery_1st_stage.sh

#!/bin/bash
# This script is executed by "recovery_1st_stage" to recovery a Standby node.
set -o xtrace
exec > >(logger -i -p local1.info) 2>&1
PRIMARY_NODE_PGDATA="$1"
DEST_NODE_HOST="$2"
DEST_NODE_PGDATA="$3"
PRIMARY_NODE_PORT="$4"
DEST_NODE_PORT=5432
PRIMARY_NODE_HOST=$(hostname)
PGHOME=/usr/pgsql-11
ARCHIVEDIR=/home/pgdata/pgsql/archivedir
REPL_USER=repl
logger -i -p local1.info recovery_1st_stage: start: pg_basebackup for Standby node PostgreSQL@{$DEST_NODE_HOST}
## Test passwrodless SSH
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${DEST_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ls /tmp > /dev/null
if [ $? -ne 0 ]; then
    logger -i -p local1.error recovery_1st_stage: passwrodless SSH to postgres@${DEST_NODE_HOST} failed. Please setup passwrodless SSH.
    exit 1
fi
## Get PostgreSQL major version
PGVERSION=`${PGHOME}/bin/initdb -V | awk '{print $3}' | sed 's/\..*//' | sed 's/\([0-9]*\)[a-zA-Z].*/\1/'`
if [ $PGVERSION -ge 12 ]; then
    RECOVERYCONF=${DEST_NODE_PGDATA}/myrecovery.conf
else
    RECOVERYCONF=${DEST_NODE_PGDATA}/recovery.conf
fi
## Execute pg_basebackup to recovery Standby node
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@$DEST_NODE_HOST -i ~/.ssh/id_rsa_pgpool "
    set -o errexit
    rm -rf $DEST_NODE_PGDATA
    rm -rf $ARCHIVEDIR/*
    ${PGHOME}/bin/pg_basebackup -h ${PRIMARY_NODE_HOST} -U ${REPL_USER} -p ${PRIMARY_NODE_PORT} -D ${DEST_NODE_PGDATA} -X stream
    if [ ${PGVERSION} -ge 12 ]; then
        sed -i -e \"\\\$ainclude_if_exists = '$(echo ${RECOVERYCONF} | sed -e 's/\//\\\//g')'\" \
               -e \"/^include_if_exists = '$(echo ${RECOVERYCONF} | sed -e 's/\//\\\//g')'/d\" ${DEST_NODE_PGDATA}/postgresql.conf
    fi
    cat > ${RECOVERYCONF} << EOT
primary_conninfo = 'host=${PRIMARY_NODE_HOST} port=${PRIMARY_NODE_PORT} user=${REPL_USER} passfile=''/var/lib/pgsql/.pgpass'''
recovery_target_timeline = 'latest'
restore_command = 'scp ${PRIMARY_NODE_HOST}:${ARCHIVEDIR}/%f %p'
EOT
    if [ ${PGVERSION} -ge 12 ]; then
        touch ${DEST_NODE_PGDATA}/standby.signal
    else
        echo \"standby_mode = 'on'\" >> ${RECOVERYCONF}
    fi
    sed -i \"s/#*port = .*/port = ${DEST_NODE_PORT}/\" ${DEST_NODE_PGDATA}/postgresql.conf"
if [ $? -ne 0 ]; then
    logger -i -p local1.error recovery_1st_stage: end: pg_basebackup failed. online recovery failed
    exit 1
fi
logger -i -p local1.info recovery_1st_stage: end: recovery_1st_stage complete
exit 0
```

##### failover.sh

```bash
vi /etc/pgpool-II/failover.sh
chown postgres:postgres /etc/pgpool-II/failover.sh
chmod 0700 /etc/pgpool-II/failover.sh

#!/bin/bash

set -o xtrace
exec > >(logger -i -p local1.info) 2>&1
FAILED_NODE_ID="$1"
FAILED_NODE_HOST="$2"
FAILED_NODE_PORT="$3"
FAILED_NODE_PGDATA="$4"
NEW_MASTER_NODE_ID="$5"
NEW_MASTER_NODE_HOST="$6"
OLD_MASTER_NODE_ID="$7"
OLD_PRIMARY_NODE_ID="$8"
NEW_MASTER_NODE_PORT="$9"
NEW_MASTER_NODE_PGDATA="${10}"
PGHOME=/usr/pgsql-11
logger -i -p local1.info failover.sh: start: failed_node_id=${FAILED_NODE_ID} old_primary_node_id=${OLD_PRIMARY_NODE_ID} \
    failed_host=${FAILED_NODE_HOST} new_master_host=${NEW_MASTER_NODE_HOST}
## Test passwrodless SSH
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null postgres@${NEW_MASTER_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ls /tmp > /dev/null
if [ $? -ne 0 ]; then
    logger -i -p local1.error failover.sh: passwrodless SSH to postgres@${NEW_MASTER_NODE_HOST} failed. Please setup passwrodless SSH.
    exit 1
fi
# If standby node is down, skip failover.
if [ ${FAILED_NODE_ID} -ne ${OLD_PRIMARY_NODE_ID} ]; then
    logger -i -p local1.info failover.sh: Standby node is down. Skipping failover.
    exit 0
fi
# Promote standby node.
logger -i -p local1.info failover.sh: Primary node is down, promote standby node PostgreSQL@${NEW_MASTER_NODE_HOST}.
ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
postgres@${NEW_MASTER_NODE_HOST} -i ~/.ssh/id_rsa_pgpool ${PGHOME}/bin/pg_ctl -D ${NEW_MASTER_NODE_PGDATA} -w promote
if [ $? -ne 0 ]; then
    logger -i -p local1.error failover.sh: new_master_host=${NEW_MASTER_NODE_HOST} promote failed
    exit 1
fi
logger -i -p local1.info failover.sh: end: new_master_node_id=$NEW_MASTER_NODE_ID started as the primary node
exit 0
```

##### 新建两个命令脚本

```bash
#ip_w脚本

cat>/bin/ip_w<<'EOF'
#!/bin/bash
if [ $UID -eq 0 ]
then
        /sbin/ip $@
else
        sudo /sbin/ip $@
fi
exit 0
EOF

cat>/usr/bin/arping_w<<'EOF'
#!/bin/bash
if [ $UID -eq 0 ]
then
        /usr/sbin/arping $@
else
        sudo /usr/sbin/arping $@
fi
exit 0
EOF

chmod +x /usr/bin/arping_w
chmod +x /bin/ip_w
ll /usr/bin/arping_w   /bin/ip_w 


#配置sudo权限
sed -i '$a\\n#By:zhouxy   At:2021-1-29\npostgres ALL=(root) NOPASSWD: /sbin/ip\npostgres ALL=(root) NOPASSWD: /usr/sbin/arping' /etc/sudoers
```

##### 启动pgpool

```bash
#两台机器都启动服务
systemctl start pgpool
systemctl enable pgpool
systemctl status pgpool
```

## 查看集群状态

```bash
su - postgres
psql -h 192.168.50.190 -p 5432
show pool_nodes;
```

## 恢复集群

```bash
首先关闭Pgpool-II，再关闭Postgresql
首先开启Postgresl，再开启Pgpool-II

su - postgres
psql -h 192.168.50.190 -p 5432
show pool_nodes;
返回结果中有status为down的记录是需要恢复的服务器
如down状态的服务器为192.168.50.121，执行恢复操作
systemctl stop postgresql-11
cd /home/pgdata/pgsql/
mv data data-backupxxxx
pg_basebackup -v -D data -R -P -h 192.168.50.125 -p 5433 -U replication
chown -R postgres.postgres data
systemctl start postgresql-11
在up状态服务器192.168.50.125上执行添加节点操作，最后的参数为node_id
pcp_attach_node -h 192.168.50.121 -U pgpool -p 9898  -n 0
```

