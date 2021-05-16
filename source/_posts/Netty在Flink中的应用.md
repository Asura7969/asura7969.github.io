---
title: Netty在Flink中的应用
date: 2021-03-29 08:31:57
tags: flink
categories: flink
cover: https://i.loli.net/2021/05/16/USDe9tZmuqTNGRo.jpg
---


# 使用Netty定义Client与Server端的协议

## client 端协议
![flink-client-netty.png](http://ww1.sinaimg.cn/large/b3b57085gy1gp0jjef7d5j214e0qidn5.jpg)

## server 端协议
![flink-server-netty.png](http://ww1.sinaimg.cn/large/b3b57085gy1gp0jkjgt86j213a0redmq.jpg)

> org.apache.flink.runtime.io.network.netty.NettyProtocol


**CreditBasedPartitionRequestClientHandler**

```java
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        try {
            decodeMsg(msg);
        } catch (Throwable t) {
            notifyAllChannelsOfErrorAndClose(t);
        }
    }

    private void decodeMsg(Object msg) throws Throwable {
        final Class<?> msgClazz = msg.getClass();
        
        // ---- Buffer --------------------------------------------------------
        if (msgClazz == NettyMessage.BufferResponse.class) {
            NettyMessage.BufferResponse bufferOrEvent = (NettyMessage.BufferResponse) msg;
        
            RemoteInputChannel inputChannel = inputChannels.get(bufferOrEvent.receiverId);
            if (inputChannel == null || inputChannel.isReleased()) {
                bufferOrEvent.releaseBuffer();
        
                cancelRequestFor(bufferOrEvent.receiverId);
        
                return;
            }
        
            try {
                // 解码buffer
                decodeBufferOrEvent(inputChannel, bufferOrEvent);
            } catch (Throwable t) {
                inputChannel.onError(t);
            }
        
        } else if (msgClazz == NettyMessage.ErrorResponse.class) {
            .....
        }
    }

    private void decodeBufferOrEvent(
            RemoteInputChannel inputChannel, NettyMessage.BufferResponse bufferOrEvent)
            throws Throwable {
        if (bufferOrEvent.isBuffer() && bufferOrEvent.bufferSize == 0) {
            // TODO:触发反压？
            inputChannel.onEmptyBuffer(bufferOrEvent.sequenceNumber, bufferOrEvent.backlog);
        } else if (bufferOrEvent.getBuffer() != null) {
            // 承接下面的源码
            inputChannel.onBuffer(
                    bufferOrEvent.getBuffer(), bufferOrEvent.sequenceNumber, bufferOrEvent.backlog);
        } else {
            throw new IllegalStateException(
                "The read buffer is null in credit-based input channel.");
        }
    }
```

**RemoteInputChannel**

```java
    // 承接上面的源码(bufferOrEvent 方法)
    public void onBuffer(Buffer buffer, int sequenceNumber, int backlog) throws IOException {
        boolean recycleBuffer = true;
    
        try {
            if (expectedSequenceNumber != sequenceNumber) {
                onError(new BufferReorderingException(expectedSequenceNumber, sequenceNumber));
                return;
            }
    
            final boolean wasEmpty;
            boolean firstPriorityEvent = false;
            synchronized (receivedBuffers) {
                NetworkActionsLogger.traceInput(
                        "RemoteInputChannel#onBuffer",
                        buffer,
                        inputGate.getOwningTaskName(),
                        channelInfo,
                        channelStatePersister,
                        sequenceNumber);
                // Similar to notifyBufferAvailable(), make sure that we never add a buffer
                // after releaseAllResources() released all buffers from receivedBuffers
                // (see above for details).
                if (isReleased.get()) {
                    return;
                }
    
                wasEmpty = receivedBuffers.isEmpty();
    
                SequenceBuffer sequenceBuffer = new SequenceBuffer(buffer, sequenceNumber);
                DataType dataType = buffer.getDataType();
                if (dataType.hasPriority()) {
                    // 如果有优先级, 添加到队列头部
                    firstPriorityEvent = addPriorityBuffer(sequenceBuffer);
                } else {
                    receivedBuffers.add(sequenceBuffer);
                    if (dataType.requiresAnnouncement()) {
                        // 针对 CheckpointBarrier 元素
                        firstPriorityEvent = addPriorityBuffer(announce(sequenceBuffer));
                    }
                }
                channelStatePersister
                        .checkForBarrier(sequenceBuffer.buffer)
                        .filter(id -> id > lastBarrierId)
                        .ifPresent(
                                id -> {
                                    // checkpoint was not yet started by task thread,
                                    // so remember the numbers of buffers to spill for the time when
                                    // it will be started
                                    lastBarrierId = id;
                                    lastBarrierSequenceNumber = sequenceBuffer.sequenceNumber;
                                });
                channelStatePersister.maybePersist(buffer);
                ++expectedSequenceNumber;
            }
            recycleBuffer = false;
    
            if (firstPriorityEvent) {
                // 通知 有优先级事件(此方法调用InputGate方法)
                notifyPriorityEvent(sequenceNumber);
            }
            if (wasEmpty) {
                // 通知下游消费(此方法调用InputGate方法)
                notifyChannelNonEmpty();
            }
    
            if (backlog >= 0) {
                // 如果上游 ResultSubpartition 有囤积的backlog
                onSenderBacklog(backlog);
            }
        } finally {
            if (recycleBuffer) {
                // 释放buffer资源
                buffer.recycleBuffer();
            }
        }
    }

```

![flink-通信栈.png](http://ww1.sinaimg.cn/large/b3b57085gy1gp1ov646qdj20q60ecmz4.jpg)

