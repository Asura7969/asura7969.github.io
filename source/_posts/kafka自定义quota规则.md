---
title: kafka自定义quota规则
date: 2021-09-23 15:33:30
tags: kafka
categories: kafka
cover: /img/topimg/202109231535.png
---

## 背景
现在运行kafka集群未做限流,为预防流量暴增影响集群稳定性,需对kafka集群做限流

## 目标
可通过指定client-id限流

> kafka自带的user认证部署较复杂,不考虑

## 现状
**应用app**: 接入标准mq sdk的app自带client-id参数，但需要修改，格式：`[固定前缀]_[sdkName]_[appName]_[topic]_xxxx`

**flink**: 未设置client-id，且sdk未推广，格式：`[固定前缀]_flink_[jobId]_[topic]_xxxx`

## 方案
修改参数:client.quota.callback.class
扩展`ClientQuotaCallback`接口


## 实现
### client-Id解析类
```java

import java.util.HashMap;
import java.util.Map;

public class HbClientIdParser {
    public static final String DEFAULT_CLIENT_ID = "client-id";

    public static Map<String, String> quotaMetricTags(String clientId) {
        Map<String, String> metricTags = new HashMap<>();
        if (isHbSDK(clientId)) {

            metricTags.put(DEFAULT_CLIENT_ID, getClientId(clientId));
        } else {
            metricTags.put(DEFAULT_CLIENT_ID, clientId);
        }

        return metricTags;
    }

    private static boolean isHbSDK(String clientId) {
        return clientId.startsWith("hb_") && clientId.split("_").length > 3;
    }

    private static String getClientId(String clientId) {
        String[] clientSplit = clientId.split("_");
        String appType = clientSplit[1];
        String appIdentity = clientSplit[2];
        String topic = clientSplit[3];
        return "hb_" + appType + "_" + appIdentity + "_" + topic;
    }

}

```

### ClientQuotaCallback扩展
```java

import kafka.server.ClientQuotaManager;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.metrics.Quota;
import org.apache.kafka.common.security.auth.KafkaPrincipal;
import org.apache.kafka.server.quota.ClientQuotaCallback;
import org.apache.kafka.server.quota.ClientQuotaEntity;
import org.apache.kafka.server.quota.ClientQuotaType;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

import static com.asura7969.quota.HbClientIdParser.DEFAULT_CLIENT_ID;


/**
 * 按client-id限流
 *<pre>
 *  clientId格式：
 *  {固定前缀}_{hms/flink}_{appName/flinkJob-appId}_{topic}_xxxx
 *{@code
 * bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=1024,request_percentage=200' --entity-type clients --entity-name [固定前缀]_[sdkName/flink]_[appName/flinkJob-id]_[topicName]
 *}</pre>
 *
 */
public class HbQuotaCallback implements ClientQuotaCallback {

    private static final Logger log = LoggerFactory.getLogger(HbQuotaCallback.class);

    private final Map<String, Quota> overriddenQuotas = new ConcurrentHashMap<>();

    @Override
    public Map<String, String> quotaMetricTags(ClientQuotaType quotaType, KafkaPrincipal principal, String clientId) {
        return HbClientIdParser.quotaMetricTags(clientId);
    }

    @Override
    public Double quotaLimit(ClientQuotaType quotaType, Map<String, String> metricTags) {
        String clientId = metricTags.get(DEFAULT_CLIENT_ID);
        Quota quota = null;
        if (clientId != null) {
            quota = overriddenQuotas.get(clientId);
        }
        return quota == null ? null : quota.bound();
    }

    @Override
    public void updateQuota(ClientQuotaType quotaType, ClientQuotaEntity entity, double newValue) {
        ClientQuotaManager.KafkaQuotaEntity quotaEntity = (ClientQuotaManager.KafkaQuotaEntity)entity;
        log.info("Changing {} quota for {} to {}", quotaType, quotaEntity.clientId(), newValue);
        overriddenQuotas.put(quotaEntity.clientId(), new Quota(newValue, true));
    }

    @Override
    public void removeQuota(ClientQuotaType quotaType, ClientQuotaEntity entity) {
        ClientQuotaManager.KafkaQuotaEntity quotaEntity = (ClientQuotaManager.KafkaQuotaEntity)entity;
        log.info("Removing {} quota for {}", quotaType, quotaEntity.clientId());
        overriddenQuotas.remove(quotaEntity.clientId());
    }

    @Override
    public boolean quotaResetRequired(ClientQuotaType quotaType) {
        return false;
    }

    @Override
    public boolean updateClusterMetadata(Cluster cluster) {
        return false;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }
}

```

## 上线&测试
* 修改配置文件
* 重启broker
* 设置client-id限流
* 观察效果

## 参考
> https://github.com/lulf/kafka-static-quota-plugin
