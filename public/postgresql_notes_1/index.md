# PostgreSQL学习笔记1--PostgreSQL安装


## 环境准备

```bash
[fan@MiWiFi-R1CM-srv ~]$ cat /etc/redhat-release 
CentOS Linux release 7.3.1611 (Core)
```



## 下载安装包

https://www.postgresql.org/ftp/source/

```bash
wget https://ftp.postgresql.org/pub/source/v9.6.2/postgresql-9.6.2.tar.gz
```



## 安装PostgreSQL



### 安装依赖包

```bash
yum -y install coreutils glib2 lrzsz mpstat dstat sysstat e4fsprogs xfsprogs ntp readline-devel zlib-devel openssl-devel pam-devel libxml2-devel libxslt-devel python-devel tcl-devel gcc make smartmontools flex bison perl-devel perl-ExtUtils* openldap-devel jadetex  openjade bzip2 
```



### 编译安装PostgreSQL

```bash
tar xf postgresql-9.6.2.tar.gz 
cd postgresql-9.6.2/
./configure --prefix=/home/fan/pgsql9.6
make world -j 8 
make install-world 
```

执行`world -j 8`的时候看到如下内容，则表示成功：

```bash
make[2]: Leaving directory `/home/fan/postgresql-9.6.2/contrib/pgcrypto'
make[1]: Leaving directory `/home/fan/postgresql-9.6.2/contrib'
PostgreSQL, contrib, and documentation successfully made. Ready to install.
```

执行`make install-wold`的时候看到如下内容，表示成功：

```bash
/usr/bin/install -c -m 644 ./unaccent.rules '/home/fan/pgsql9.6/share/tsearch_data/'
make[2]: Leaving directory `/home/fan/postgresql-9.6.2/contrib/unaccent'
make -C vacuumlo install
make[2]: Entering directory `/home/fan/postgresql-9.6.2/contrib/vacuumlo'
/usr/bin/mkdir -p '/home/fan/pgsql9.6/bin'
/usr/bin/install -c  vacuumlo '/home/fan/pgsql9.6/bin'
make[2]: Leaving directory `/home/fan/postgresql-9.6.2/contrib/vacuumlo'
make[1]: Leaving directory `/home/fan/postgresql-9.6.2/contrib'
PostgreSQL, contrib, and documentation installation complete
```



## 配置linux用户环境变量

将如下内容追加到`~/.bash_profile`

```bash
export PS1="$USER@`/bin/hostname -s`-> "  
export PGPORT=1921  
export PGDATA=/home/fan/pgdata  
export LANG=en_US.utf8  
export PGHOME=/home/digoal/pgsql9.6  
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH  
export PATH=$PGHOME/bin:$PATH:.  
export DATE=`date +"%Y%m%d%H%M"`  
export MANPATH=$PGHOME/share/man:$MANPATH  
export PGHOST=$PGDATA  
export PGUSER=postgres  
export PGDATABASE=postgres  
alias rm='rm -i'  
alias ll='ls -lh'  
unalias vi  
```

执行`source ~/.bash_profile`使配置生效



## 初始化数据库集群

命令： `initdb -D $PGDATA -E UTF8 --locale=C -U postgres `

```bash
fan@MiWiFi-R1CM-srv-> initdb -D $PGDATA -E UTF8 --locale=C -U postgres 
The files belonging to this database system will be owned by user "fan".
This user must also own the server process.

The database cluster will be initialized with locale "C".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory /home/fan/pgdata ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

WARNING: enabling "trust" authentication for local connections
You can change this by editing pg_hba.conf or using the option -A, or
--auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    pg_ctl -D /home/fan/pgdata -l logfile start

fan@MiWiFi-R1CM-srv-> 
```

## 配置数据库

配置文件在$PGDATA目录中

### 配置postgresql.conf

将如下内容追加到postgresql.conf中

```bash
listen_addresses = '0.0.0.0'  
port = 1921  
max_connections = 200  
unix_socket_directories = '.'  
tcp_keepalives_idle = 60  
tcp_keepalives_interval = 10  
tcp_keepalives_count = 10  
shared_buffers = 512MB  
dynamic_shared_memory_type = posix  
vacuum_cost_delay = 0  
bgwriter_delay = 10ms  
bgwriter_lru_maxpages = 1000  
bgwriter_lru_multiplier = 10.0  
bgwriter_flush_after = 0   
old_snapshot_threshold = -1  
backend_flush_after = 0   
wal_level = replica  
synchronous_commit = off  
full_page_writes = on  
wal_buffers = 16MB  
wal_writer_delay = 10ms  
wal_writer_flush_after = 0   
checkpoint_timeout = 30min   
max_wal_size = 2GB  
min_wal_size = 128MB  
checkpoint_completion_target = 0.05    
checkpoint_flush_after = 0    
random_page_cost = 1.3   
log_destination = 'csvlog'  
logging_collector = on  
log_truncate_on_rotation = on  
log_checkpoints = on  
log_connections = on  
log_disconnections = on  
log_error_verbosity = verbose  
autovacuum = on  
log_autovacuum_min_duration = 0  
autovacuum_naptime = 20s  
autovacuum_vacuum_scale_factor = 0.05  
autovacuum_freeze_max_age = 1500000000  
autovacuum_multixact_freeze_max_age = 1600000000  
autovacuum_vacuum_cost_delay = 0  
vacuum_freeze_table_age = 1400000000  
vacuum_multixact_freeze_table_age = 1500000000  
datestyle = 'iso, mdy'  
timezone = 'PRC'  
lc_messages = 'C'  
lc_monetary = 'C'  
lc_numeric = 'C'  
lc_time = 'C'  
default_text_search_config = 'pg_catalog.english'  
shared_preload_libraries='pg_stat_statements'  
```



### 配置pg_hba.conf

将如下内容追加到pg_hgb.conf中

```
host all all 0.0.0.0/0 md5  
```



## 启动数据库集群

命令：`pg_ctl start `

```
fan@MiWiFi-R1CM-srv-> pg_ctl start 
server starting
fan@MiWiFi-R1CM-srv-> LOG:  00000: redirecting log output to logging collector process
HINT:  Future log output will appear in directory "pg_log".
LOCATION:  SysLogger_Start, syslogger.c:622

fan@MiWiFi-R1CM-srv-> 

```



## 连接数据库

```
fan@MiWiFi-R1CM-srv-> psql
psql (9.6.2)
Type "help" for help.

postgres=#
```



## 简单使用



### 查看psql帮助

```
postgres=# \?
General
  \copyright             show PostgreSQL usage and distribution terms
  \errverbose            show most recent error message at maximum verbosity
  \g [FILE] or ;         execute query (and send results to file or |pipe)
  \gexec                 execute query, then execute each value in its result
  \gset [PREFIX]         execute query and store results in psql variables
  \q                     quit psql
  \crosstabview [COLUMNS] execute query and display results in crosstab
  \watch [SEC]           execute query every SEC seconds
......
```







### 使用psql 命令行连接数据库

psql  -h IP地址 -p端口 -U 用户名  数据库名

```bash
fan@MiWiFi-R1CM-srv-> psql -h 127.0.0.1 -p 1921 -U postgres postgres
psql (9.6.2)
Type "help" for help.

postgres=# 
```



###  新增用户

新建用户属于数据库操作，先使用psql和超级用户postgres连接到数据库。

新增一个普通用户

```bash
postgres=# create role fan login encrypted password '123456';
CREATE ROLE
```

新增一个超级用户

```bash
postgres=# create role dba_fan login superuser encrypted password '123456';
CREATE ROLE
```

新增一个流复制用户

```bash
postgres=# create role fan_rep replication login encrypted password '123456';
CREATE ROLE
```

将一个用户在不同角色之间切换

例如将fan设置为超级用户

```bash
ostgres=# alter role fan superuser;
ALTER ROLE
```

### 查看已有用户

```bash
postgres=# \du+
                                          List of roles
 Role name |                         Attributes                         | Member of | Description 
-----------+------------------------------------------------------------+-----------+-------------
 dba_fan   | Superuser                                                  | {}        | 
 fan       | Superuser                                                  | {}        | 
 fan_rep   | Replication                                                | {}        | 
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}        | 

postgres=# 

```

### 查看当前配置

show 参数名

```bash
postgres=# show client_encoding;
 client_encoding 
-----------------
 UTF8
(1 row)

```

查看pg_settings

```bash
postgres=# select * from pg_settings;
```




