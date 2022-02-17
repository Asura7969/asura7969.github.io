---
title: kakafka升级-Security
date: 2022-02-17 21:11:12
tags: kafka
categories: kafka
cover: /img/topimg/202202172112.png
---
## 搭建

### 环境

* kafka-2.6.2_2.13
* zookeeper-3.5.9
* jdk-8u201 ( jdk1.8.0_201 )
* centos7

### Zookeeper

#### 开启SASL认证

`vim conf/zoo.cfg`，增加如下安全配置

```properties
...

authProvider.1=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl
# 授权登录时限
jaasLoginRenew=3600000
```

#### 创建JASS配置

`vim conf/zookeeper_sasl_jaas.conf`
```properties
# 配置账户信息
Server {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    user_super="super!@#" 		# 创建超级管理员，用户名：super，密码：super!@#
    user_kafka="kafka!@#"		  # 创建kafka服务专用账户：用户名：kafka，密码：kafka!@#
    user_public="public!@#";  # 创建一个专门的对外的开放账户用于接入zk服务，防止破坏kafka数据
};

# 本地bin/*.sh连接zk服务使用的客户端认证jass配置，使用超级账户，这里本地连接需要使用启动zk服务的用户或root
Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="super"
    password="super!@#";
};

```

注意：jass配置中，没个模块和用户信息后以 ";" 结束

#### 将JASS配置注入环境变量
`vim bin/zkEnv.sh`
```properties
# 启动服务时用
export SERVER_JVMFLAGS="-Djava.security.auth.login.config=/workspace/zookeeper/latest/conf/zookeeper_sasl_jaas.conf $SERVER_JVMFLAGS"

# 本地命令行创建客户端时用
export CLIENT_JVMFLAGS="-Djava.security.auth.login.config=/workspace/zookeeper/latest/conf/zookeeper_sasl_jaas.conf $CLIENT_JVMFLAGS"

#### 启动zk集群
如上步骤中的配置应用到每个节点后，可以启动zk级群
```shell
bin/zkServer.sh start
```

### Kafka
#### 开启sasl和acl

`vim config/server.properties`, 增加sasl配置和acl配置，同时增加2个超级用户：
* admin：用于kafka之间互连、内部运维
* public：用于外部平台服务接入，必要时可以将用户从zk中删除或修改其密码

A：开启sasl和acl配置：

```properties
zookeeper.connect=127.0.0.1:2181,127.0.0.2:2181,127.0.0.3:2181
...

listeners=SASL_PLAINTEXT://127.0.0.1:9092

# sasl
super.users=User:admin;User:public
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

# acl
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
zookeeper.set.acl=true
allow.everyone.if.no.acl.found=false
```


B：当启用kafka内外网分流时，应该这样配置：

```properties

zookeeper.connect=127.0.0.1:2181,127.0.0.2:2181,127.0.0.3:2181
...

listener.security.protocol.map=INTERNAL:SASL_PLAINTEXT,EXTERNAL:SASL_PLAINTEXT
listeners=INTERNAL://127.0.0.1:9092,EXTERNAL://公网IP:9093
advertised.listeners=INTERNAL://127.0.0.1:9092,EXTERNAL://xxxx
inter.broker.listener.name=INTERNAL

# sasl
super.users=User:admin;User:public
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

# acl
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
zookeeper.set.acl=true
allow.everyone.if.no.acl.found=false
```

#### 创建JASS配置
这里，将3个必要的jass配置模块写到一起，方便使用：

* `KafkaServer` 此模块为broker内部互连的认证配置
* `client` 此模块为broker连接zk的客户端认证配置
* `KafkaClient`此模块为kafka客户端连接kafka服务的客户端认证配置

`vim config/kafka_sasl_jaas.conf`
```conf
# broker内部通信认证配置，这里配置的用户必须为超级用户，见config/server.properties中的super.users配置
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-secret!@#";
};

# kafka连接zk的客户端认证配置，注意：这里的用户要与zk中已配置的用户对应，见zk的jass配置：conf/zookeeper_sasl_jaas.conf
Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="kafka"
    password="kafka!@#";
};

# kafka客户端连接broker的客户端认证配置，用于bin/*.sh下的各种kafkaTool连接kafka使用
KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="admin"
  password="admin-secret!@#";
};
```

#### 将JASS配置注入环境变量

注入到 kafka-run-class.sh 中即可，各个kafka tool都会调用这个shell
`vim bin/kafka-run-class.sh`
```shell
export KAFKA_OPTS="-Djava.security.auth.login.config=/workspace/kafka/latest/config/kafka_sasl_jaas.conf ${KAFKA_OPTS}"
```

#### 创建kafka用户
将以上步骤中的配置应用到所有kafka broker节点
至此，kafka jass.config中配置的broker内部通信用户还未创建起来，直接启动将失败，所以需要先创建kafka用户

* 创建broker内部通信超级用户 (用户名密码与config/kafka_sasl_jaas.conf文件中一致)

```shell
bin/kafka-configs.sh --zookeeper zk:2181 --alter --add-config 'SCRAM-SHA-512=[password=admin-secret!@#]' --entity-type users --entity-name admin
```

* 创建对外开放超级用户（超级用户在config/server.properties中的super.users配置指定）

```shell
bin/kafka-configs.sh --zookeeper zk:2181 --alter --add-config 'SCRAM-SHA-512=[password=public-secret!@#]' --entity-type users --entity-name public
```


* 创建一个测试用户（可选项）

```shell
bin/kafka-configs.sh --zookeeper zk:2181 --alter --add-config 'SCRAM-SHA-512=[password=test-kafka]' --entity-type users --entity-name test-kafka
```

#### 启动kafka集群

```shell
source /etc/profile && cd /workspace/kafka/latest && nohup bin/kafka-server-start.sh config/server.properties > /dev/null 2>&1 &
```

#### 验证broker创建的zk节点信息

查看zk上创建的kafka相关目录acl权限，若kafka通过sasl+acl连接zk集群，则创建的zk节点归kafka用户所有，其它非超级用户仅可读，如下：

```shell
[zk: localhost:2181(CONNECTED) 0] getAcl /brokers/topics
'sasl,'kafka
: cdrwa
'world,'anyone
: r
[zk: localhost:2181(CONNECTED) 1]
```

若权限不正确，请检查kefka配置是否启用了zk acl机制，即配置：zookeeper.set.acl=true

## 测试
### 读写数据
#### 创建topic
（一）zookeeper方式创建
```shell
# 1、由于上述步骤中，在kafka-run-class.sh中已经配置kafka_sasl_jaas.conf文件(Client信息),因此该步骤可忽略，如未配置，需在kafka-topics.sh中添加Client信息

# 2、创建topic
bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --replication-factor 2 --partitions 3 --topic test
```

（二）bootstrap-server方式创建
```shell
# 1、config/conf.properties 文件内容
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512

# 2、由于上述步骤中，在kafka-run-class.sh中已经配置kafka_sasl_jaas.conf文件(KafkaClient信息),因此该步骤可忽略，如未配置，需在kafka-topics.sh中添加KafkaClient信息

# 3、创建topic
bin/kafka-topics.sh --create --bootstrap-server 127.0.0.1:9092 --command-config ./config/conf.properties --replication-factor 2 --partitions 3 --topic test
```

#### 写数据
> 请先确保对应用户是否已经创建，本例中使用test-kafka用户
##### 创建config/producer.properties
```conf
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="test-kafka" password="test-kafka";
```

```shell
bin/kafka-console-producer.sh --topic test --bootstrap-server 127.0.0.1:9092 --producer.config config/producer.properties
```

数据数据后会报以下错误, 原因是还未添加 test-kafka 该用户对test的写权限

```conf
[2021-11-11 15:07:28,160] WARN [Producer clientId=console-producer] Error while fetching metadata with correlation id 3 : {test=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
[2021-11-11 15:07:28,164] ERROR [Producer clientId=console-producer] Topic authorization failed for topics [test] (org.apache.kafka.clients.Metadata)
[2021-11-11 15:07:28,166] ERROR Error when sending message to topic test with key: null, value: 1 bytes with error: (org.apache.kafka.clients.producer.internals.ErrorLoggingCallback)
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [test]
```

```shell
# 第一种：连接zookeeper创建
bin/kafka-acls.sh --authorizer kafka.security.authorizer.AclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:test-kafka --operation Write --topic test

或

# 第二种：使用KafkaAdmin创建（该方式会加载kafka_sasl_jaas.conf文件中KafkaClient信息）
bin/kafka-acls.sh --bootstrap-server 127.0.0.1:9092 --command-config ./conf.properties --add --allow-principal User:test-kafka --operation Write --topic test
```

```log
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)
```

#### 消费数据
##### 添加读权限
```shell
# 第一种：连接zookeeper创建
bin/kafka-acls.sh --authorizer kafka.security.authorizer.AclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:test-kafka --operation Read --topic test

或

# 第二种：使用KafkaAdmin创建（该方式会加载kafka_sasl_jaas.conf文件中KafkaClient信息）
bin/kafka-acls.sh --bootstrap-server 127.0.0.1:9092 --command-config ./conf.properties --add --allow-principal User:test-kafka --operation Read --topic test
```

```log
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)
	(principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)
```

##### 添加消费组权限
```shell
# 第一种：连接zookeeper创建
bin/kafka-acls.sh --authorizer kafka.security.authorizer.AclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:test-kafka --operation Read --topic test --group test

或

# 第二种：使用KafkaAdmin创建（该方式会加载kafka_sasl_jaas.conf文件中KafkaClient信息）
bin/kafka-acls.sh --bootstrap-server 127.0.0.1:9092 --command-config ./conf.properties --add --allow-principal User:test-kafka --operation Read --topic test --group test
```

```log
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)

Adding ACLs for resource `ResourcePattern(resourceType=GROUP, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)
	(principal=User:test-kafka, host=*, operation=WRITE, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=test, patternType=LITERAL)`:
 	(principal=User:test-kafka, host=*, operation=READ, permissionType=ALLOW)
```

##### 消费
```shell
bin/kafka-console-consumer.sh --topic test --bootstrap-server 127.0.0.1:9092 --group test --consumer.config producer.conf
```

### 动态配置用户

添加用户 SCRAM证书
> 注：动态添加用户SCRAM证书，只能使用--zookeeper创建，--bootstrap-server参数只支持创建Quota配置

```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=simba],SCRAM-SHA-512=[password=simba]' --entity-type users --entity-name simba
```

查看用户 SCRAM证书
```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --describe --entity-type users --entity-name simba
```

删除用户 SCRAM证书

```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --delete-config 'SCRAM-SHA-512' --entity-type users --entity-name simba
```

```shell
bin/kafka-configs.sh --zookeeper 127.0.0.1:2181 --alter --delete-config 'SCRAM-SHA-256' --entity-type users --entity-name simba
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
