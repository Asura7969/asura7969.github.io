---
title: kafka && SCRAM + ACL配置
date: 2022-02-17 21:04:36
tags: kafka
categories: kafka
cover: /img/topimg/202111111633.png
---
## 环境
* kafka-2.6.2_2.13
* zookeeper-3.5.9
* OS-Centos7
* java:1.8.0_201
> kafka-broker 与 zk节点混部

## 配置
### Zookeeper
#### **zoo.cfg**
```properties
tickTime=2000
dataDir=/data/zk/data
dataLogDir=/data/zk/logs
maxClientCnxns=50
minSessionTimeout=60000
maxSessionTimeout=120000
clientPort=2181
syncLimit=5
initLimit=10
autopurge.snapRetainCount=20
autopurge.purgeInterval=2
4lw.commands.whitelist=*
server.1=node1:2888:3888
server.2=node2:2888:3888
server.3=node3:2888:3888


authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
jaasLoginRenew=3600000
# 打开sasl开关, 默认是关的
quorum.auth.enableSasl=true
# ZK做为leaner的时候, 会发送认证信息
quorum.auth.learnerRequireSasl=true
# 设置为true的时候,learner连接的时候需要发送认证信息,否则拒绝
quorum.auth.serverRequireSasl=true
# JAAS 配置里面的 Context 名字
quorum.auth.learner.loginContext=QuorumLearner
# JAAS 配置里面的 Context 名字
quorum.auth.server.loginContext=QuorumServer
# 建议设置成ZK节点的数量乘2
quorum.cnxn.threads.size=6
```
> 参考 [zookeeper wiki](https://cwiki.apache.org/confluence/display/ZOOKEEPER/ZooKeeper+and+SASL)


#### **zookeeper_jaas.conf**
```conf
# QuorumServer 和 QuorumLearner 都是配置的ZK节点之间的认证配置,
# 一般称为 Server-to-Server authentication, 并不影响 Kafka 的连接认证.
# Server 是配置的Kafka连接需要的认证信息, 称为 Client-to-Server authentication
Server {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    user_super=super # zookeeper超级管理员
    username=admin # zookeeper之间的认证用户名
    password=admin # zookeeper之间的认证密码
    user_kafka=admin # 为kafka服务创建账号密码：用户名kafka，密码admin
    user_producer=admin; # 根据实际情况增加用户，这里增加一个用户名为producer,密码为admin的用户
};
Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="super"
    password="super";
};
QuorumServer {
  org.apache.zookeeper.server.auth.DigestLoginModule required
    user_zookeeper="zookeeper"; # 用户名为zookeeper，密码为zookeeper
};
QuorumLearner {
  org.apache.zookeeper.server.auth.DigestLoginModule required
    username="zookeeper"
    password="zookeeper";
};
```

#### 添加zk环境变量
> zkEnv.sh 中添加
```shell
export SERVER_JVMFLAGS="-Djava.security.auth.login.config=/workspace/zookeeper/latest/conf/zookeeper_jaas.conf $SERVER_JVMFLAGS"


export CLIENT_JVMFLAGS="-Djava.security.auth.login.config=/workspace/zookeeper/latest/conf/zookeeper_jaas.conf $CLIENT_JVMFLAGS"
```


### Kafka
#### **server.properties**
```properties
auto.create.topics.enable=true
broker.id=1
default.replication.factor=2
delete.topic.enable=true
log.dirs=/data/kafka-data
log.message.timestamp.type=LogAppendTime
num.io.threads=2
num.network.threads=2
num.partitions=2
num.recovery.threads.per.data.dir=2
num.replica.fetchers=4
log.cleaner.threads=2
offsets.topic.replication.factor=2
transaction.state.log.min.isr=2
transaction.state.log.replication.factor=3
unclean.leader.election.enable=true
zookeeper.connect=127.0.0.1:2181,127.0.0.2:2181,127.0.0.3:2181


listener.security.protocol.map=INTERNAL:SASL_PLAINTEXT,EXTERNAL:SASL_PLAINTEXT
listeners=INTERNAL://127.0.0.1:9092,EXTERNAL://xxxx
advertised.listeners=INTERNAL://127.0.0.1:9092,EXTERNAL://xxxx
inter.broker.listener.name=INTERNAL


authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:admin
sasl.enabled.mechanisms=SCRAM-SHA-512
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
allow.everyone.if.no.acl.found=false
zookeeper.set.acl=true
```


#### **kafka_server_jaas.conf**
> 配置路径自定义
```conf
# KafkaServer 值 kafka broker内部通信
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    # username和password是broker用于初始化连接到其他的broker
    username="admin"
    password="admin-secret";
};
# kafka客户端连接broker的客户端认证配置，用于bin/*.sh下的各种kafkaTool连接kafka使用
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-secret";
};
# 这里是kafka客户端连接broker的用户名，对应 zookeeper_jaas.conf 文件中Server的配置：user_kafka=admin
Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="kafka"
    password="admin";
};
```


#### 添加kafka环境变量
> kafka-run-class.sh 中添加 kafka_server_jaas.conf 文件配置
```shell
export KAFKA_OPTS="-Djava.security.auth.login.config=/ops/kafka/latest/config/kafka_server_jaas.conf"
```


## 启动集群
1、先启动zookeeper集群
> bin/zkServer.sh


2、创建 kafka_init_acl_jass.conf 文件（用于连接zookeeper），并添加到 kafka-configs.sh 中
```shell
# kafka_init_acl_jass.conf 文件内容
Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="kafka"
    password="admin";
};


# 脚本中添加环境变量
export KAFKA_OPTS="-Djava.security.auth.login.config=/workspace/kafka/latest/bin/kafka_init_acl_jass.conf"
```


3、创建kafka broker之间的用户(kafka_server_jaas.conf 中 KafkaServer信息)SCRAM证书
```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin
创建一个测试用户
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=test-kafka],SCRAM-SHA-512=[password=test-kafka]' --entity-type users --entity-name test-kafka
```


4、启动kafka
```shell
source /etc/profile && cd /ops/kafka/latest && nohup bin/kafka-server-start.sh config/server.properties > /dev/null 2>&1 &
```

## 测试

### 读写数据
1、创建topic
```shell
bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 2 --partitions 3 --topic test
```
> 注：
> 1. 使用 --zookeeper 方式连接创建topic，环境变量需要加载 kafka_server_jaas.conf 文件中 Client 的配置才能创建成功
> 2. 使用 --bootstrap-server，需添加参数 --command-config ./conf.properties，并在 kafka-topics.sh 脚本中添加 jass_client.conf 文件路径
```properties
# 1、conf.properties 文件内容
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512


# 2、kafka-topics.sh 中添加
export KAFKA_OPTS="-Djava.security.auth.login.config=/workspace/kafka/latest/bin/jass_client.conf"


# 3、jass_client.conf 文件内容
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-secret";
};


# 4、创建topic
bin/kafka-topics.sh --create --bootstrap-server 127.0.0.1:9092 --command-config ./conf.properties --replication-factor 2 --partitions 3 --topic test
```

2、写数据
> 请先确保对应用户是否已经创建，本例中使用test-kafka用户

创建 producer.conf
```conf
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="test-kafka" password="test-kafka";
```
```shell
bin/kafka-console-producer.sh --topic test --bootstrap-server 127.0.0.1:9092 --producer.config producer.conf
```


数据数据后会报以下错误, 原因是还未添加 test-kafka 该用户对test的写权限
```log
[2021-11-11 15:07:28,160] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 3 : {test=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
[2021-11-11 15:07:28,164] ERROR [Producer clientId=console-producer] Topic authorization failed for topics [test] (org.apache.kafka.clients.Metadata)
[2021-11-11 15:07:28,166] ERROR Error when sending message to topic test with key: null, value: 1 bytes with error: (org.apache.kafka.clients.producer.internals.ErrorLoggingCallback)
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [test]
```
```shell
# 第一种：连接zookeeper创建
bin/kafka-acls.sh --authorizer kafka.security.authorizer.AclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:test-kafka --operation Write --topic test


或


# 第二种：使用KafkaAdmin创建（配置文件见上文）
bin/kafka-acls.sh --bootstrap-server 127.0.0.1:9092 --command-config ./conf.properties --add --allow-principal User:test-kafka --operation Write --topic test
```
```log
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)


Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)
```


3、消费数据


添加读权限
```shell
# 第一种：连接zookeeper创建
bin/kafka-acls.sh --authorizer kafka.security.authorizer.AclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:test-kafka --operation Read --topic test

或

# 第二种：使用KafkaAdmin创建（配置文件见上文）
bin/kafka-acls.sh --bootstrap-server 127.0.0.1:9092 --command-config ./conf.properties --add --allow-principal User:test-kafka --operation Read --topic test
```
```log
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)


Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)
   (principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)
```
添加消费组权限
```shell
# 第一种：连接zookeeper创建
bin/kafka-acls.sh --authorizer kafka.security.authorizer.AclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:test-kafka --operation Read --topic test --group test

或

# 第二种：使用KafkaAdmin创建（配置文件见上文）
bin/kafka-acls.sh --bootstrap-server 127.0.0.1:9092 --command-config ./conf.properties --add --allow-principal User:test-kafka --operation Read --topic test --group test
```
```log
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)


Adding ACLs for resource `ResourcePattern(resourceType=GROUP, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)


Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)


Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
   (principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)
   (principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)
```


消费
```shell
bin/kafka-console-consumer.sh --topic test --bootstrap-server 127.0.0.1:9092 --group test --consumer.config consumer.conf
```

### 动态配置用户
添加用户 SCRAM证书
> 注：动态添加用户SCRAM证书，只能使用--zookeeper创建，--bootstrap-server参数只支持创建Quota配置

```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=asura7969],SCRAM-SHA-512=[password=asura7969]' --entity-type users --entity-name asura7969
```

查看用户 SCRAM证书
```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --describe --entity-type users --entity-name asura7969
```

删除用户 SCRAM证书
```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --delete-config 'SCRAM-SHA-512' --entity-type users --entity-name asura7969

bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --delete-config 'SCRAM-SHA-256' --entity-type users --entity-name asura7969
```

### java 客户端配置
#### 生产者
```java
    @Test
    public void producer() throws Exception {


        Properties props = new Properties();
        String jaasTemplate = "org.apache.kafka.common.security.scram.ScramLoginModule required username=\"%s\" password=\"%s\";";
        String jaasCfg = String.format(jaasTemplate, "test-kafka", "test-kafka");
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BROKER_SEFVER);
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "DemoProducer");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put("security.protocol", "SASL_PLAINTEXT");
        props.put("sasl.mechanism", "SCRAM-SHA-512");
        props.put("sasl.jaas.config", jaasCfg);
        KafkaProducer<String, String> producer = new KafkaProducer<>(props);


        try {
            for (int i = 0; i < 10; i++) {
                Future<RecordMetadata> test = producer.send(new ProducerRecord<>("test", i + ""));
                RecordMetadata metadata = test.get();
                System.out.println(metadata.offset());
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            producer.close();
        }
        System.out.println("结束");
    }
```


#### 消费者
```java
    @Test
    public void consumer() {
        String jaasTemplate = "org.apache.kafka.common.security.scram.ScramLoginModule required username=\"%s\" password=\"%s\";";
        String jaasCfg = String.format(jaasTemplate, "test-kafka", "test-kafka");
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BROKER_SEFVER);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test");
        props.put("security.protocol", "SASL_PLAINTEXT");
        props.put("sasl.mechanism", "SCRAM-SHA-512");
        props.put("sasl.jaas.config", jaasCfg);
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singleton("test"));


        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
        Iterator<ConsumerRecord<String, String>> iterator = records.iterator();
        while (iterator.hasNext()) {
            ConsumerRecord<String, String> next = iterator.next();
            System.out.println(next.value());
        }
    }
```
