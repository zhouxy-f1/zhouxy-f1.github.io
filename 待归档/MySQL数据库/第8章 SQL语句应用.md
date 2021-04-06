##### 如何修改库名？

##### update 和delete强制使用where,如何操作

## 数据定义语言：DDL

```shell
DDL（ definition ）主要是用在定义或改变表的结构，数据类型，表之间的链接和约束等初始化工作上
实现对库和表的创建、删除、修改，如CREATE、ALTER、DROP等。
```

##### 建库

```mysql
mysql> create database if not exists stu charset utf8 collate utf8_general_ci;

#建库规范
1.库名不能有大写字母
2.建库要加字符集，因为二进制安装默认用拉丁字符集，不识别中文
3.库名不能用数字开头
4.库名要和业务相关


#注意
1.库名、表名区分大小写，字符集只作用于表内容
mysql> create database fruit;
mysql> create database FRUIT;

2.建库时指定字符集后，此库下建表默认用建库时的字符集

```

##### 建表

```mysql
mysql> create table stu(
    -> id int(5) zerofill  primary key auto_increment comment '学号',
    -> sname varchar(12) not null comment '姓名',
    -> sage tinyint unsigned comment '年龄',
    -> sgender enum('m','f','n') default 'n' comment '性别',
    -> sid char(18) not null unique comment '身份证',
    -> intime  datetime not null default now() comment '报道时间'
    -> )engine=innodb charset=utf8 comment '学生表';

#拷贝表(含索引)
mysql> create table stu  like student;						//含索引一起拷贝
mysql> create table stu1 select * from student;		//不含索引

#建表规范
1.表名要小写
2.表名不可以数字开头
3.字符集和存储引擎
4.表名要和业务相关
5.选择合适的数据类型
6.每个列都要有注释
7.每个列都设为非空，配合默认值或自增使用，无法保证非空时用0代替

#建表约束（建表时添加）
primary key			主键约束，非空且唯一，一个表只能有一个主键，可用两个列联合作为主键
unique key			唯一键	（可以为空）
not null				非空
unsigned				无符号，针对数字列，配合数值可表示非负，非负时正数的范围增长
zerofill				用零补齐位数，比如00012，设置了zerofill后字段默认转为无符号类型(unsigned)

#其它约束
key							索引
default					默认值
auto_increment	自增长，针对数字列
comment					注释，规范做法，一定要加


#int(5) 与int(11)区别
5和11仅代表插入的数据的显示宽度，在数值小于显示宽度且设置了zerofill时起作用，用零补齐满5位
数值超过5位则int(5)不起作用；
```

##### drop

```shell
mysql> drop database school;	
mysql> drop table tb1,tb2;

生产中禁止使用
```

##### alter

```mysql
#修改库属性
mysql> alter database world charset utf8 collate utf8_general_ci;					//修改库字符集

#修改表结构
mysql> alter table student rename stu;							//改表名
mysql> alter table stu charset utf8;								//改表的字符集
mysql> alter table stu add qq bigint after name;		//添加列在某列后
mysql> alter table stu add num int first    		  	//添加到第1列
mysql> alter table stu drop id;											//删除列（危险），可删主键，会锁表，数据页迁移，影响性能
mysql> alter table stu modify age char(12);					//修改列的数据类型
mysql> alter table stu change intime time date;			//修改列名和数据类型

#注意
1.修改表结构时，会锁表，要避免高峰期执行，也可用pt-osc实现在线DDL
2.修改列及属性(需要带属性且不可只改列名，因为修改列会重新开辟空间，弃用原列的空间)
```

##### 查看库和表信息

```mysql
#库
mysql> show databases;										//查所有库
mysql> select database();									//查当前使用的库
mysql> show create database db_name;			//查看建库语句
mysql> show variables like 'character%';	//查看当前使用的数据库的编码方式，要保证所有编码一致，否则可能乱码

#表
mysql> show tables;			//查看当前使用的库内所有表
mysql> desc tb;					//查看表所有字段及属性
```

## 数据库控制语言：DCL

```shell
DCL（Control）是用来设置或更改数据库用户或角色权限的语句，包括grant、deny、revoke等语句
```

##### grant与revoke

```mysql
db01 [tlbb]> grant select,create,drop,alter on tlbb.* to dev@'10.0.0.%';
db01 [tlbb]> revoke drop,delete on tlbb.* from dev@'10.0.0.%';
```

## 数据操纵语言:DML

```shell
DML（manipulation）主要用来对数据库的数据进行一些操作,就是我们最经常用到的 UPDATE、INSERT、DELETE
```

##### insert

```mysql
mysql> insert into stu(id,sname,sage,sgender) values(1,'zhou1',18,'m');			//标准插入数据
mysql> insert into stu values(2,'zhou2',19,'f');														//插入所有列时可用省略
mysql> insert into stu values(null,'zhou3',20,'m');													//自增字段可用null代替
mysql> insert into stu(sname,sage) values('zhou3','m'),('zhou4','f');				//同时插入多行数据


#注意
1.生产中需要插入数万行数据时，最好每次只插入1000行
```

##### delete

```mysql
mysql> delete from stu where id=14;


#生产中禁用，如何强制？
mysql> delete from tb;
mysql> truncate table tb;

#区别
delete：是DML操作，在逻辑上逐行删除，不释放表空间，所有delete后表内会有空隙，过多操作会让少量数据占很多空间
truncate：DDL操作，对表段中的数据页进行清空，保留表结构,并重新计算索引，速度快

#禁用delete后可使用update做伪删除
mysql> alter table stu add status enum('0','1') default '1';
mysql> update tb set status='0' where id=3;
mysql> select * from tb where status='1';
```

##### update

```shell
mysql> update tb set name='zhou3' where id=3;

#注意
此命令非常危险，一定要加where条件，否则会把该列的所有值全改成一样的了，整表全废
```

## 数据查询语言：DQL

```shell
DQL（query）主要是SELECT操作
```

##### 查看系统参数

官方参考文档：https://dev.mysql.com/doc/refman/5.7/en/func-op-summary-ref.html?tdsourcetag=s_pcqq_aiomsg

```mysql
#单独使用select
db01 [world]> select @@port;
db01 [world]> select @@socket;
db01 [world]> select @@server_id;
db01 [world]> select @@basedir;
db01 [world]> select @@datadir;

#使用函数
db01 [(none)]> select database();
db01 [(none)]> select now();
db01 [(none)]> select user();
db01 [(none)]> select concat('hello world');
db01 [(none)]> select concat(user,'@',host) from mysql.user;
+-------------------------+
| concat(user,'@',host)   |
+-------------------------+
| root@%                  |
| zhouxy@10.0.0.1%        |
| mysql.session@localhost |
| mysql.sys@localhost     |
| root@localhost          |
+-------------------------+
db01 [(none)]> select group_concat(user,'@','host') from mysql.user; 
+-------------------------------------------------------------------+
| group_concat(user,'@','host')                                     |
+-------------------------------------------------------------------+
| root@host,zhouxy@host,mysql.session@host,mysql.sys@host,root@host |
+-------------------------------------------------------------------+
```

##### 单表查询

```mysql
select from where group by having order by [desc] limit n,m;

--select 列1，列2 from 表;
--select 列2，列2 from 表 where 某列=条件;
--select 列1，列2 from 表 where 某列 < <= > >= <> 条件;
--select 列1，列2 from 表 where 某列=条件1 and|or 某列>条件2;
--select 列1，列2 from 表 where 某列 like '字母%';
--select 列1，列2 from 表 where 某列 in ('列值1','列值2');
--select 列1，列2 from 表 where 某列 between 数值1 and 数值2;

group by + 聚合函数
--select 列1,函数(列2) from 表 where 条件1 group by 列1;

having：是在计算出聚合函数之后，对生成的表做条件判断
--select 列1,函数(列2) from 表 where 条件1 group by 列1 having 条件2;
order by + limit
--select 列1,函数(列2) from 表 where 条件1 group by 列1 having 条件2 order by 函数 [desc] limit n,m;

#注意
distinct (列名): 去重
limit n:只选前n列
limit n,m:跳过前n列,共选取m列

```

##### 联合查询

```shell
#一般用union all 改写in|or语句来提高性能

select * from city where countrycode in ('chn','usa')
改写为：
select * from city where countrycode='chn' union all select * from city where countrycode='usa';

union:去重
union all:不去重，性能高于union
```

##### 多表连查

```mysql
select 表1.name,表2.address
from 表1 join 表2
on 表1.id=表2.id
join 表3
on 表2.name=表3.name
where 表1.name='zhang3';
```

## 练习题

##### 单表查询

```shell
1.查询甘肃省所有城市信息
	select * from city where district='gansu';
	
2.查询世界上少于100人的城市
	select countrycode,population from city where population<100; 
	
3.中国人口数量大于500w的城市
	select name,population from city where countrycode='chn' and population>5000000;
	
4.中国或美国城市信息
	select * from city where countrycode in('chn','usa');
	select * from city where countrycode='chn' or countrycode='usa';
	
5.统每个国家的城市数量
	select countrycode,count(id) from city group by countrycode order by count(id) desc; 
	
6.查询中国省名以guang开头的省份
	select distinct(district) from city where district like 'guang%';
	
7.查询世界上人口数量大于1000小于2000的城市信息
	select * from city where population<2000 and population >1000; 
	select * from city where population between 1000 and 2000;
	
8.统计世界上每个国家的总人口数
	select countrycode,sum(population) from city group by countrycode;
	
9.统计中国各个省的总人口数量
	select district,sum(population) from city where countrycode='chn' group by district;
	
10.统计世界上每个国家的城市数量
	select countrycode,sum(population) from city group by countrycode;
	
11.统计中国每个省的总人口数，只打印总人口数小于100万的
	select district,sum(population) from city where countrycode='chn' group by district having sum(population) <1000000;
	
12.查看中国所有的省份，并按人口数进行排序(从大到小)
	select district,sum(population) from city where countrycode='chn' group by district order by sum(population) desc;
	
13.统计中国各个省的总人口数量，按照总人口从大到小排序
	--select district as 省份,sum(population) as 总人口 from city where countrycode='chn' group by district order by 总人口 desc;
	
14.统计中国,每个省的总人口,找出总人口大于500w的,并按总人口从大到小排序,只显示前三名
mysql> select district as 省份,sum(population) as 总人口
    -> from city
    -> where countrycode='chn'
    -> group by 省份
    -> having 总人口>5000000
    -> order by 总人口 desc
    -> limit 3;
```

##### 多表连查1

```shell
#查看world库的所有表结构
select table_name,group_concat(column_name) from information_schema.columns where table_schema='world' and table_name='city';

+------------+-----------------------------------------+
| table_name | group_concat(column_name)               |
+------------+-----------------------------------------+
| city       | ID,Name,CountryCode,District,Population |
+------------+----------------------------------------------------------------------+
| country    | Code,Name,Continent,Region,SurfaceArea,IndepYear,Population,Capital,	|
|						 | LifeExpectancy,GNP,GNPOld,LocalName,GovernmentForm,HeadOfState,Code2	|
+------------+----------------------------------------------------------------------+
| countrylanguage | CountryCode,Language,IsOfficial,Percentage |
+-----------------+--------------------------------------------+

1.查询世界上人口数量小于100人的城市名和国家名
db01 [world]> select country.name , city.name,city.population 
    -> from  city join country
    -> on city.countrycode=country.code
    -> where city.population<100;

2.查询城市shenyang，城市人口，所在国家名（name）及国土面积（SurfaceArea）
db01 [world]> select city.name,country.name,country.surfacearea
    -> from city join country
    -> on city.countrycode=country.code
    -> where city.name='shenyang';
3.查世界上人口数量最少的城市在哪个国家，城市和国家人口数量分别是多少？
db01 [world]> select city.name,city.population,country.name,country.population
    -> from country join city
    -> on country.code=city.countrycode
    -> order by city.population limit 1;

4.世界上人口数量小于100的城市在哪个国家，说的什么语言？
db01 [world]> select country.name,city.name,city.population,countrylanguage.language
    -> from city join country
    -> on city.countrycode=country.code
    -> join countrylanguage
    -> on country.code=countrylanguage.countrycode
    -> where city.population<100;
```

##### 多表连查2（未完成）

```mysql
#查看school的所有表结构
select table_name,group_concat(column_name) from information_schema.columns where table_schema='school' and table_name='course';
+------------+---------------------------+
| table_name | group_concat(column_name) |
+------------+---------------------------+
| course     | cno,cname,tno             |
+------------+---------------------------+
| sc         | sno,cno,score             |
+------------+---------------------------+
| student    | sno,sname,sage,ssex       |
+------------+---------------------------+
| teacher    | tno,tname                 |
+------------+---------------------------+
1.统计zhang3,学习了几门课
2.查询zhang3,学习的课程名称有哪些?
3.查询oldguo老师教的学生名.
4.查询oldguo所教课程的平均分数
5.每位老师所教课程的平均分,并按平均分排序
6.查询oldguo所教的不及格的学生姓名
7.查询所有老师所教学生不及格的信息
8.查询平均成绩大于60分的同学的学号和平均成绩；
9.查询所有同学的学号、姓名、选课数、总成绩；
10.查询各科成绩最高和最低的分：以如下形式显示：课程ID，最高分，最低分 
11.统计各位老师,所教课程的及格率
12.查询每门课程被选修的学生数
13.查询出只选修了一门课程的全部学生的学号和姓名
14.查询选修课程门数超过1门的学生信息
15.统计每门课程:优秀(85分以上),良好(70-85),一般(60-70),不及格(小于60)的学生列表
16.查询平均成绩大于85的所有学生的学号、姓名和平均成绩
```
