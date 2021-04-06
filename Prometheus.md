## 一、Prometheus

##### 监控体系

```bash
#基础设施层
系统层：CPU、load、mem、swap、disk、io、process、keynel parameters等
网络层：网络设备、工作负载、网络延迟、丢包率

#中间件及服务层
消息中间件：Kafka、RocketMQ、RabbitMQ等
Web服务器：Tomcat、Nginx、Jetty
数据库系统：MySQL、PostgreSQL、MogoDB、ES、Redis
数据连接池：ShardingSpere
存储系统：Ceph、GlusterFS

#应用层
衡量应用程序代码的状态和性能

#业务层
1.衡量应用程序的价值，如电子商务网站的销售量
2.QPS、DAU日活、转化率
3.业务接口：登陆数、注册数、订单量、搜索量、支付量等
```

```bash
#相关概念

target:每一个被监控的目标，可以是节点，也可以是mysql、tomcat等
job: 把多个同类的target归到同一个组，即job
instance:在job中的target叫instance

存储格式：时间序列
向量：即时向量和范围向量

metric_name{label="valume",...}
=
!=
=~
!~

ms s m h d w y

#聚合函数（内置11个，仅支持应用于单个即时向量）
sum()
avg()
count()
stddev()		标准差
stdvar()		方差
min()
max()
topk()			返回最大的前k个值
bottomk()
quantile()
count_values()

#运算符优先级
^ * / % == != <= >= < > and unless or

算术运算：	+ - * / % ^
比较运算：	== != < <= > >= 
逻辑运算： and or unless	(逻辑运算仅支持即时向量间进行，不支持对标量运算)

```

##### 服务发现

```yml
#静态配置
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets:
      - 192.168.1.11:9090
      - 192.168.1.12:9100
      - 192.168.1.13:9100
      - 192.168.1.14:9100

#基于文件：指定targets目录由外部文件传入，修改targets后，不需要重启普罗
scrape_configs:
  - job_name: 'prometheus'
    file_sd_configs:
    - refresh_interval: 2m
      files: [targets.yml]				#files是列表类型，单个文件可用[]，多个文件用下面这种写法
      files:
      - targets.yml
---
- targets:
  - localhost:9090
  - 192.168.1.12:9100
  - 192.168.1.13:9100
  - 192.168.1.14:9100
  labels:
    app: node-exporter
    job: prometheus
```























