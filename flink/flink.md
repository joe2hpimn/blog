# flink quick start

## 创建用户

```sh
useradd flink
echo -n "xdhSIS123" | passwd flink --stdin
```

## 下载安装包

```sh
su - flink
wget https://www.apache.org/dyn/closer.lua/flink/flink-1.9.1/flink-1.9.1-bin-scala_2.11.tgz
tar -zxf flink-1.9.1-bin-scala_2.11.tgz
```

## 修改配置参数

```sh
cd flink-1.9.1/conf
cat >masters <<EOF
ecs-xdh-0001:8081
EOF

cat >slaves <<EOF
ecs-xdh-0001
ecs-xdh-0002
ecs-xdh-0004
EOF

vi flink-conf.yaml
jobmanager.rpc.address: ecs-xdh-0001
jobmanager.rpc.port: 6123
jobmanager.heap.size: 4096m
taskmanager.heap.size: 4096m
taskmanager.numberOfTaskSlots: 2
parallelism.default: 6
jobmanager.execution.failover-strategy: region
rest.port: 20181
```

## 分发软件

```sh
scp -r flink-1.9.1 ecs-xdh-0002:~/
scp -r flink-1.9.1 ecs-xdh-0004:~/
```

## 启动集群

```sh
bin/start-cluster.sh
jps
```

## flink sql

### 下载各种依赖包

>下载以下依赖 jar 包，并拷贝到 flink-1.9.1/lib/ 目录下。因为我们运行时需要依赖各个 connector 实现。

```sh
# flink-sql-connector-kafka_2.11-1.9.1.jar
wget http://central.maven.org/maven2/org/apache/flink/flink-sql-connector-kafka_2.11/1.9.1/flink-sql-connector-kafka_2.11-1.9.1.jar
# flink-json-1.9.1-sql-jar.jar
wget http://central.maven.org/maven2/org/apache/flink/flink-json/1.9.1/flink-json-1.9.1-sql-jar.jar
# flink-jdbc_2.11-1.9.1.jar
wget http://central.maven.org/maven2/org/apache/flink/flink-jdbc_2.11/1.9.1/flink-jdbc_2.11-1.9.1.jar
# mysql-connector-java-5.1.48.jar
https://dev.mysql.com/downloads/connector/j/5.1.html
```

### 安装MySQL

```sh
docker run --name mysqldb -p 20133:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
```

### 演示代码

>flink-sql-submit

### 编译

```sh
cd flink-sql-submit
mvn clean package -Dmaven.test.skip
```

### 修改参数

```sh
vi env.sh
FLINK_DIR=/home/flink/flink-1.9.1_2.11
KAFKA_DIR=/data/pgkafka/confluent-5.0.0

vi source-generator.sh 
#!/usr/bin/env bash
################################################################################
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
################################################################################

source "$(dirname "$0")"/kafka-common.sh

# prepare Kafka
echo "Generating sources..."

create_kafka_topic 1 1 user_behavior
java -cp target/flink-sql-submit.jar com.github.wuchong.sqlsubmit.SourceGenerator 1000 | $KAFKA_DIR/bin/kafka-console-producer --broker-list 192.168.1.25:9094 --topic userbehavior --producer-property security.protocol=SASL_PLAINTEXT --producer-property sasl.mechanism=PLAIN


```

### 修改flink-sql-submit/src/main/resources/q1.sql

```sql
CREATE TABLE user_log
(
    user_id VARCHAR,
    item_id VARCHAR,
    category_id VARCHAR,
    behavior VARCHAR,
    ts TIMESTAMP
) WITH
(
    'connector.type' = 'kafka', -- 使用 kafka connector
    'connector.version' = 'universal',  -- kafka 版本，universal 支持 0.11 以上的版本
    'connector.topic' = 'user_behavior',  -- kafka topic
    'connector.startup-mode' = 'latest-offset', -- 从最新offset开始读取
    'connector.properties.0.key' = 'zookeeper.connect',  -- 连接信息
    'connector.properties.0.value' = '192.168.1.25:2182,192.168.1.112:2182,192.168.1.220:2182',
    'connector.properties.1.key' = 'bootstrap.servers',
    'connector.properties.1.value' = '192.168.1.25:9094,192.168.1.112:9094,192.168.1.220:9094',
    'connector.properties.2.key' = 'security.protocol',
    'connector.properties.2.value' = 'SASL_PLAINTEXT',
    'connector.properties.3.key' = 'sasl.mechanism',
    'connector.properties.3.value' = 'PLAIN',
    'connector.properties.4.key' = 'sasl.jaas.config', -- 只能在此配置
    'connector.properties.4.value' = 'org.apache.kafka.common.security.plain.PlainLoginModule required username="test" password ="123456";',
    'update-mode' = 'append',
    'format.type' = 'json',  -- 数据源格式为 json
    'format.derive-schema' = 'true' -- 从 DDL schema 确定 json 解析规则
);
```

### 提交 SQL 任务

>1.在 flink-sql-submit 目录下运行 ./source-generator.sh，会自动创建 user_behavior topic，并实时往里灌入数据。
>2.在 flink-sql-submit 目录下运行 ./run.sh q1， 提交成功后，可以在 Web UI 中看到拓扑。
