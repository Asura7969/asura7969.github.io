---
title: Expand Flink Sql
date: 2021-01-06 08:31:57
tags: flink
categories: flink
cover: /img/topimg/202105161045.png
---

# 扩展format
![expand_flink_sql_format_2.png](http://ww1.sinaimg.cn/large/b3b57085gy1gmdzcyhqk5j20wg1664gq.jpg)

**输入数据格式**

```text
ANALYSIS,2020-08-06 17:20:24.066,2,{"condition":{"mail":"zhangsan@qq.com","guid":"1111111","request_id":"b1ca18645f03e01abfd9bcfe5b2e0f3a","status":"200"},"entityMata":{"ipAddress":"0.0.0.0","logValue":2.0,"metric":"metric1","service":"service1"},"traceId":"385307967614d7c1"}
ANALYSIS,2020-08-06 17:21:24.066,2,{"condition":{"mail":"zhangsan@qq.com","guid":"1111111","request_id":"aass2dsfsd3445gdfgd9bcfe5b2e0f3a","status":"200"},"entityMata":{"ipAddress":"0.0.0.0","logValue":3.0,"metric":"metric1","service":"service1"},"traceId":"3asd7967614d7c12"}
```

**解析格式**

![expand_flink_sql_format_3.png](http://ww1.sinaimg.cn/large/b3b57085gy1gmdzg3jsymj20ra0dodmb.jpg)

**源码**

[链接](https://github.com/Asura7969/asuraflink/blob/main/asuraflink-common/src/test/java/org/apache/flink/formats/json/user/UserJsonSerDeSchemaTest.java)