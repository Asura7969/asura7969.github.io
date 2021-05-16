---
title: Flink Checkpoint 恢复流程
date: 2021-03-04 08:31:57
tags: flink
categories: flink
cover: /img/topimg/202105161039.png
---

# Checkpoint简介
Flink定期保存数据，failover后从上次成功的保存点处恢复，并提供Exactly-Once的投递保障机制

## CheckpointCoordinator
```java
    // 恢复保存点
    public boolean restoreSavepoint(
            String savepointPointer,
            boolean allowNonRestored,
            Map<JobVertexID, ExecutionJobVertex> tasks,
            ClassLoader userClassLoader) throws Exception {
    
        Preconditions.checkNotNull(savepointPointer, "The savepoint path cannot be null.");
    
        LOG.info("Starting job {} from savepoint {} ({})",
                job, savepointPointer, (allowNonRestored ? "allowing non restored state" : ""));
        // 从指定目录获取hdfs的地址
        final CompletedCheckpointStorageLocation checkpointLocation = checkpointStorage.resolveCheckpoint(savepointPointer);
    
        // 1、加载 metadata 信息
        // 2、生成operator 到 task 的映射
        // 3、检查 并行度
        // 4、转换这次的 savepoint为checkpoint，以便失败后恢复
        CompletedCheckpoint savepoint = Checkpoints.loadAndValidateCheckpoint(
                job, tasks, checkpointLocation, userClassLoader, allowNonRestored);
    
        // 将要恢复的checkpoint信息写到zk，并异步删除旧的checkpoint
        completedCheckpointStore.addCheckpoint(savepoint);
    
        // 重置checkpoint 计数器
        long nextCheckpointId = savepoint.getCheckpointID() + 1;
        checkpointIdCounter.setCount(nextCheckpointId);
    
        LOG.info("Reset the checkpoint ID of job {} to {}.", job, nextCheckpointId);
        // 从最近一次 Checkpoint 处恢复 State
        // 获取OperatorState，分配state
        return restoreLatestCheckpointedStateInternal(new HashSet<>(tasks.values()), true, true, allowNonRestored);
    }
```


## 总结

* 首先客户端提供 Checkpoint 或 Savepoint 的目录
* JM 从给定的目录中找到 _metadata 文件（Checkpoint 的元数据文件）
* JM 解析元数据文件，做一些校验，将信息写入到 zk 中，然后准备从这一次 Checkpoint 中恢复任务
* JM 拿到所有算子对应的 State，给各个 subtask 分配 StateHandle（状态文件句柄）
* TM 启动时，也就是 StreamTask 的初始化阶段会创建 KeyedStateBackend 和 OperatorStateBackend
* 创建过程中就会根据 JM 分配给自己的 StateHandle 从 dfs 上恢复 State