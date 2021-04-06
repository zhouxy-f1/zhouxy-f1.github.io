```mysql


[client]    
port = 3306
socket = /tmp/mysql.sock
default-character-set=utf8
[mysqld]
default-character-set=utf8
user = mysql    --- 表示MySQL的管理用户
port = 3306    --- 端口
socket = /tmp/mysql.sock    -- 启动的sock文件
log-bin = /data/mysql-bin
basedir = /usr/local/mysql
datadir = /data/
pid-file = /data/mysql.pid
user = mysql
bind-address = 0.0.0.0
server-id = 1 #表示是本机的序号为1,一般来讲就是master的意思

skip-name-resolve
# 禁止MySQL对外部连接进行DNS解析，使用这一选项可以消除MySQL进行DNS解析的时间。但需要注意，如果开启该选项，
# 则所有远程主机连接授权都要使用IP地址方式，否则MySQL将无法正常处理连接请求

#skip-networking

back_log = 600
# MySQL能有的连接数量。当主要MySQL线程在一个很短时间内得到非常多的连接请求，这就起作用，
# 然后主线程花些时间(尽管很短)检查连接并且启动一个新线程。back_log值指出在MySQL暂时停止回答新请求之前的短时间内多少个请求可以被存在堆栈中。
# 如果期望在一个短时间内有很多连接，你需要增加它。也就是说，如果MySQL的连接数据达到max_connections时，新来的请求将会被存在堆栈中，
# 以等待某一连接释放资源，该堆栈的数量即back_log，如果等待连接的数量超过back_log，将不被授予连接资源。
# 另外，这值（back_log）限于您的操作系统对到来的TCP/IP连接的侦听队列的大小。
# 你的操作系统在这个队列大小上有它自己的限制（可以检查你的OS文档找出这个变量的最大值），试图设定back_log高于你的操作系统的限制将是无效的。

max_connections = 1000
# MySQL的最大连接数，如果服务器的并发连接请求量比较大，建议调高此值，以增加并行连接数量，当然这建立在机器能支撑的情况下，因为如果连接数越多，介于MySQL会为每个连接提供连接缓冲区，就会开销越多的内存，所以要适当调整该值，不能盲目提高设值。可以过'conn%'通配符查看当前状态的连接数量，以定夺该值的大小。

max_connect_errors = 6000
# 对于同一主机，如果有超出该参数值个数的中断错误连接，则该主机将被禁止连接。如需对该主机进行解禁，执行：FLUSH HOST。

open_files_limit = 65535
# MySQL打开的文件描述符限制，默认最小1024;当open_files_limit没有被配置的时候，比较max_connections*5和ulimit -n的值，哪个大用哪个，
# 当open_file_limit被配置的时候，比较open_files_limit和max_connections*5的值，哪个大用哪个。

table_open_cache = 128
# MySQL每打开一个表，都会读入一些数据到table_open_cache缓存中，当MySQL在这个缓存中找不到相应信息时，才会去磁盘上读取。默认值64
# 假定系统有200个并发连接，则需将此参数设置为200*N(N为每个连接所需的文件描述符数目)；
# 当把table_open_cache设置为很大时，如果系统处理不了那么多文件描述符，那么就会出现客户端失效，连接不上

max_allowed_packet = 4M
# 接受的数据包大小；增加该变量的值十分安全，这是因为仅当需要时才会分配额外内存。例如，仅当你发出长查询或MySQLd必须返回大的结果行时MySQLd才会分配更多内存。
# 该变量之所以取较小默认值是一种预防措施，以捕获客户端和服务器之间的错误信息包，并确保不会因偶然使用大的信息包而导致内存溢出。

binlog_cache_size = 1M
# 一个事务，在没有提交的时候，产生的日志，记录到Cache中；等到事务提交需要提交的时候，则把日志持久化到磁盘。默认binlog_cache_size大小32K

max_heap_table_size = 8M
# 定义了用户可以创建的内存表(memory table)的大小。这个值用来计算内存表的最大行数值。这个变量支持动态改变

tmp_table_size = 16M
# MySQL的heap（堆积）表缓冲大小。所有联合在一个DML指令内完成，并且大多数联合甚至可以不用临时表即可以完成。
# 大多数临时表是基于内存的(HEAP)表。具有大的记录长度的临时表 (所有列的长度的和)或包含BLOB列的表存储在硬盘上。
# 如果某个内部heap（堆积）表大小超过tmp_table_size，MySQL可以根据需要自动将内存中的heap表改为基于硬盘的MyISAM表。还可以通过设置tmp_table_size选项来增加临时表的大小。也就是说，如果调高该值，MySQL同时将增加heap表的大小，可达到提高联接查询速度的效果

read_buffer_size = 2M
# MySQL读入缓冲区大小。对表进行顺序扫描的请求将分配一个读入缓冲区，MySQL会为它分配一段内存缓冲区。read_buffer_size变量控制这一缓冲区的大小。
# 如果对表的顺序扫描请求非常频繁，并且你认为频繁扫描进行得太慢，可以通过增加该变量值以及内存缓冲区大小提高其性能

read_rnd_buffer_size = 8M
# MySQL的随机读缓冲区大小。当按任意顺序读取行时(例如，按照排序顺序)，将分配一个随机读缓存区。进行排序查询时，
# MySQL会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速度，如果需要排序大量数据，可适当调高该值。但MySQL会为每个客户连接发放该缓冲空间，所以应尽量适当设置该值，以避免内存开销过大

sort_buffer_size = 8M
# MySQL执行排序使用的缓冲大小。如果想要增加ORDER BY的速度，首先看是否可以让MySQL使用索引而不是额外的排序阶段。
# 如果不能，可以尝试增加sort_buffer_size变量的大小

join_buffer_size = 8M
# 联合查询操作所能使用的缓冲区大小，和sort_buffer_size一样，该参数对应的分配内存也是每连接独享

thread_cache_size = 8
# 这个值（默认8）表示可以重新利用保存在缓存中线程的数量，当断开连接时如果缓存中还有空间，那么客户端的线程将被放到缓存中，
# 如果线程重新被请求，那么请求将从缓存中读取,如果缓存中是空的或者是新的请求，那么这个线程将被重新创建,如果有很多新的线程，
# 增加这个值可以改善系统性能.通过比较Connections和Threads_created状态的变量，可以看到这个变量的作用。(–>表示要调整的值)
# 根据物理内存设置规则如下：
# 1G  —> 8
# 2G  —> 16
# 3G  —> 32
# 大于3G  —> 64

query_cache_size = 8M
#MySQL的查询缓冲大小（从4.0.1开始，MySQL提供了查询缓冲机制）使用查询缓冲，MySQL将SELECT语句和查询结果存放在缓冲区中，
# 今后对于同样的SELECT语句（区分大小写），将直接从缓冲区中读取结果。根据MySQL用户手册，使用查询缓冲最多可以达到238%的效率。
# 通过检查状态值'Qcache_%'，可以知道query_cache_size设置是否合理：如果Qcache_lowmem_prunes的值非常大，则表明经常出现缓冲不够的情况，
# 如果Qcache_hits的值也非常大，则表明查询缓冲使用非常频繁，此时需要增加缓冲大小；如果Qcache_hits的值不大，则表明你的查询重复率很低，
# 这种情况下使用查询缓冲反而会影响效率，那么可以考虑不用查询缓冲。此外，在SELECT语句中加入SQL_NO_CACHE可以明确表示不使用查询缓冲

query_cache_limit = 2M
#指定单个查询能够使用的缓冲区大小，默认1M

key_buffer_size = 4M
#指定用于索引的缓冲区大小，增加它可得到更好处理的索引(对所有读和多重写)，到你能负担得起那样多。如果你使它太大，
# 系统将开始换页并且真的变慢了。对于内存在4GB左右的服务器该参数可设置为384M或512M。通过检查状态值Key_read_requests和Key_reads，
# 可以知道key_buffer_size设置是否合理。比例key_reads/key_read_requests应该尽可能的低，
# 至少是1:100，1:1000更好(上述状态值可以使用SHOW STATUS LIKE 'key_read%'获得)。注意：该参数值设置的过大反而会是服务器整体效率降低

ft_min_word_len = 4
# 分词词汇最小长度，默认4

transaction_isolation = REPEATABLE-READ
# MySQL支持4种事务隔离级别，他们分别是：
# READ-UNCOMMITTED, READ-COMMITTED, REPEATABLE-READ, SERIALIZABLE.
# 如没有指定，MySQL默认采用的是REPEATABLE-READ，ORACLE默认的是READ-COMMITTED

log_bin = mysql-bin
binlog_format = mixed
expire_logs_days = 30 #超过30天的binlog删除

log_error = /data/mysql/mysql-error.log #错误日志路径
slow_query_log = 1
long_query_time = 1 #慢查询时间 超过1秒则为慢查询
slow_query_log_file = /data/mysql/mysql-slow.log

performance_schema = 0
explicit_defaults_for_timestamp

#lower_case_table_names = 1 #不区分大小写

skip-external-locking #MySQL选项以避免外部锁定。该选项默认开启

default-storage-engine = InnoDB #默认存储引擎

innodb_file_per_table = 1
# InnoDB为独立表空间模式，每个数据库的每个表都会生成一个数据空间
# 独立表空间优点：
# 1．每个表都有自已独立的表空间。
# 2．每个表的数据和索引都会存在自已的表空间中。
# 3．可以实现单表在不同的数据库中移动。
# 4．空间可以回收（除drop table操作处，表空不能自已回收）
# 缺点：
# 单表增加过大，如超过100G
# 结论：
# 共享表空间在Insert操作上少有优势。其它都没独立表空间表现好。当启用独立表空间时，请合理调整：innodb_open_files

innodb_open_files = 500
# 限制Innodb能打开的表的数据，如果库里的表特别多的情况，请增加这个。这个值默认是300

innodb_buffer_pool_size = 64M
# InnoDB使用一个缓冲池来保存索引和原始数据, 不像MyISAM.
# 这里你设置越大,你在存取表里面数据时所需要的磁盘I/O越少.
# 在一个独立使用的数据库服务器上,你可以设置这个变量到服务器物理内存大小的80%
# 不要设置过大,否则,由于物理内存的竞争可能导致操作系统的换页颠簸.
# 注意在32位系统上你每个进程可能被限制在 2-3.5G 用户层面内存限制,
# 所以不要设置的太高.

innodb_write_io_threads = 4
innodb_read_io_threads = 4
# innodb使用后台线程处理数据页上的读写 I/O(输入输出)请求,根据你的 CPU 核数来更改,默认是4
# 注:这两个参数不支持动态改变,需要把该参数加入到my.cnf里，修改完后重启MySQL服务,允许值的范围从 1-64

innodb_thread_concurrency = 0
# 默认设置为 0,表示不限制并发数，这里推荐设置为0，更好去发挥CPU多核处理能力，提高并发量

innodb_purge_threads = 1
# InnoDB中的清除操作是一类定期回收无用数据的操作。在之前的几个版本中，清除操作是主线程的一部分，这意味着运行时它可能会堵塞其它的数据库操作。
# 从MySQL5.5.X版本开始，该操作运行于独立的线程中,并支持更多的并发数。用户可通过设置innodb_purge_threads配置参数来选择清除操作是否使用单
# 独线程,默认情况下参数设置为0(不使用单独线程),设置为 1 时表示使用单独的清除线程。建议为1

innodb_flush_log_at_trx_commit = 2
# 0：如果innodb_flush_log_at_trx_commit的值为0,log buffer每秒就会被刷写日志文件到磁盘，提交事务的时候不做任何操作（执行是由mysql的master thread线程来执行的。
# 主线程中每秒会将重做日志缓冲写入磁盘的重做日志文件(REDO LOG)中。不论事务是否已经提交）默认的日志文件是ib_logfile0,ib_logfile1
# 1：当设为默认值1的时候，每次提交事务的时候，都会将log buffer刷写到日志。
# 2：如果设为2,每次提交事务都会写日志，但并不会执行刷的操作。每秒定时会刷到日志文件。要注意的是，并不能保证100%每秒一定都会刷到磁盘，这要取决于进程的调度。
# 每次事务提交的时候将数据写入事务日志，而这里的写入仅是调用了文件系统的写入操作，而文件系统是有 缓存的，所以这个写入并不能保证数据已经写入到物理磁盘
# 默认值1是为了保证完整的ACID。当然，你可以将这个配置项设为1以外的值来换取更高的性能，但是在系统崩溃的时候，你将会丢失1秒的数据。
# 设为0的话，mysqld进程崩溃的时候，就会丢失最后1秒的事务。设为2,只有在操作系统崩溃或者断电的时候才会丢失最后1秒的数据。InnoDB在做恢复的时候会忽略这个值。
# 总结
# 设为1当然是最安全的，但性能页是最差的（相对其他两个参数而言，但不是不能接受）。如果对数据一致性和完整性要求不高，完全可以设为2，如果只最求性能，例如高并发写的日志服务器，设为0来获得更高性能

innodb_log_buffer_size = 2M
# 此参数确定些日志文件所用的内存大小，以M为单位。缓冲区更大能提高性能，但意外的故障将会丢失数据。MySQL开发人员建议设置为1－8M之间

innodb_log_file_size = 32M
# 此参数确定数据日志文件的大小，更大的设置可以提高性能，但也会增加恢复故障数据库所需的时间

innodb_log_files_in_group = 3
# 为提高性能，MySQL可以以循环方式将日志文件写到多个文件。推荐设置为3

innodb_max_dirty_pages_pct = 90
# innodb主线程刷新缓存池中的数据，使脏数据比例小于90%

innodb_lock_wait_timeout = 120 
# InnoDB事务在被回滚之前可以等待一个锁定的超时秒数。InnoDB在它自己的锁定表中自动检测事务死锁并且回滚事务。InnoDB用LOCK TABLES语句注意到锁定设置。默认值是50秒

bulk_insert_buffer_size = 8M
# 批量插入缓存大小， 这个参数是针对MyISAM存储引擎来说的。适用于在一次性插入100-1000+条记录时， 提高效率。默认值是8M。可以针对数据量的大小，翻倍增加。

myisam_sort_buffer_size = 8M
# MyISAM设置恢复表之时使用的缓冲区的尺寸，当在REPAIR TABLE或用CREATE INDEX创建索引或ALTER TABLE过程中排序 MyISAM索引分配的缓冲区

myisam_max_sort_file_size = 10G
# 如果临时文件会变得超过索引，不要使用快速排序索引方法来创建一个索引。注释：这个参数以字节的形式给出

myisam_repair_threads = 1
# 如果该值大于1，在Repair by sorting过程中并行创建MyISAM表索引(每个索引在自己的线程内) 

interactive_timeout = 28800
# 服务器关闭交互式连接前等待活动的秒数。交互式客户端定义为在mysql_real_connect()中使用CLIENT_INTERACTIVE选项的客户端。默认值：28800秒（8小时）

wait_timeout = 28800
# 服务器关闭非交互连接之前等待活动的秒数。在线程启动时，根据全局wait_timeout值或全局interactive_timeout值初始化会话wait_timeout值，
# 取决于客户端类型(由mysql_real_connect()的连接选项CLIENT_INTERACTIVE定义)。参数默认值：28800秒（8小时）
# MySQL服务器所支持的最大连接数是有上限的，因为每个连接的建立都会消耗内存，因此我们希望客户端在连接到MySQL Server处理完相应的操作后，
# 应该断开连接并释放占用的内存。如果你的MySQL Server有大量的闲置连接，他们不仅会白白消耗内存，而且如果连接一直在累加而不断开，
# 最终肯定会达到MySQL Server的连接上限数，这会报'too many connections'的错误。对于wait_timeout的值设定，应该根据系统的运行情况来判断。
# 在系统运行一段时间后，可以通过show processlist命令查看当前系统的连接状态，如果发现有大量的sleep状态的连接进程，则说明该参数设置的过大，
# 可以进行适当的调整小些。要同时设置interactive_timeout和wait_timeout才会生效。

[mysqldump]
quick
max_allowed_packet = 16M #服务器发送和接受的最大包长度

[myisamchk]
key_buffer_size = 8M
sort_buffer_size = 8M
read_buffer = 4M
write_buffer = 4M

```

##### my.cnf

```mysql
# *** 应用定制选项 ***

#
# MySQL 服务端
#
[mysqld]

# 一般配置选项
port = @MYSQL_TCP_PORT@
socket = @MYSQL_UNIX_ADDR@

# back_log 是操作系统在监听队列中所能保持的连接数,
# 队列保存了在MySQL连接管理器线程处理之前的连接.
# 如果你有非常高的连接率并且出现”connection refused” 报错,
# 你就应该增加此处的值.
# 检查你的操作系统文档来获取这个变量的最大值.
# 如果将back_log设定到比你操作系统限制更高的值,将会没有效果
back_log = 50

# 不在TCP/IP端口上进行监听.
# 如果所有的进程都是在同一台服务器连接到本地的mysqld,
# 这样设置将是增强安全的方法
# 所有mysqld的连接都是通过Unix sockets 或者命名管道进行的.
# 注意在windows下如果没有打开命名管道选项而只是用此项
# (通过 “enable-named-pipe” 选项) 将会导致mysql服务没有任何作用!
#skip-networking

# MySQL 服务所允许的同时会话数的上限
# 其中一个连接将被SUPER权限保留作为管理员登录.
# 即便已经达到了连接数的上限.
max_connections = 100
# 每个客户端连接最大的错误允许数量,如果达到了此限制.
# 这个客户端将会被MySQL服务阻止直到执行了”FLUSH HOSTS” 或者服务重启
# 非法的密码以及其他在链接时的错误会增加此值.
# 查看 “Aborted_connects” 状态来获取全局计数器.
max_connect_errors = 10

# 所有线程所打开表的数量.
# 增加此值就增加了mysqld所需要的文件描述符的数量
# 这样你需要确认在[mysqld_safe]中 “open-files-limit” 变量设置打开文件数量允许至少4096
table_cache = 2048

# 允许外部文件级别的锁. 打开文件锁会对性能造成负面影响
# 所以只有在你在同样的文件上运行多个数据库实例时才使用此选项(注意仍会有其他约束!)
# 或者你在文件层面上使用了其他一些软件依赖来锁定MyISAM表
#external-locking

# 服务所能处理的请求包的最大大小以及服务所能处理的最大的请求大小(当与大的BLOB字段一起工作时相当必要)
# 每个连接独立的大小.大小动态增加
max_allowed_packet = 16M

# 在一个事务中binlog为了记录SQL状态所持有的cache大小
# 如果你经常使用大的,多声明的事务,你可以增加此值来获取更大的性能.
# 所有从事务来的状态都将被缓冲在binlog缓冲中然后在提交后一次性写入到binlog中
# 如果事务比此值大, 会使用磁盘上的临时文件来替代.
# 此缓冲在每个连接的事务第一次更新状态时被创建
binlog_cache_size = 1M

# 独立的内存表所允许的最大容量.
# 此选项为了防止意外创建一个超大的内存表导致永尽所有的内存资源.
max_heap_table_size = 64M

# 排序缓冲被用来处理类似ORDER BY以及GROUP BY队列所引起的排序
# 如果排序后的数据无法放入排序缓冲,
# 一个用来替代的基于磁盘的合并分类会被使用
# 查看 “Sort_merge_passes” 状态变量.
# 在排序发生时由每个线程分配
sort_buffer_size = 8M

# 此缓冲被使用来优化全联合(full JOINs 不带索引的联合).
# 类似的联合在极大多数情况下有非常糟糕的性能表现,
# 但是将此值设大能够减轻性能影响.
# 通过 “Select_full_join” 状态变量查看全联合的数量
# 当全联合发生时,在每个线程中分配
join_buffer_size = 8M

# 我们在cache中保留多少线程用于重用
# 当一个客户端断开连接后,如果cache中的线程还少于thread_cache_size,
# 则客户端线程被放入cache中.
# 这可以在你需要大量新连接的时候极大的减少线程创建的开销
# (一般来说如果你有好的线程模型的话,这不会有明显的性能提升.)
thread_cache_size = 8

# 此允许应用程序给予线程系统一个提示在同一时间给予渴望被运行的线程的数量.
# 此值只对于支持 thread_concurrency() 函数的系统有意义( 例如Sun Solaris).
# 你可可以尝试使用 [CPU数量]*(2..4) 来作为thread_concurrency的值
thread_concurrency = 8

# 查询缓冲常被用来缓冲 SELECT 的结果并且在下一次同样查询的时候不再执行直接返回结果.
# 打开查询缓冲可以极大的提高服务器速度, 如果你有大量的相同的查询并且很少修改表.
# 查看 “Qcache_lowmem_prunes” 状态变量来检查是否当前值对于你的负载来说是否足够高.
# 注意: 在你表经常变化的情况下或者如果你的查询原文每次都不同,
# 查询缓冲也许引起性能下降而不是性能提升.转载请注明文章来源:[http://www.linuxso.com/a/linuxrumen/413.html](http://www.linuxso.com/a/linuxrumen/413.html)

query_cache_size = 64M

# 只有小于此设定值的结果才会被缓冲
# 此设置用来保护查询缓冲,防止一个极大的结果集将其他所有的查询结果都覆盖.
query_cache_limit = 2M

# 被全文检索索引的最小的字长.
# 你也许希望减少它,如果你需要搜索更短字的时候.
# 注意在你修改此值之后,
# 你需要重建你的 FULLTEXT 索引
ft_min_word_len = 4

# 如果你的系统支持 memlock() 函数,你也许希望打开此选项用以让运行中的mysql在在内存高度紧张的时候,数据在内存中保持锁定并且防止可能被swapping out
# 此选项对于性能有益
#memlock

# 当创建新表时作为默认使用的表类型,
# 如果在创建表示没有特别执行表类型,将会使用此值
default_table_type = MYISAM

# 线程使用的堆大小. 此容量的内存在每次连接时被预留.
# MySQL 本身常不会需要超过64K的内存
# 如果你使用你自己的需要大量堆的UDF函数
# 或者你的操作系统对于某些操作需要更多的堆,
# 你也许需要将其设置的更高一点.
thread_stack = 192K

# 设定默认的事务隔离级别.可用的级别如下:
# READ-UNCOMMITTED, READ-COMMITTED, REPEATABLE-READ, SERIALIZABLE
transaction_isolation = REPEATABLE-READ

# 内部(内存中)临时表的最大大小
# 如果一个表增长到比此值更大,将会自动转换为基于磁盘的表.
# 此限制是针对单个表的,而不是总和.
tmp_table_size = 64M

# 打开二进制日志功能.
# 在复制(replication)配置中,作为MASTER主服务器必须打开此项
# 如果你需要从你最后的备份中做基于时间点的恢复,你也同样需要二进制日志.
log-bin=mysql-bin

# 如果你在使用链式从服务器结构的复制模式 (A->B->C),
# 你需要在服务器B上打开此项.
# 此选项打开在从线程上重做过的更新的日志,
# 并将其写入从服务器的二进制日志.
#log_slave_updates

# 打开全查询日志. 所有的由服务器接收到的查询 (甚至对于一个错误语法的查询)
# 都会被记录下来. 这对于调试非常有用, 在生产环境中常常关闭此项.
#log

# 将警告打印输出到错误log文件. 如果你对于MySQL有任何问题
# 你应该打开警告log并且仔细审查错误日志,查出可能的原因.
#log_warnings

# 记录慢速查询. 慢速查询是指消耗了比 “long_query_time” 定义的更多时间的查询.
# 如果 log_long_format 被打开,那些没有使用索引的查询也会被记录.
# 如果你经常增加新查询到已有的系统内的话. 一般来说这是一个好主意,
log_slow_queries

# 所有的使用了比这个时间(以秒为单位)更多的查询会被认为是慢速查询.
# 不要在这里使用”1″, 否则会导致所有的查询,甚至非常快的查询页被记录下来(由于MySQL 目前时间的精确度只能达到秒的级别).
long_query_time = 2

# 在慢速日志中记录更多的信息.
# 一般此项最好打开.
# 打开此项会记录使得那些没有使用索引的查询也被作为到慢速查询附加到慢速日志里
log_long_format

# 此目录被MySQL用来保存临时文件.例如,
# 它被用来处理基于磁盘的大型排序,和内部排序一样.
# 以及简单的临时表.
# 如果你不创建非常大的临时文件,将其放置到 swapfs/tmpfs 文件系统上也许比较好
# 另一种选择是你也可以将其放置在独立的磁盘上.
# 你可以使用”;”来放置多个路径
# 他们会按照roud-robin方法被轮询使用.
#tmpdir = /tmp

# *** 复制有关的设置

# 唯一的服务辨识号,数值位于 1 到 2^32-1之间.
# 此值在master和slave上都需要设置.
# 如果 “master-host” 没有被设置,则默认为1, 但是如果忽略此选项,MySQL不会作为master生效.
server-id = 1

# 复制的Slave (去掉master段的注释来使其生效)
#
# 为了配置此主机作为复制的slave服务器,你可以选择两种方法:
#
# 1) 使用 CHANGE MASTER TO 命令 (在我们的手册中有完整描述) -
# 语法如下:
#
# CHANGE MASTER TO MASTER_HOST=, MASTER_PORT= ,
# MASTER_USER=, MASTER_PASSWORD= ;
#
# 你需要替换掉 , , 等被尖括号包围的字段以及使用master的端口号替换 (默认3306).
#
# 例子:
#
# CHANGE MASTER TO MASTER_HOST=’125.564.12.1′, MASTER_PORT=3306,
# MASTER_USER=’joe’, MASTER_PASSWORD=’secret’;
#
# 或者
#
# 2) 设置以下的变量. 不论如何, 在你选择这种方法的情况下, 然后第一次启动复制(甚至不成功的情况下,
# 例如如果你输入错密码在master-password字段并且slave无法连接),
# slave会创建一个 master.info 文件,并且之后任何对于包含在此文件内的参数的变化都会被忽略
# 并且由 master.info 文件内的内容覆盖, 除非你关闭slave服务, 删除 master.info 并且重启slave 服务.
# 由于这个原因,你也许不想碰一下的配置(注释掉的) 并且使用 CHANGE MASTER TO (查看上面) 来代替
#
# 所需要的唯一id号位于 2 和 2^32 – 1之间
# (并且和master不同)
# 如果master-host被设置了.则默认值是2
# 但是如果省略,则不会生效
#server-id = 2
#
# 复制结构中的master – 必须
#master-host = 
#
# 当连接到master上时slave所用来认证的用户名 – 必须
#master-user = 
#
# 当连接到master上时slave所用来认证的密码 – 必须
#master-password = #转载请注明文章来源:[http://www.linuxso.com/a/linuxrumen/413.html](http://www.linuxso.com/a/linuxrumen/413.html)

# master监听的端口.
# 可选 – 默认是3306
#master-port =

# 使得slave只读.只有用户拥有SUPER权限和在上面的slave线程能够修改数据.
# 你可以使用此项去保证没有应用程序会意外的修改slave而不是master上的数据
#read_only

#*** MyISAM 相关选项

# 关键词缓冲的大小, 一般用来缓冲MyISAM表的索引块.
# 不要将其设置大于你可用内存的30%,
# 因为一部分内存同样被OS用来缓冲行数据
# 甚至在你并不使用MyISAM 表的情况下, 你也需要仍旧设置起 8-64M 内存由于它同样会被内部临时磁盘表使用.
key_buffer_size = 32M

# 用来做MyISAM表全表扫描的缓冲大小.
# 当全表扫描需要时,在对应线程中分配.
read_buffer_size = 2M

# 当在排序之后,从一个已经排序好的序列中读取行时,行数据将从这个缓冲中读取来防止磁盘寻道.
# 如果你增高此值,可以提高很多ORDER BY的性能.
# 当需要时由每个线程分配
read_rnd_buffer_size = 16M

# MyISAM 使用特殊的类似树的cache来使得突发插入
# (这些插入是,INSERT … SELECT, INSERT … VALUES (…), (…), …, 以及 LOAD DATA
# INFILE) 更快. 此变量限制每个进程中缓冲树的字节数.
# 设置为 0 会关闭此优化.
# 为了最优化不要将此值设置大于 “key_buffer_size”.
# 当突发插入被检测到时此缓冲将被分配.
bulk_insert_buffer_size = 64M

# 此缓冲当MySQL需要在 REPAIR, OPTIMIZE, ALTER 以及 LOAD DATA INFILE 到一个空表中引起重建索引时被分配.
# 这在每个线程中被分配.所以在设置大值时需要小心.
myisam_sort_buffer_size = 128M

# MySQL重建索引时所允许的最大临时文件的大小 (当 REPAIR, ALTER TABLE 或者 LOAD DATA INFILE).
# 如果文件大小比此值更大,索引会通过键值缓冲创建(更慢)
myisam_max_sort_file_size = 10G

# 如果被用来更快的索引创建索引所使用临时文件大于制定的值,那就使用键值缓冲方法.
# 这主要用来强制在大表中长字串键去使用慢速的键值缓冲方法来创建索引.
myisam_max_extra_sort_file_size = 10G

# 如果一个表拥有超过一个索引, MyISAM 可以通过并行排序使用超过一个线程去修复他们.
# 这对于拥有多个CPU以及大量内存情况的用户,是一个很好的选择.
myisam_repair_threads = 1

# 自动检查和修复没有适当关闭的 MyISAM 表.
myisam_recover

# 默认关闭 Federated
skip-federated

# *** BDB 相关选项 ***

# 如果你运行的MySQL服务有BDB支持但是你不准备使用的时候使用此选项. 这会节省内存并且可能加速一些事.
skip-bdb

# *** INNODB 相关选项 ***

# 如果你的MySQL服务包含InnoDB支持但是并不打算使用的话,
# 使用此选项会节省内存以及磁盘空间,并且加速某些部分
#skip-innodb

# 附加的内存池被InnoDB用来保存 metadata 信息
# 如果InnoDB为此目的需要更多的内存,它会开始从OS这里申请内存.
# 由于这个操作在大多数现代操作系统上已经足够快, 你一般不需要修改此值.
# SHOW INNODB STATUS 命令会显示当先使用的数量.
innodb_additional_mem_pool_size = 16M

# InnoDB使用一个缓冲池来保存索引和原始数据, 不像 MyISAM.
# 这里你设置越大,你在存取表里面数据时所需要的磁盘I/O越少.
# 在一个独立使用的数据库服务器上,你可以设置这个变量到服务器物理内存大小的80%
# 不要设置过大,否则,由于物理内存的竞争可能导致操作系统的换页颠簸.
# 注意在32位系统上你每个进程可能被限制在 2-3.5G 用户层面内存限制,
# 所以不要设置的太高.
innodb_buffer_pool_size = 2G

# InnoDB 将数据保存在一个或者多个数据文件中成为表空间.
# 如果你只有单个逻辑驱动保存你的数据,一个单个的自增文件就足够好了.
# 其他情况下.每个设备一个文件一般都是个好的选择.
# 你也可以配置InnoDB来使用裸盘分区 – 请参考手册来获取更多相关内容
innodb_data_file_path = ibdata1:10M:autoextend

# 设置此选项如果你希望InnoDB表空间文件被保存在其他分区.
# 默认保存在MySQL的datadir中.
#innodb_data_home_dir =

# 用来同步IO操作的IO线程的数量. This value is
# 此值在Unix下被硬编码为4,但是在Windows磁盘I/O可能在一个大数值下表现的更好.
innodb_file_io_threads = 4

# 如果你发现InnoDB表空间损坏, 设置此值为一个非零值可能帮助你导出你的表.
# 从1开始并且增加此值知道你能够成功的导出表.
#innodb_force_recovery=1

# 在InnoDb核心内的允许线程数量.
# 最优值依赖于应用程序,硬件以及操作系统的调度方式.
# 过高的值可能导致线程的互斥颠簸.
innodb_thread_concurrency = 16

# 如果设置为1 ,InnoDB会在每次提交后刷新(fsync)事务日志到磁盘上,
# 这提供了完整的ACID行为.
# 如果你愿意对事务安全折衷, 并且你正在运行一个小的食物, 你可以设置此值到0或者2来减少由事务日志引起的磁盘I/O
# 0代表日志只大约每秒写入日志文件并且日志文件刷新到磁盘.
# 2代表日志写入日志文件在每次提交后,但是日志文件只有大约每秒才会刷新到磁盘上.
innodb_flush_log_at_trx_commit = 1

# 加速InnoDB的关闭. 这会阻止InnoDB在关闭时做全清除以及插入缓冲合并.
# 这可能极大增加关机时间, 但是取而代之的是InnoDB可能在下次启动时做这些操作.
#innodb_fast_shutdown

# 用来缓冲日志数据的缓冲区的大小.
# 当此值快满时, InnoDB将必须刷新数据到磁盘上.
# 由于基本上每秒都会刷新一次,所以没有必要将此值设置的太大(甚至对于长事务而言)

innodb_log_buffer_size = 8M

# 在日志组中每个日志文件的大小.转载请注明文章来源:[http://www.linuxso.com/a/linuxrumen/413.html](http://www.linuxso.com/a/linuxrumen/413.html)

# 你应该设置日志文件总合大小到你缓冲池大小的25%~100%
# 来避免在日志文件覆写上不必要的缓冲池刷新行为.
# 不论如何, 请注意一个大的日志文件大小会增加恢复进程所需要的时间.
innodb_log_file_size = 256M

# 在日志组中的文件总数.
# 通常来说2~3是比较好的.
innodb_log_files_in_group = 3

# InnoDB的日志文件所在位置. 默认是MySQL的datadir.
# 你可以将其指定到一个独立的硬盘上或者一个RAID1卷上来提高其性能
#innodb_log_group_home_dir

# 在InnoDB缓冲池中最大允许的脏页面的比例.
# 如果达到限额, InnoDB会开始刷新他们防止他们妨碍到干净数据页面.
# 这是一个软限制,不被保证绝对执行.
innodb_max_dirty_pages_pct = 90

# InnoDB用来刷新日志的方法.
# 表空间总是使用双重写入刷新方法
# 默认值是 “fdatasync”, 另一个是 “O_DSYNC”.
#innodb_flush_method=O_DSYNC

# 在被回滚前,一个InnoDB的事务应该等待一个锁被批准多久.
# InnoDB在其拥有的锁表中自动检测事务死锁并且回滚事务.
# 如果你使用 LOCK TABLES 指令, 或者在同样事务中使用除了InnoDB以外的其他事务安全的存储引擎
# 那么一个死锁可能发生而InnoDB无法注意到.
# 这种情况下这个timeout值对于解决这种问题就非常有帮助.
innodb_lock_wait_timeout = 120

[mysqldump]
# 不要在将内存中的整个结果写入磁盘之前缓存. 在导出非常巨大的表时需要此项
quick

max_allowed_packet = 16M

[mysql]
no-auto-rehash

# 仅仅允许使用键值的 UPDATEs 和 DELETEs .
#safe-updates

[isamchk]
key_buffer = 512M
sort_buffer_size = 512M
read_buffer = 8M
write_buffer = 8M

[myisamchk]
key_buffer = 512M
sort_buffer_size = 512M
read_buffer = 8M
write_buffer = 8M

[mysqlhotcopy]
interactive-timeout

[mysqld_safe]
# 增加每个进程的可打开文件数量.
# 警告: 确认你已经将全系统限制设定的足够高!
# 打开大量表需要将此值设b
open-files-limit = 8192

示例：
安装

rmp -ivh MySQL-server-4.1.22-0.glibc23.i386.rpm --nodeps

rmp -ivh MySQL-client-4.1.22-0.glibc23.i386.rpm --nodeps

查看是否安装成功

netstat -atln 命令看到3306端口开放说明安装成功

登录

mysql [-u username] [-h host] [-p[password]] [dbname]

初始无密码，这个mysql可执行文件在/usr/bin/mysql

目录

1、数据库目录

　　  /var/lib/mysql/

2、配置文件
/usr/share/mysql（mysql.server命令及配置文件）
3、相关命令
/usr/bin(mysqladmin mysqldump等命令)
4、启动脚本
/etc/rc.d/init.d/（启动脚本文件mysql的目录）
修改登录密码
MySQL默认没有密码
usr/bin/mysqladmin -u root password 'new-password'
格式：mysqladmin -u用户名 -p旧密码 password 新密码

启动与停止

MySQL安装完成后启动文件mysql在/etc/init.d目录下，在需要启动时运行下面命令即可

启动：

/etc/init.d/mysql start
停止：
/usr/bin/mysqladmin -u root -p shutdown
重新动：

sudo /etc/init.d/mysql restart 

自动启动：

察看mysql是否在自动启动列表中 /sbin/chkconfig --list

把MySQL添加到你系统的启动服务组里面去 /sbin/chkconfig --add mysql

把MySQL从启动服务组里面删除 /sbin/chkconfig --del mysql

配置

将/usr/share/mysql/my-medium.cnf复制到/etc/my.cnf，以后修改my.cnf文件来修改mysql的全局设置

将my.cnf文件中的innodb_flush_log_at_trx_commit设成0来优化

  [mysqld]后添加添加lower_case_table_names设成1来不区分表名的大小写

设置字符集
MySQL的默认编码是Latin1，不支持中文，要支持需要把数据库的默认编码修改为gbk或者utf8。
1、中止MySQL服务（bin/mysqladmin -u root shutdown）
2、在/etc/下找到my.cnf，如果没有就把MySQL的安装目录下的support-files目录下的my-medium.cnf复制到/etc/下并改名为my.cnf即可 
3、打开my.cnf以后，在[client]和[mysqld]下面均加上default-character-set=utf8，保存并关闭
4、启动MySQL服务（bin/mysqld_safe &）
查询字符集：show variables like '%set%';
增加MySQL用户
格式：grant select on 数据库.* to 用户名@登录主机 identified by "密码"
grant select,insert,update,delete on *.* to user_1@'%' Identified by '123';
grant all on *.* to user_1@'localhost' Identified by '123';
远程访问
　　其一：
　　GRANT ALL PRIVILEGES ON *.* TO xoops_root@'%' IDENTIFIED BY '654321';
　　允许xoops_root用户可以从任意机器上登入MySQL。
　　其二：
　　编辑 /etc/mysql/my.cnf
　　>skip-networking => # skip-networking
　　这样就可以允许其他机器访问MySQL了。
   grant all on *.* to 'root'@'ip' identified by 'password'; 
备份与恢复
备份
进入到库目录，cd /val/lib/mysql
mysqldump -u root -p --opt aaa > back_aaa
恢复
mysql -u root -p ccc < back_aaa
```

