---
title: Flink1.12 Checkpoint源码解析
date: 2021-03-25 08:31:57
tags: flink
categories: flink
cover: https://i.loli.net/2021/05/16/LcIghOnCaFf9TV8.png
---

# Checkpoint

## CheckpointCoordinator
### startTriggeringCheckpoint

```java
    private void startTriggeringCheckpoint(CheckpointTriggerRequest request) {
        ...
        // 1、初始化checkpointId和checkpoint存储state位置
        final CompletableFuture<PendingCheckpoint> pendingCheckpointCompletableFuture =
                initializeCheckpoint(request.props, request.externalSavepointLocation)
                    .thenApplyAsync(
                        (checkpointIdAndStorageLocation) -> createPendingCheckpoint(
                            timestamp,
                            request.props,
                            ackTasks,
                            request.isPeriodic,
                            checkpointIdAndStorageLocation.checkpointId,
                            checkpointIdAndStorageLocation.checkpointStorageLocation,
                            request.getOnCompletionFuture()),
                        timer);
        // 2、创建待处理的checkpoint（pendingCheckpoint）并设置超时回调
        final CompletableFuture<?> coordinatorCheckpointsComplete = pendingCheckpointCompletableFuture
                .thenComposeAsync((pendingCheckpoint) ->
                        OperatorCoordinatorCheckpoints.triggerAndAcknowledgeAllCoordinatorCheckpointsWithCompletion(
                                coordinatorsToCheckpoint, pendingCheckpoint, timer),
                        timer);
    
        // We have to take the snapshot of the master hooks after the coordinator checkpoints has completed.
        // This is to ensure the tasks are checkpointed after the OperatorCoordinators in case
        // ExternallyInducedSource is used.
        // 3、source端触发checkpoint（主要用于外部的Source）
        final CompletableFuture<?> masterStatesComplete = coordinatorCheckpointsComplete
            .thenComposeAsync(ignored -> {
                // If the code reaches here, the pending checkpoint is guaranteed to be not null.
                // We use FutureUtils.getWithoutException() to make compiler happy with checked
                // exceptions in the signature.
                PendingCheckpoint checkpoint =
                    FutureUtils.getWithoutException(pendingCheckpointCompletableFuture);
                return snapshotMasterState(checkpoint);
            }, timer);
    
        FutureUtils.assertNoException(
            CompletableFuture.allOf(masterStatesComplete, coordinatorCheckpointsComplete)
                .handleAsync(
                    (ignored, throwable) -> {
                        final PendingCheckpoint checkpoint =
                            FutureUtils.getWithoutException(pendingCheckpointCompletableFuture);
    
                        Preconditions.checkState(
                            checkpoint != null || throwable != null,
                            "Either the pending checkpoint needs to be created or an error must have been occurred.");
    
                        if (throwable != null) {
                            // the initialization might not be finished yet
                            if (checkpoint == null) {
                                onTriggerFailure(request, throwable);
                            } else {
                                onTriggerFailure(checkpoint, throwable);
                            }
                        } else {
                            if (checkpoint.isDisposed()) {
                                onTriggerFailure(
                                    checkpoint,
                                    new CheckpointException(
                                        CheckpointFailureReason.TRIGGER_CHECKPOINT_FAILURE,
                                        checkpoint.getFailureCause()));
                            } else {
                                // no exception, no discarding, everything is OK
                                // 4、判断前几个步骤异步执行是否发生异常，没有异常则rpc调用各个task执行checkpoint
                                // 5、JobMaster接受各个operator的结果信息
                                final long checkpointId = checkpoint.getCheckpointId();
                                snapshotTaskState(
                                    timestamp,
                                    checkpointId,
                                    checkpoint.getCheckpointStorageLocation(),
                                    request.props,
                                    executions,
                                    request.advanceToEndOfTime);
    
                                coordinatorsToCheckpoint.forEach((ctx) -> ctx.afterSourceBarrierInjection(checkpointId));
                                // It is possible that the tasks has finished checkpointing at this point.
                                // So we need to complete this pending checkpoint.
                                if (!maybeCompleteCheckpoint(checkpoint)) {
                                    return null;
                                }
                                onTriggerSuccess();
                            }
                        }
                        return null;
                    },
                    timer)
                .exceptionally(error -> {
                    if (!isShutdown()) {
                        throw new CompletionException(error);
                    } else if (findThrowable(error, RejectedExecutionException.class).isPresent()) {
                        LOG.debug("Execution rejected during shutdown");
                    } else {
                        LOG.warn("Error encountered during shutdown", error);
                    }
                    return null;
                }));
    
    }

```

## SubtaskCheckpointCoordinatorImpl
### checkpointState

```java
    public void checkpointState(
            CheckpointMetaData metadata,
            CheckpointOptions options,
            CheckpointMetricsBuilder metrics,
            OperatorChain<?, ?> operatorChain,
            Supplier<Boolean> isCanceled) throws Exception {
    
        checkNotNull(options);
        checkNotNull(metrics);
    
        // All of the following steps happen as an atomic step from the perspective of barriers and
        // records/watermarks/timers/callbacks.
        // We generally try to emit the checkpoint barrier as soon as possible to not affect downstream
        // checkpoint alignments
        // 1、检查checkpoint是否是最新的，不是最新的丢弃
        if (lastCheckpointId >= metadata.getCheckpointId()) {
            LOG.info("Out of order checkpoint barrier (aborted previously?): {} >= {}", lastCheckpointId, metadata.getCheckpointId());
            channelStateWriter.abort(metadata.getCheckpointId(), new CancellationException(), true);
            checkAndClearAbortedStatus(metadata.getCheckpointId());
            return;
        }
    
        // Step (0): Record the last triggered checkpointId and abort the sync phase of checkpoint if necessary.
        // 2、记录metadata中的checkpointId为最新的checkpoint
        lastCheckpointId = metadata.getCheckpointId();
        if (checkAndClearAbortedStatus(metadata.getCheckpointId())) {
            // broadcast cancel checkpoint marker to avoid downstream back-pressure due to checkpoint barrier align.
            operatorChain.broadcastEvent(new CancelCheckpointMarker(metadata.getCheckpointId()));
            LOG.info("Checkpoint {} has been notified as aborted, would not trigger any checkpoint.", metadata.getCheckpointId());
            return;
        }
    
        // Step (1): Prepare the checkpoint, allow operators to do some pre-barrier work.
        //           The pre-barrier work should be nothing or minimal in the common case.
        // 3、checkpoint之前的准备工作（轻量级的）
        operatorChain.prepareSnapshotPreBarrier(metadata.getCheckpointId());
    
        // Step (2): Send the checkpoint barrier downstream
        // 4、往下游发送checkpoint
        operatorChain.broadcastEvent(
            new CheckpointBarrier(metadata.getCheckpointId(), metadata.getTimestamp(), options),
            options.isUnalignedCheckpoint());
    
        // Step (3): Prepare to spill the in-flight buffers for input and output
        // 5、判断是否是UnalignedCheckpoint
        if (options.isUnalignedCheckpoint()) {
            // output data already written while broadcasting event
            channelStateWriter.finishOutput(metadata.getCheckpointId());
        }
        // 6、异步snapshot
        // Step (4): Take the state snapshot. This should be largely asynchronous, to not impact progress of the
        // streaming topology
    
        Map<OperatorID, OperatorSnapshotFutures> snapshotFutures = new HashMap<>(operatorChain.getNumberOfOperators());
        try {
            if (takeSnapshotSync(snapshotFutures, metadata, metrics, options, operatorChain, isCanceled)) {
                finishAndReportAsync(snapshotFutures, metadata, metrics, options);
            } else {
                cleanup(snapshotFutures, metadata, metrics, new Exception("Checkpoint declined"));
            }
        } catch (Exception ex) {
            cleanup(snapshotFutures, metadata, metrics, ex);
            throw ex;
        }
    }
```

# Checkpoint恢复流程

* 首先客户端提供 Checkpoint 或 Savepoint 的目录
* JM 从给定的目录中找到 _metadata 文件（Checkpoint 的元数据文件）
* JM 解析元数据文件，做一些校验，将信息写入到 zk 中，然后准备从这一次 Checkpoint 中恢复任务
* JM 拿到所有算子对应的 State，给各个 subtask 分配 StateHandle（状态文件句柄）
* TM 启动时，也就是 StreamTask 的初始化阶段会创建 KeyedStateBackend 和 OperatorStateBackend
* 创建过程中就会根据 JM 分配给自己的 StateHandle 从 dfs 上恢复 State

## CheckpointCoordinator
### restoreSavepoint
```java
    public boolean restoreSavepoint(
            String savepointPointer,
            boolean allowNonRestored,
            Map<JobVertexID, ExecutionJobVertex> tasks,
            ClassLoader userClassLoader)
            throws Exception {
    
        Preconditions.checkNotNull(savepointPointer, "The savepoint path cannot be null.");
    
        LOG.info(
                "Starting job {} from savepoint {} ({})",
                job,
                savepointPointer,
                (allowNonRestored ? "allowing non restored state" : ""));
        // 获取存储地址
        final CompletedCheckpointStorageLocation checkpointLocation =
                checkpointStorageView.resolveCheckpoint(savepointPointer);
    
        // Load the savepoint as a checkpoint into the system
        // 加载并校验数据
        CompletedCheckpoint savepoint =
                Checkpoints.loadAndValidateCheckpoint(
                        job, tasks, checkpointLocation, userClassLoader, allowNonRestored);
        
        // 将要恢复的 Checkpoint 信息写入到 zk 中
        completedCheckpointStore.addCheckpoint(
                savepoint, checkpointsCleaner, this::scheduleTriggerRequest);
    
        // Reset the checkpoint ID counter
        long nextCheckpointId = savepoint.getCheckpointID() + 1;
        checkpointIdCounter.setCount(nextCheckpointId);
    
        LOG.info("Reset the checkpoint ID of job {} to {}.", job, nextCheckpointId);
        // 从最近一次 Checkpoint 处恢复 State
        final OptionalLong restoredCheckpointId =
                restoreLatestCheckpointedStateInternal(
                        new HashSet<>(tasks.values()),
                        OperatorCoordinatorRestoreBehavior.RESTORE_IF_CHECKPOINT_PRESENT,
                        true,
                        allowNonRestored);
    
        return restoredCheckpointId.isPresent();
    }


```