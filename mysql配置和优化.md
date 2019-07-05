```
show status; 显示状态信息;
show variables; 显示系统变量;
show processlist; 查看当前SQL执行，包括执行状态、是否锁表等;
```

## MySql参数配置

### `group_concat_max_len`

```
group_concat_max_len=-1（-1为最大值或根据实际需求设置长度）
group_concat函数结果字符串的长度；

不能擅自重启MySQL服务，则可以通过语句设置group_concat的作用范围，如：
SET GLOBAL group_concat_max_len=-1;
SET SESSION group_concat_max_len=-1;
```

## MySQL非缓存参数变量介绍及修改

### `wait_timeout`

```
wait_timeout=3000（单位为妙）
interactive_timeout= 3000（单位为妙）
MySQL客户端的数据库连接闲置最大时间值。

wait_timeout: 服务器关闭非交互连接之前等待活动的秒数。
interactive_timeout: 服务器关闭交互式连接前等待活动的秒数。
这两个参数必须配合使用。否则单独设置wait_timeout无效。
```

### `max_connections`

```
max_connections=3000（由默认的151，修改为3000（750M））
MySql的最大连接数;
如果连接数越多，介于MySql会为每个连接提供连接缓冲区，就会开销越多的内存，所以要适当调整该值，不能盲目提高设值。
可以通过下面命令查看当前已使用过的最大连接数来参考；
show status like 'max_used_connections';
MySQL服务器允许的最大连接数16384；
```

### `back_log`

```
back_log=300
MySQL能暂存的连接数量。
当主要MySQL线程在一个很短时间内得到非常多的连接请求，这就起作用。
如果MySQL的连接数据达到max_connections时，新来的请求将会被存在堆栈中，以等待某一连接释放资源，该堆栈的数量即back_log，如果等待连接的数量超过back_log，将不被授予连接资源。
默认数值是50，可调优为128，对于Linux系统设置范围为小于512的整数。
```

### `max_user_connections`

```
max_user_connections=800（由默认的0（不限制），修改为800）

针对某一个账号的所有客户端并行连接到MYSQL服务的最大并行连接数。
简单说是指同一个账号能够同时连接到mysql服务的最大连接数。
```

## MySQL缓存变量介绍及修改

### `query_cache_size`

```
query_cache_size=40M
使用查询缓冲，MySQL将查询结果存放在缓冲区中，今后对于同样的SELECT语句（区分大小写），将直接从缓冲区中读取结果。

通过检查状态值Qcache_*，可以知道query_cache_size设置是否合理: show status like 'Qcache%';
如果Qcache_lowmem_prunes的值非常大，则表明经常出现缓冲不够的情况，
如果Qcache_hits的值也非常大，则表明查询缓冲使用非常频繁，此时需要增加缓冲大小；
如果Qcache_hits的值不大，则表明你的查询重复率很低，这种情况下使用查询缓冲反而会影响效率，那么可以考虑不用查询缓冲。

相关配置还有：show variables like 'query_cache_%';
query_cache_type指定是否使用查询缓冲，可以设置为0、1、2，该变量是SESSION级的变量。
query_cache_limit指定单个查询能够使用的缓冲区大小，缺省为1M。
query_cache_min_res_unit是在4.1版本以后引入的，它指定分配缓冲区空间的最小单位，缺省为4K。
检查状态值Qcache_free_blocks，如果该值非常大，则表明缓冲区中碎片很多，这就表明查询结果都比较小，此时需要减小query_cache_min_res_unit。
```

### `read_buffer_size`

```
read_buffer_size=16773120
每个进行一个顺序扫描的线程为其扫描的每张表分配这个大小的一个缓冲区。如果你做很多顺序扫描，你可能想要增加该值。
默认数值是131072(128K)，可改为16773120 (16M)。
```

### `read_rnd_buffer_size `

```
read_rnd_buffer_size=16773120
随机读缓冲区大小。当按任意顺序读取行时(例如，按照排序顺序)，将分配一个随机读缓存区。
进行排序查询时，MySQL会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速度，如果需要排序大量数据，可适当调高该值。
但MySQL会为每个客户连接发放该缓冲空间，所以应尽量适当设置该值，以避免内存开销过大。
默认数值是262144(256K)，可改为16773120 (16M)。
```

### `sort_buffer_size `

```
sort_buffer_size=16773120
每个需要进行排序的线程分配该大小的一个缓冲区。增加这值加速ORDER BY或GROUP BY操作。
默认数值是2097144(2M)，可改为16777208 (16M)。
```

### `innodb_buffer_pool_size`

```
innodb_buffer_pool_size=2048M
用于索引块的缓冲区大小：主要针对InnoDB表性能影响最大的一个参数。

InnoDB使用该参数指定大小的内存来缓冲数据和索引。对于单独的MySQL数据库服务器，最大可以把该值设置成物理内存的80%。
```

### `innodb_flush_log_at_trx_commit`

```
innodb_flush_log_at_trx_commit=2
主要控制了innodb将log buffer中的数据写入日志文件并flush磁盘的时间点，取值分别为0、1、2三个。
0，表示当事务提交时，不做日志写入操作，而是每秒钟将log buffer中的数据写入日志文件并flush磁盘一次；
1，则在每秒钟或是每次事物的提交都会引起日志文件写入、flush磁盘的操作，确保了事务的ACID；
2，每次事务提交引起写入日志文件的动作，但每秒钟完成一次flush磁盘操作。
实际测试发现，该值对插入数据的速度影响非常大，设置为2时插入10000条记录只需要2秒，设置为0时只需要1秒，而设置为1时则需要229秒。
因此，MySQL手册也建议尽量将插入操作合并成一个事务，这样可以大幅提高速度。
根据MySQL手册，在允许丢失最近部分事务的危险的前提下，可以把该值设为0或2。
```

### `innodb_log_buffer_size`

```
innodb_log_buffer_size=20M
事务日志所使用的缓冲区，对于较大的事务，可以增大缓存大小。
默认是8MB，系的如频繁的系统可适当增大至4MB～8MB。一般来说不建议超过32MB
```

### `innodb_additional_mem_pool_size`

```
innodb_additional_mem_pool_size=20M
该参数指定InnoDB用来存储数据字典和其他内部数据结构的内存池大小。
通常不用太大，只要够用就行，应该与表结构的复杂度有关系。如果不够用，MySQL会在错误日志中写入一条警告信息。
对于2G内存的机器，推荐值是20M；32G内存的 100M。
```
