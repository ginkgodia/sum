

​	** Ubuntu 搭建zabbix + prometheus + grafana + orabbix  **



1. 搭建lnmp 架构

   mysql

   ```
   # 创建monitor_zabbix 数据库，并授予zabbix 所有权限
   create database monitor_zabbix character set utf8 collate utf8_bin;
   grant all privileges on monitor_zabbix.* to zabbix@'%' identified by '000000';
   # 授权zabbixjk  监控权限(监控数据库需要的最低权限)
   grant process, replication client, select on *.* to 'zabbixjk'@'%' identified by "000000";
   
   ```

   授权 monitor

   ```
   授权monitor用户执行mysql的权利
   在/etc/sudoer中授权用户执行权限
   root    ALL=(ALL:ALL) ALL
   monitor ALL=(root) /user/bin/mysql
   授权用户 主机=命令动作 
   这三个要素缺一不可，但在动作之前也可以指定切换到特定用户下，在这里指定切换的用户要用括号括起来，
   如果不需要密码直接运行命令的，应该加NOPASSWD:参数，但这些可以省略；举例说明；
   beinan ALL=/bin/chown,/bin/chmod 
   如果我们在/etc/sudoers 中添加这一行，表示beinan可以在任何可能出现的主机名的系统中，可以切换到root用户下执行/bin/chown和/bin/chmod 命令，通过sudo -l 来查看beinan 在这台主机上允许和禁止运行的命令； 
   值得注意的是，在这里省略了指定切换到哪个用户下执行/bin/shown 和/bin/chmod命令；在省略的情况下默认为是切换到root用户下执行；同时也省略了是不是需要beinan用户输入验证密码，如果省略了，默认为是需要验证密码。 
   授权用户 主机=[(切换到哪些用户或用户组)] [是否需要密码验证] 命令1,[(切换到哪些用户或用户组)] [是否需要密码验证] [命令2],[(切换到哪些用户或用户组)] [是否需要密码验证] [命令3]
   ```

php

```
依赖：sudo apt install libxml2-dev
	sudo apt install libjpeg-dev
	 sudo apt install libmcrypt-dev
编译参数:
./configure  --prefix=/home/monitor/php  --with-curl --with-zlib  --with-pcre-regex --with-mysqli --with-gd --with-jpeg-dir --with-png-dir --with-freetype-dir  --enable-fpm --with-mcrypt -with-mhash --with-----------
配置变动：
diff php.ini php.ini-production 
383c383
< max_execution_time = 600
---
> max_execution_time = 30
393c393
< max_input_time = 600
---
> max_input_time = 60
404c404
< memory_limit = 256M
---
> memory_limit = 128M
671c671
< post_max_size = 50M
---
> post_max_size = 8M
824c824
< upload_max_filesize = 50M
---
> upload_max_filesize = 2M
940d939
< date.timezone =RPC/


```

nginx 

```
编译参数
./configure  --prefix=/home/monitor/nginx/ --with-poll_module --with-http_stub_status_module  --with-http_gzip_static_module  --with-http_ssl_module  --with-openssl=/usr/lib/ssl

nginx是通过alias设置虚拟目录，在nginx的配置中，alias目录和root目录是有区别的：
1）alias指定的目录是准确的，即location匹配访问的path目录下的文件直接是在alias目录下查找的；
2）root指定的目录是location匹配访问的path目录的上一级目录,这个path目录一定要是真实存在root指定目录下的；
3）使用alias标签的目录块中不能使用rewrite的break（具体原因不明）；另外，alias指定的目录后面必须要加上"/"符号！！
4）alias虚拟目录配置中，location匹配的path目录如果后面不带"/"，那么访问的url地址中这个path目录后面加不加"/"不影响访问，访问时它会自动加上"/"；
    但是如果location匹配的path目录后面加上"/"，那么访问的url地址中这个path目录必须要加上"/"，访问时它不会自动加上"/"。如果不加上"/"，访问就会失败！
5）root目录配置中，location匹配的path目录后面带不带"/"，都不会影响访问。
```

zabbix

```
./configure  --prefix=/home/monitor/zabbix-server --enable-server --with-mysql  --with-libcurl --with-libxml

./configure  --prefix=/home/monitor/zabbix-agent --enable-agent --with-mysql  --with-libcurl --with-libxml
```

添加开机启动

```
php:
在源码目录下默认有个脚本文件，拷贝到/etc/init.d目录下就可以直接用
sudo cp init.d.php-fpm /etc/init.d/php-fpm
php-fpm启动脚本依赖与php-fpm.pid文件，此时需要在php-fpm.conf开启pid参数
/etc/init.d/php-fpm stop/start

sudo -u monitor /etc/init.d/php-fpm start
sudo -u monitor nginx
sudo -u monitor /home/monitor/zabbix-server/sbin/zabbix-server -c /home/monitor/zabbix-server/etc/zabbix_server.conf

```

Mysql 

```
分区分表
zabbix 的 history ,history_unit , trends， Trends_uint 数据量较大，需要分区分表
查询sql
mysql> use information_schema
mysql> select table_name, table_rows from tables  where TABLE_SCHEMA='monitor_zabbix' order by table_rows desc;
初始分区创建：
以日期中的日为单位，为history, history_uint 创建即日起9个连续的分区表，每个分区表只存放当日的历史数据，为trends和trends_uint 创建从本月起2个连续的分区表，每个分区表中只存放当月的历史数据
系统自动创建，删除分区表:
通过mysql 数据库创建job， 每天的凌晨三点调用存储过程的方式实现，对于history, history_uint表，删除7天前的历史分区表，创建8天后的分区表，对于trend, trends_int表，每月1日，创建下个月的分区表，删除三个月前的历史分区表，保留至少三个月的完整历史数据
运维监控保障：
通过zabbix 创建4个监控项，4个触发器，监控已创建的分区表是否创建成功，如果失败，则会产生严重告警(表存在就表示成功)
mysql 开启general_log跟踪sql执行记录方法如下：
方法1：
修改/etc/my.cnf ，增加以下配置
general_log=ON
general_log_file=/var/lib/mysql/log/sql_row.log
重启mysql
方法2：
mysql> set global general_log_file=/var/lib/mysql/log/sql_row.log
mysql > set global general_log=on
在general_log模式开启过程中，所有对数据库的操作都将被记录到general.log中
在linux中log目录只能设置到tmp 和/var/目录中，其他路径会出错

表分区的准备：
关掉zabbix 自带的house_keeping 删除过期的历史数据和趋势数据功能，
管理>一般>管家设置>开启内部管家复选框
点掉 Enable internal housekeeping <必须操作>

操作前准备：
1. 检查数据表是否为innoDB引擎(包括需要变更的四个表)
	show create table history  
	结果: ENGINE=innoDB DEFAULT CHARASET=UTF-8
2. 检查数据库是否未进行分区，包括需要变更的四个表
	show create table history 
	结果: 无PARTITION BY RANGE (lock)等信息
3. 检查mysql日志文件，检查是否存在warnning ,error等异常

系统备份：
mysqldump -h127.0.0.1 -uzabbix -p000000 monitor_zabbix.history >/data/backup/history.sql
mysqldump -h127.0.0.1 -uzabbix -p000000 monitor_zabbix.history_uint >/data/backup/history_uint.sql
mysqldump -h127.0.0.1 -uzabbix -p000000 monitor_zabbix.trends >/data/backup/trends.sql
mysqldump -h127.0.0.1 -uzabbix -p000000 monitor_zabbix.trends_uint >/data/backup/trends_uint.sql

变更操作：
1. 对zabbix历史数据，趋势表建立表分区
	建立history 表分区
    sql 语句中的unix_timestamp为执行明天的日期，并依次加一，分区名称p201****是当天的日期，并依次加一
    alter table history 
    PARTITION BY RANGE(clock)(
    PARTITION p20180729 VALUES LESS THAN (unix_timestamp("2018/07/30 00:00:00")),
    PARTITION p20180730 VALUES LESS THAN (unix_timestamp("2018/07/31 00:00:00")),
    PARTITION p20180731 VALUES LESS THAN (unix_timestamp("2018/08/01 00:00:00")),
    PARTITION p20180801 VALUES LESS THAN (unix_timestamp("2018/08/02 00:00:00")),
    PARTITION p20180802 VALUES LESS THAN (unix_timestamp("2018/08/03 00:00:00")),
    PARTITION p20180803 VALUES LESS THAN (unix_timestamp("2018/08/04 00:00:00")),
    PARTITION p20180804 VALUES LESS THAN (unix_timestamp("2018/08/05 00:00:00")),
    PARTITION p20180805 VALUES LESS THAN (unix_timestamp("2018/08/06 00:00:00")),
    );
    
2. 建立history_uint 表分区
	alter table history_uint 
    PARTITION BY RANGE(clock)(
    PARTITION p20180729 VALUES LESS THAN (unix_timestamp("2018/07/30 00:00:00")),
    PARTITION p20180730 VALUES LESS THAN (unix_timestamp("2018/07/31 00:00:00")),
    PARTITION p20180731 VALUES LESS THAN (unix_timestamp("2018/08/01 00:00:00")),
    PARTITION p20180801 VALUES LESS THAN (unix_timestamp("2018/08/02 00:00:00")),
    PARTITION p20180802 VALUES LESS THAN (unix_timestamp("2018/08/03 00:00:00")),
    PARTITION p20180803 VALUES LESS THAN (unix_timestamp("2018/08/04 00:00:00")),
    PARTITION p20180804 VALUES LESS THAN (unix_timestamp("2018/08/05 00:00:00")),
    PARTITION p20180805 VALUES LESS THAN (unix_timestamp("2018/08/06 00:00:00")),
    );
3. 建立trends 表分区
	alter table trends 
    PARTITION BY RANGE(clock)(
    PARTITION p201807 VALUES LESS THAN (unix_timestamp("2018/08/01 00:00:00")),
    PARTITION p201808 VALUES LESS THAN (unix_timestamp("2018/09/01 00:00:00")),
    PARTITION p201809 VALUES LESS THAN (unix_timestamp("2018/10/01 00:00:00")),
    )
    
4. 建立trends_uint 表分区
	alter table trends_uint 
    PARTITION BY RANGE(clock)(
    PARTITION p201807 VALUES LESS THAN (unix_timestamp("2018/08/01 00:00:00")),
    PARTITION p201808 VALUES LESS THAN (unix_timestamp("2018/09/01 00:00:00")),
    PARTITION p201809 VALUES LESS THAN (unix_timestamp("2018/10/01 00:00:00")),
    )
    
5. 检查数据表分区是否创建成功
select * from information_schema.'PARTITIONS' where table_name="history"
select * from information_schema.'PARTITIONS' where table_name="history_uint"
select * from information_schema.'PARTITIONS' where table_name="trends"
select * from information_schema.'PARTITIONS' where table_name="trends_uint"
6. 检查数据是否按表分区保存
select * from history PARTITION(p20180729) limit 10;
select * from history_uint PARTITION(p20180729) limit 10;
select * from trends PARTITION(p201807) limit 10;
select * from trends_uint PARTITION(p201808) limit 10;

自动创建分区，删除分区

配置mysql 参数，开启定时器功能(主备均需开启)
编辑 /etc/my.cnf 在[mysqld] 下配置
event_scheduler=ON (重启生效)

zabbix 用户登录设置参数和授权

mysql >set GLOBAL event_scheduler=ON(开启定时器功能，立刻生效)
mysql >update mysql.user set event_priv = "Y" where host='%' and user='zabbix'
mysql > flush privileges;

检查定时器是否开启成功
mysql> show variables like 'event_scheduler'; (为ON 表示成功)
mysql> slect host, user, event_priv from mysql.user where user='zabbix' (为Y表成功)



```

 





wk$^&6@.1s#.