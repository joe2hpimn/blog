# Linux下Postgres-XL编译安装  
## 安装依赖包
```
yum -y install lrzsz sysstat e4fsprogs ntp readline-devel zlib zlib-devel openssl openssl-devel pam-devel libxml2-devel libxslt-devel python-devel tcl-devel gcc make smartmontools flex bison perl perl-devel perl-ExtUtils* OpenIPMI-tools openldap openldap-devel
```
## Postgres-XL编译
```
wget http://downloads.sourceforge.net/project/postgres-xl/Releases/Version_9.2rc/postgres-xl-v9.2-src.tar.gz?r=http%3A%2F%2Fsourceforge.net%2Fprojects%2Fpostgres-xl%2F&ts=1406180225&use_mirror=jaist     
./configure --prefix=/opt/pgxl9.2 --with-pgport=11921 --with-perl --with-tcl --with-python --with-openssl --with-pam --without-ldap --with-libxml --with-libxslt --enable-thread-safety --enable-dtrace --enable-debug --enable-cassert  
gmake world   
gmake world install
```
## 系统配置
### 时钟
```
crontab -e
-- 8 * * * * /usr/sbin/ntpdate asia.pool.ntp.org && /sbin/hwclock --systohc
/usr/sbin/ntpdate asia.pool.ntp.org && /sbin/hwclock --systohc
```

### 时区
```
vi /etc/sysconfig/clock 
  -- ZONE="Asia/Shanghai"
     UTC=false
     ARC=false

rm /etc/localtime 
cp /usr/share/zoneinfo/PRC /etc/localtime
``` 
### 编码
```
vi /etc/sysconfig/i18n
  -- LANG="en_US.UTF-8"
```

### ssh配置
```
vi /etc/ssh/sshd_config
UseDNS no
PubkeyAuthentication no

vi /etc/ssh/ssh_config
GSSAPIAuthentication no
```

### 内核参数
```
vi /etc/sysctl.conf
kernel.shmmni = 4096
kernel.sem = 50100 64128000 50100 1280
fs.file-max = 7672460
net.ipv4.ip_local_port_range = 9000 65000
net.core.rmem_default = 1048576
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.core.netdev_max_backlog = 10000
vm.overcommit_memory = 0
net.ipv4.ip_conntrack_max = 655360
fs.aio-max-nr = 1048576
net.ipv4.tcp_timestamps = 0
```

### 限制
```
vi /etc/security/limits.conf
* soft    nofile  131072
* hard    nofile  131072
* soft    nproc   131072
* hard    nproc   131072
* soft    core    unlimited
* hard    core    unlimited
* soft    memlock 50000000
* hard    memlock 50000000

vi /etc/security/limits.d/90-nproc.conf 
# 注释所有其他, 并添加
* soft    nproc   131072
* hard    nproc   131072
```

### 配置selinux
```
vi /etc/sysconfig/selinux
SELINUX=disabled

vi /etc/sysconfig/iptables
# 允许各节点相互通信
```
## 创建pgxl用户与数据目录
```
useradd pgxl
# 也可以是其他用户名，按需配置
```
### coordinator节点    
```
mkdir -p /home/dbadmin/pgxl/c11921   
chown -R pgxl:pgxl /home/dbadmin/pgxl
```
### gtm 节点 
```
mkdir -p /home/dbadmin/pgxl/g11926
# 一般情况下不需要配置gtm-proxy节点，除非考虑到高负载的情况
```
## 环境变量
```
export PGDATA=/home/dbadmin/pgxl/c11921/pg_root   
export LANG=en_US.utf8    
export PGHOME=/opt/pgxl9.2    
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH    
export DATE=\`date +"%Y%m%d%H%M"\`    
export PATH=$PGHOME/bin:$PATH:.    
export MANPATH=$PGHOME/share/man:$MANPATH   
export PGHOST=$PGDATA   
export PGPORT=11921    
export PGUSER=postgres    
export PGDATABASE=pg_tds
```

## 节点初始化
```
initdb -D /home/dbadmin/pgxl/c11921/pg_root --nodename=c11921 -E UTF8 --locale=C -U postgres -W     
initgtm -Z gtm -D /home/dbadmin/pgxl/g11926
```

#### datanode节点与其他节点
```
mkdir -p /home/dbadmin/pgxl/d11922  
mkdir -p /home/dbadmin/pgxl/d11923  
mkdir -p /home/dbadmin/pgxl/d11924   
chown -R pgxl:pgxl /home/dbadmin/pgxl

initdb -D /home/dbadmin/pgxl/d11922/pg_root --nodename=d11922 -E UTF8 --locale=C -U postgres -W   
initdb -D /home/dbadmin/pgxl/d11923/pg_root --nodename=d11923 -E UTF8 --locale=C -U postgres -W   
initdb -D /home/dbadmin/pgxl/d11924/pg_root --nodename=d11924 -E UTF8 --locale=C -U postgres -W

host all all 10.248.112.106/32 trust   
host all all 10.248.112.105/32 trust   
host all all 10.248.112.104/32 trust   
host all all 10.248.112.103/32 trust
```
## 启动集群
```
1:gtm_ctl -Z gtm start -D /home/dbadmin/pgxl/g11926   
2:pg_ctl start -Z datanode -D /home/dbadmin/pgxl/d11922/pg_root   
3:pg_ctl start -Z datanode -D /home/dbadmin/pgxl/d11923/pg_root   
4:pg_ctl start -Z datanode -D /home/dbadmin/pgxl/d11924/pg_root   
5:pg_ctl start -Z coordinator -D /home/dbadmin/pgxl/c11921/pg_root    
注意：需要严格安装上述顺序来启动集群！

create node c11921 with (type=coordinator, host='10.248.112.106', port=11921);    
create node d11922 with (type=datanode, host='10.248.112.103', port=11922, primary=true);    
create node d11923 with (type=datanode, host='10.248.112.104', port=11923, primary=false);   
create node d11924 with (type=datanode, host='10.248.112.105', port=11924, primary=false);

alter node c11921 with (type=coordinator, host='10.248.112.106', port=11921);   
alter node d11922 with (type=datanode, host='10.248.112.103', port=11922,primary=true);    
alter node d11923 with (type=datanode, host='10.248.112.104', port=11923);   
alter node d11924 with (type=datanode, host='10.248.112.105', port=11924);   
create node group gp1 with (d11922, d11923, d11924);

SELECT pgxc_pool_reload();    
create table t1(id serial8 primary key, info text, crt_time timestamp) distribute by hash(id) to group gp1;
```
## 关闭集群
```
pg_ctl stop -m fast -Z coordinator -D /home/dbadmin/pgxl/c11921/pg_root  
pg_ctl stop -m fast -Z datanode -D /home/dbadmin/pgxl/d11922/pg_root  
pg_ctl stop -m fast -Z datanode -D /home/dbadmin/pgxl/d11923/pg_root   
pg_ctl stop -m fast -Z datanode -D /home/dbadmin/pgxl/d11924/pg_root   
gtm_ctl stop -m fast -Z gtm -D /home/dbadmin/pgxl/g11926
```
## postgresql.conf
```
listen_addresses = '0.0.0.0'                  # what IP address(es) to listen on;   
port = 11924                                  # (change requires restart)   
max_connections = 500                         # (change requires restart)   
superuser_reserved_connections = 13           # (change requires restart)   
unix_socket_directory = '.'                   # (change requires restart)   
unix_socket_permissions = 0700                # begin with 0 to use octal notation   
tcp_keepalives_idle = 60                      # TCP_KEEPIDLE, in seconds;   
tcp_keepalives_interval = 10                  # TCP_KEEPINTVL, in seconds;   
tcp_keepalives_count = 10                               # TCP_KEEPCNT;   
shared_buffers = 2048MB                                 # min 128kB   
max_prepared_transactions = 500                         # zero disables the feature   
shared_preload_libraries = 'pg_stat_statements'         # (change requires restart)   
vacuum_cost_delay = 10ms                                # 0-100 milliseconds   
vacuum_cost_limit = 10000                               # 1-10000 credits   
bgwriter_delay = 10ms                   # 10-10000ms between rounds   
shared_queues = 64                      # min 16      
shared_queue_size = 262144               # min 16KB   
wal_level = hot_standby                 # minimal, archive, or hot_standby   
synchronous_commit = off                # synchronization level;   
wal_sync_method = fdatasync             # the default is the first option   
wal_buffers = 16384kB                   # min 32kB, -1 sets based on shared_buffers   
wal_writer_delay = 10ms         		# 1-10000 milliseconds   
checkpoint_segments = 128               # in logfile segments, min 1, 16MB each   
archive_mode = on               		# allows archiving to be done   
archive_command = '/bin/date'           # command to use to archive a logfile segment   
max_wal_senders = 32            		# max number of walsender processes    
hot_standby = on                        # "on" allows queries during recovery   
max_standby_archive_delay = 300s        # max delay before canceling queries  
max_standby_streaming_delay = 300s      # max delay before canceling queries   
wal_receiver_status_interval = 1s       # send replies at least this often   
hot_standby_feedback = on               # send info from standby to prevent   
remote_query_cost = 100.0               # same scale as above   
effective_cache_size = 96000MB   
log_destination = 'csvlog'              # Valid values are combinations of   
logging_collector = on          		# Enable capturing of stderr and csvlog   
log_directory = 'pg_log'                # directory where log files are written,  
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # log file name pattern,   
log_file_mode = 0600                    # creation mode for log files,   
log_truncate_on_rotation = on           # If on, an existing log file with the   
log_min_duration_statement = 1s 		# -1 is disabled, 0 logs all statements   
log_checkpoints = on   
log_connections = on   
log_disconnections = on   
log_error_verbosity = verbose           # terse, default, or verbose messages   
log_lock_waits = on                     # log lock waits >= deadlock_timeout   
log_statement = 'ddl'                   # none, ddl, mod, all   
log_timezone = 'PRC'   
autovacuum = on                 		# Enable autovacuum subprocess?  'on'    
log_autovacuum_min_duration = 0 		# -1 disables, 0 logs all actions and    
autovacuum_vacuum_cost_delay = 10ms     # default vacuum cost delay for   
datestyle = 'iso, mdy'   
timezone = 'PRC'   
lc_messages = 'C'                       # locale for system error message   
lc_monetary = 'C'                       # locale for monetary formatting   
lc_numeric = 'C'                        # locale for number formatting   
lc_time = 'C'                           # locale for time formatting    
default_text_search_config = 'pg_catalog.english'   
pooler_port = 21924                     # Pool Manager TCP port    
max_pool_size = 100                     # Maximum pool size   
pool_conn_keepalive = 60                # Close connections if they are idle   
pool_maintenance_timeout = 30           # Launch maintenance routine if pooler   
max_coordinators = 16                   # Maximum number of Coordinators    
max_datanodes = 16                      # Maximum number of Datanodes   
gtm_host = '10.248.112.106'             # Host name or address of GTM   
gtm_port = 11926                        # Port of GTM   
pgxc_node_name = 'd11924'               # Coordinator or Datanode name  
```
