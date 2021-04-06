## mysql数据字典表

##### 表结构

tables表：提供了关于数据库中的表的信息（包括视图）。详细表述了某个表属于哪个schema，表类型，表引擎，创建时间等

```mysql
select * from information_schema.tables where  table_schema='库名';
db01 [(none)]> desc information_schema.tables;
table_catalog  		#数据表登记目录
table_schema   		#库名
table_name     		#表名
table_type     		#表类型[system view|base table]
engine         		#使用的数据库引擎[MyISAM|CSV|InnoDB]
version        		#版本，默认值10
row_format     		#行格式[compact|dynamic|fixed]
table_rows     		#表里有多少行数据
avg_row_length 		#平均每行占多少字节
data_length    		#数据长度
max_data_length		#最大数据长度
index_length   		#索引占多少字节
data_free      		#碎片占多少字节
auto_increment 		#自增主键当前值是多少
create_time    		#表的创建时间
update_time    		#表的更新时间
check_time     		#表的检查时间
table_collation		#表的字符校验编码集
checksum       		#校验和
create_options 		#创建选项
table_comment 		#表的注释信息
```

##### 表字段

columns表：提供了表中的列信息。详细表述了某张表的所有列以及每个列的信息。

```mysql
select * from information_schema.columns where table_schema='数据库名' and table_name='表名';

db01 [(none)]> desc information_schema.columns;
table_catalog               #数据表登记目录
table_schema                #库名
table_name                  #表名
column_name                 #列名
ordinal_position            #字段在表中第几列
column_default              #列的默认值
is_nullable                 #字段可否为空
data_type                   #数据类型
character_maximum_length    #字符最大长度
character_octet_length      #字节长度
numeric_precision           #数据精度
numeric_scale               #数据规模
datetime_precision          #
character_set_name          #字符集名称
collation_name              #字符集校验名称
column_type                 #列类型
column_key                  #列上索引类型[null|mul|pri]
extra                       #额外描述 [null|on update|current_timestamp|auto_increment]
privileges                  #字段操作权限[select|select,insert,update,references]
column_comment              #字段注释
generation_expression   		#
```

##### 表键值

key_column_usage表:存取表的健值

```mysql
select * from information_schema.key_column_usage where table_schema='库名' and table_name='表名';

db01 [(none)]> desc information_schema.key_column_usage
constraint_catalog           			#数据表登记目录
constraint_schema            			#索引所属表的数据库名
constraint_name              			#索引所属的表名
table_catalog                			#数据表等级目录
table_schema                 			#键值所属表所属的数据库名
table_name                   			#键值所属的表名
column_name                  			#键值所属的列名
ordinal_position             			#键值所属的字段在表中第几列
position_in_unique_constraint			#键值所属的字段在唯一约束的位置（若为外键值为1）
referenced_table_schema      			#外键依赖的数据库名（一般与Constraint_schema值相同）
referenced_table_name        			#外键依赖的表名
referenced_column_name       			#外键依赖的列名
```

##### 表索引

statistics表：提供了关于表索引的信息

```mysql
mysql> select * from information_schema.statistics where table_schema='库名' and table_name='表名';

db01 [(none)]> desc information_schema.statistics;
table_catalog			#数据表登记目录
table_schema 			#索引所属表的数据库名
table_name   			#索引所属的表名
non_unique   			#字段不唯一的标识
index_schema 			#索引所属的数据库名（一般与table_schema值相同）
index_name   			#索引名称
seq_in_index 			#
column_name  			#索引列的列名
collation    			#校对，列值全显示为A
cardinality  			#基数（一般与该表的数据行数相同）
sub_part     			#
packed       			#是否包装过，默认为NULL
nullable     			#是否为空[‘’|YES|NO]
index_type   			#索引的类型，列值全显示为BTREE（平衡树索引）
comment      			#索引注释、备注
index_comment			#
```

##### 表约束

table_constraints表：存储主键约束、外键约束、唯一约束、check约束。各字段的说明信息如下

```mysql
select * from information_schema.table_constraints where table_schema='库名' and table_name='表名'

db01 [(none)]> desc information_schema.table_constraints;
constraint_catalog 			#约束登记目录
constraint_schema  			#约束所属的库库名
constraint_name    			#约束的名称
table_schema       			#约束依赖表所属的数据库名（一般与Constraint_schema值相同）
table_name         			#约束所属的表名
constraint_type    			#约束类型[primary key|foreign key|unique|check]
```

## 应用案例

```mysql
1.查询整个数据库中所有库和所对应的表信息
mysql> select table_schema,group_concat(table_name) 
		-> from tables 
    -> group by table_schema\G

2.统计所有库下的表个数
mysql> select table_schema,count(table_name) as 表数量 
    -> from tables 
    -> group by table_schema 
    -> order by 表数量 desc;

3.查询所有innodb引擎的表及所在的库
mysql> select table_schema,table_name
		-> from tables 
		-> where engine='innodb';

4.统计world数据库下每张表的磁盘空间占用
select table_schema,table_name,concat((table_rows*avg_row_length+index_length)/1024,' KB') as size 
from tables 
where table_schema='world';  

5.统计所有数据库的总的磁盘空间占用
select table_schema as 库名,concat(sum(table_rows*avg_row_length+index_length)/1024," KB")as 总大小 from tables 
group by 库名; 

6.生成整个数据库下的所有表的单独备份语句
select concat("mysqldump ",table_schema," ",table_name,">/tmp/",table_schema,"_",table_name,".sql") 
from information_schema.tables
where table_schema not in('information_schema','performace_schema','sys')
into outfile '/tmp/bak.sh';


7.107张表，都需要执行以下2条语句
														alter table world.city discard tablespace;
														alter table world.city import tablespace;

select concat("alter table ",table_schema,".",table_name," discard tablespace")
from information_schema.tables
where table_schema='world'
into outfile '/tmp/dis.sql';

#备份前需要取消默认对导入导出的限制
[root@db01 ~]# vim /etc/my.cnf
[mysqld]
secure_file_priv=""
```

##### view

```mysql
#创建视图


#查看


#修改



#用视图



```

## show语句

官方参数文档：http://dev.mysql.com/doc/refman/5.7/en/show.html

```mysql
show databases;												#查看所有数据库
show tables;													#查看当前库的所有表
SHOW TABLES FROM											#查看某个指定库下的表
show create database world            #查看建库语句
show create table world.city          #查看建表语句
show grants for  root@'localhost'			#查看用户的权限信息
show charset;													#查看字符集
show collation                        #查看校对规则
show processlist;                     #查看数据库连接情况
show index from                       #表的索引情况
show status                           #数据库状态查看
SHOW STATUS LIKE '%lock%';         		#模糊查询数据库某些状态
SHOW VARIABLES                        #查看所有配置信息
SHOW variables LIKE '%lock%';         #查看部分配置信息
show engines                          #查看支持的所有的存储引擎
show engine innodb status\G           #查看InnoDB引擎相关的状态信息
show binary logs                      #列举所有的二进制日志
show master status                    #查看数据库的日志位置信息
show binlog evnets in                 #查看二进制日志事件
show slave status \G                  #查看从库状态
SHOW RELAYLOG EVENTS              		#查看从库relaylog事件信息
desc (show colums from city)    	    #查看表的列定义信息
```
