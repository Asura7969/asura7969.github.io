---
title: Flink 1.12.0源码编译
date: 2021-03-07 08:31:57
tags: flink
categories: flink
cover: https://i.loli.net/2021/05/16/7ySFcK6aLv1MBZm.png
---


# 本机环境
操作系统：win10

```shell
> scala -version
Scala code runner version 2.12.12 -- Copyright 2002-2020, LAMP/EPFL and Lightbend, Inc.


> java -version
java version "1.8.0_271"
Java(TM) SE Runtime Environment (build 1.8.0_271-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.271-b09, mixed mode)


> mvn -version
Apache Maven 3.6.3 (cecedd343002696d0abb50b32b541b8a6ba2883f)
Maven home: E:\apache-maven-3.6.3\bin\..
Java version: 1.8.0_261, vendor: Oracle Corporation, runtime: C:\Program Files\Java\jdk1.8.0_261\jre
Default locale: zh_CN, platform encoding: GBK
OS name: "windows 10", version: "10.0", arch: "amd64", family: "windows"


> git --version
git version 2.27.0.windows.1
```



# flink 源码编译

## maven mirror 配置
```xml
    <mirror>
        <id>nexus-hortonworks</id>
        <mirrorOf>central</mirrorOf>
        <name>Nexus hortonworks</name>
        <url>https://repo.hortonworks.com/content/groups/public/</url>
    </mirror>
    <mirror>
        <id>central</id>
        <name>Maven Repository Switchboard</name>
        <url>http://repo1.maven.org/maven2/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
        <id>central2</id>
        <name>Maven Repository Switchboard</name>
        <url>http://repo1.maven.apache.org/maven2/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
```

## 源码下载编译

* 下载
```shell
git clone git@github.com:apache/flink.git
```
* 下载完后直接导入idea
* 切换分支到1.12.0
* 修改根目录下pom文件内容，如下：
```xml

```
* 执行如下操作
![1615126622(1).jpg](http://ww1.sinaimg.cn/large/b3b57085gy1gobpb8yodtj21hc0u04d3.jpg)
* 开始编译
```shell
mvn clean install -DskipTests -Drat.skip=true -Dcheckstyle.skip=true -Dscala=2.12.12
```


## 编译报错
编译期间若有jar下载不到, [到这下载](http://packages.confluent.io/maven/io/confluent/kafka-schema-registry-client/5.5.2/)
> 注意替换包路径

下载完后添加到本地maven仓库对应的路径下面

编译失败后从失败处继续往下编译,如下为**flink-avro-confluent-registry**编译失败
> mvn -rf :flink-avro-confluent-registry clean install -DskipTests -Drat.skip=true -Dcheckstyle.skip=true -Dscala=2.12.12

跳过失败的模块，最后再报错
> mvn clean install --fail-at-end

IDEA运行flink 报Error:java: 无效的标记: --add-exports=java.base/sun.ne
> "Intellij" -> "View" -> "Tool Windows" ->"Maven" -> "Profiles" -> 取消 "java11" -> 重新导入 maven 项目。
> 重新reload maven

java: 警告: 源发行版 11 需要目标发行版 11

修改parent-pom(根目录pom文件)
```xml
<!-- 大概在1001行 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <!-- 此处 11 改为 8 -->
        <source>11</source>
        <target>11</target>
        <compilerArgs combine.children="append">
            <arg>--add-exports=java.base/sun.net.util=ALL-UNNAMED</arg>
            <arg>--add-exports=java.management/sun.management=ALL-UNNAMED</arg>
            <arg>--add-exports=java.rmi/sun.rmi.registry=ALL-UNNAMED</arg>
            <arg>--add-exports=java.security.jgss/sun.security.krb5=ALL-UNNAMED</arg>
        </compilerArgs>
    </configuration>
</plugin>
```