# 安装配置支持SASL认证的Kafka集群

## 1 服务器  

`Hardware:华为云ECS 16vCPUS 64GB`  
`OS:CentOS Linux release 7.4.1708`

序号|IP
---|---
1|192.168.1.220
2|192.168.1.176
3|192.168.1.194
4|192.168.1.231
5|192.168.1.25

---

## 2 Java环境

`下载jdk-8u191-linux-x64.tar.gz,解压安装`

---

## 3 安装Kafka

---

### 3.1 下载confluent-kafka

`下载confluent-oss-5.0.0-2.11.tar.gz`  
`https://www.confluent.io/download/`  

---

### 3.2 修改配置文件

**1.解压安装包**  
`将下载的安装包confluent-oss-5.0.0-2.11.tar.gz拷贝到其中一台主机的/data文件夹下，接下来以/data目录为例:`  

```sh
cd /data
tar -zxvf confluent-oss-5.0.0-2.11.tar.gz
```

---
**2创建软连接**  
`Tips：没什么用`

```sh
ln –s /data/confluent-5.0.0 /data/confluent
```

---

**3.修改server.properties**  
`Tips:ZK需要自行安装或直接用confluent自带的ZK，具体安装细节略`

```sh
vi /data/confluent/etc/kafka/server.properties
# kafka日志路径，需要先创建好路径
log.dirs=/data/kafka-logs
zookeeper.connect=localhost:2181
# zookeeper地址，逗号隔开
```

---
**4.修改内存设置**  
`Tips:Java堆内存大小需根据机器的配置而定`

```sh
vi /data/confluent/bin/kafka-server-start
# 两个参数都修改为8G即可
KAFKA_HEAP_OPTS="-Xmx8G -Xms8G"
vi /data/confluent/bin/kafka-run-class  
# 这里需要修改成2048M
KAFKA_HEAP_OPTS="-Xmx2048M"
```

---
**5.修改schema-registry.properties**  
`Tips:启动schema-registry以支持Avro格式kafka消息的发送和接收`

```sh
vi /data/confluent/etc/schema-registry/schema-registry.properties
# ZK地址需要根据实际情况修改
kafkastore.connection.url=localhost:2181
# 如果8081端口被占用，还需要修改
listeners=http://0.0.0.0:8581
# 增加SASL/PLAIN认证配置
kafkastore.security.protocol=SASL_PLAINTEXT
kafkastore.sasl.mechanism=PLAIN
# 避免新schema保持增加如下设置
avro.compatibility.level=none
```

---
**6.修改connect-avro-distributed.properties**  
`Tips:producer与consumer都要进行sasl配置`

```sh
vi /data/confluent/etc/schema-registry/connect-avro-distributed.properties
# kafka集群地址，逗号隔开
bootstrap.servers=192.168.1.220:9094,192.168.1.176:9094,192.168.1.194:9094
# 增加如下配置：
offset.flush.interval.ms=1000
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
producer.security.protocol=SASL_PLAINTEXT
producer.sasl.mechanism=PLAIN
consumer.security.protocol=SASL_PLAINTEXT
consumer.sasl.mechanism=PLAIN
# 如果刚才修改了schema-registry的listeners端口，还需要修改如下两个配置的端口
key.converter.schema.registry.url=http://localhost:8581
value.converter.schema.registry.url=http://localhost:8581
```

---
**7.修改schema-registry-run-class**  
`Tips:kafka_client_jaas.conf路径是前面建的client jaas文件路径`

```sh
vi /data/confluent/bin/schema-registry-run-class
# #Launch mode之前增加如下代码：  
JAAS_CONFIG="-Djava.security.auth.login.config=/opt/sasl/kafka_client_jaas.conf" 
# #Launch mode下面的if else代码修改成如下：
if [ "x$DAEMON_MODE" = "xtrue" ]; then
  nohup $JAVA $SCHEMA_REGISTRY_HEAP_OPTS $SCHEMA_REGISTRY_JVM_PERFORMANCE_OPTS $SCHEMA_REGISTRY_JMX_OPTS $SCHEMA_REGISTRY_LOG4J_OPTS $JAAS_CONFIG -cp $CLASSPATH $SCHEMA_REGISTRY_OPTS "$MAIN" "$@" 2>&1 < /dev/null &
else
  exec $JAVA $SCHEMA_REGISTRY_HEAP_OPTS $SCHEMA_REGISTRY_JVM_PERFORMANCE_OPTS $SCHEMA_REGISTRY_JMX_OPTS $SCHEMA_REGISTRY_LOG4J_OPTS $JAAS_CONFIG -cp $CLASSPATH $SCHEMA_REGISTRY_OPTS "$MAIN" "$@"
fi
```

---
**8.修改connect-distributed**  
`Tips:Java堆内存请跟进实际情况调整`

```sh
vi /data/confluent/bin/connect-distributed
# 在倒数第二行新增如下代码：
export KAFKA_HEAP_OPTS="-Xmx8G –Xms8G"
export JAAS_CONFIG="-Djava.security.auth.login.config=/opt/sasl/kafka_client_jaas.conf"
```

### 3.3 配置SASL/PLAIN安全认证

---
1.在每台kafka服务器上都新建kafka_server_jaas.conf文件，写入如下内容：

```java
KafkaServer {
   org.apache.kafka.common.security.plain.PlainLoginModule required
   username="admin"
   password="123456"
   user_admin="123456"
   user_test="123456";
};
```  

由于该文件中包含明文密码，所以请注意存放目录，本例存放的目录为：/data/sasl。新增kafka_client_jaas_conf文件，客户端访问服务器使用。内容如下：

```java
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="test"
  password="123456";
};
```

---
2.修改/data/confluent/bin目录下的kafka-server-start文件，在倒数第二行加上如下代码：

```sh
export JAAS_CONFIG="-Djava.security.auth.login.config=/data/sasl/kafka_server_jaas.conf"
```

---
3.修改/data/confluent/bin目录下的kafka-run-class文件，倒数第四行修改成：

```sh
nohup $JAVA $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS -cp $CLASSPATH $KAFKA_OPTS $JAAS_CONFIG "$@" > "$CONSOLE_OUTPUT_FILE" 2>&1 < /dev/null &  
```

倒数第二行修改成：  

```sh
exec $JAVA $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_OPTS -cp $CLASSPATH $KAFKA_OPTS $JAAS_CONFIG "$@"
```

---
4.修改server.properties  

```sh
vi confluen/etc/kafka/server.properties
# 修改如下配置  
listeners=SASL_PLAINTEXT://192.168.1.220:9094  
advertised.listeners=SASL_PLAINTEXT://192.168.1.220:9094  
num.network.threads=12
num.io.threads=32
num.recovery.threads.per.data.dir=1  
# 同时新增如下内容
security.inter.broker.protocol=SASL_PLAINTEXT  
sasl.enabled.mechanisms=PLAIN  
sasl.mechanism.inter.broker.protocol=PLAIN
default.replication.factor=3
unclean.leader.election.enable=false
num.replica.fetchers=15
```

### 3.4 启动程序并初始化

1.使用scp命令分发程序包到各个节点

```sh
scp –r /data/confluent-5.0.0 user@ip:/data/
```

2.修改其他节点的server.properties  

```sh
vi /opt/confluent/etc/kafka/server.properties
#从0开始，按主机依次递增
broker.id=1  
```

3.各个节点启动kafka

```sh
cd /data/confluent/bin
nohup ./kafka-server-start ../etc/kafka/server.properties >kafka.out 2>&1 &
jps
```

看到**SupportedKafka**即启动成功

4.在其中一台主机创建kafka connect需要的主题

```sh
cd /data/confluent/bin
./kafka-topics --create --zookeeper localhost:2181 –topic connect-configs --replication-factor 3 --partitions 1
./kafka-topics --create --zookeeper localhost:2181 –topic connect-offsets --replication-factor 3 --partitions 50
./kafka-topics --create --zookeeper localhost:2181 –topic connect-statuses --replication-factor 3 --partitions 25
```

5.各个节点启动schema registry

```sh
cd /data/confluent/bin
nohup ./schema-registry-start ../etc/schema-registry/schema-registry.properties >sr.out 2>&1 &
```

查看进程发现**SchemaRegistryMain**进程即启动成功

6.各个节点启动distributed connect  

```sh
cd /data/confluent/bin
nohup ./connect-distributed ../etc/schema-registry/connect-avro-distributed.properties >connect.out 2>&1 &
```

查看进程发现**ConnectDistributed**进程既启动成功

### 3.5 验证kafka是否安装成功

1.新建测试主题

```sh
cd /data/confluent/bin
./kafka-topics --create --zookeeper 192.168.1.220:2181 --replication-factor 1 --partitions 3 --topic test
```

2.启动消费者：

```sh
cd /data/confluent/bin
./kafka-console-consumer --bootstrap-server 192.168.1.220:9094 --topic test --from-beginning --consumer-property security.protocol=SASL_PLAINTEXT --consumer-property sasl.mechanism=PLAIN
# 注意需要修改kafka-console-consumer脚本，在倒数第二行增加
if [ "x$KAFKA_OPTS" ]; then
 export KAFKA_OPTS="-Djava.security.auth.login.config=/data/sasl/kafka_client_jaas.conf"
fi
```

3、在另一个终端启动生产者

```sh
cd /data/confluent/bin
./kafka-console-producer --broker-list 192.168.1.220:9094 --topic test --producer-property security.protocol=SASL_PLAINTEXT --producer-property sasl.mechanism=PLAIN
# 注意需要修改kafka-console-producer脚本，在倒数第二行增加
if [ "x$KAFKA_OPTS" ]; then
 export KAFKA_OPTS="-Djava.security.auth.login.config=/data/sasl/kafka_client_jaas.conf"
fi
```

在生产者的终端输入数据后回车，如果在消费者终端看到输出了刚刚生产的数据则kafka安装成功。
