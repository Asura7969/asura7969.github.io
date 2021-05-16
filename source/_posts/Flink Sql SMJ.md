---
title: Flink Sql SMJ
date: 2020-12-23 08:31:57
tags: flink
categories: sql
cover: /img/topimg/202105161047.png
---


# Sort Merge Join
## 简介

> 1.In most cases, its performance is weaker than HashJoin.

> 2.It is more stable than HashJoin, and most of the data can be sorted stably.

> 3.SortMergeJoin should be the best choice if sort can be omitted in the case of multi-level join cascade with the same key.

## Join流程

除了sort、spill、merge，join流程大致与spark类似

## 源码

BinaryExternalSorter
分别包含3个异步线程(sort, spill, merger)，三个线程通过circularQueues实现通信

![CircularQueues.png](http://ww1.sinaimg.cn/large/b3b57085gy1glxqec1pubj20y20mkguc.jpg)

**SortingThread**
```java
    public void go() throws IOException {
        boolean alive = true;

        while (isRunning() && alive) {
            CircularElement element;
            try {
                element = this.queues.sort.take();
            } catch (InterruptedException iex) {
                if (isRunning()) {
                    if (LOG.isErrorEnabled()) {
                        LOG.error(
                                "Sorting thread was interrupted (without being shut down) while grabbing a buffer. " +
                                        "Retrying to grab buffer...");
                    }
                    continue;
                } else {
                    return;
                }
            }

            if (element != EOF_MARKER && element != SPILLING_MARKER) {
                // 不是结束标记或者 spill标记
                if (element.buffer.size() == 0) {
                    // 重置buffer
                    element.buffer.reset();
                    this.queues.empty.add(element);
                    continue;
                }

                if (LOG.isDebugEnabled()) {
                    LOG.debug("Sorting buffer " + element.id + ".");
                }
                // 排序
                this.sorter.sort(element.buffer);

                if (LOG.isDebugEnabled()) {
                    LOG.debug("Sorted buffer " + element.id + ".");
                }
            } else if (element == EOF_MARKER) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Sorting thread done.");
                }
                alive = false;
            }
            // spill 队列中添加 element
            this.queues.spill.add(element);
        }
    }
```

**SpillingThread**
* 1、In-Memory Cache
* 2、In-Memory Merge
* 3、Spilling
* 4、releaseMemory
* 5、往merge队列加入*FINAL_MERGE_MARKER*标记

```java
    public void go() throws IOException {

        final Queue<CircularElement> cache = new ArrayDeque<>();
        CircularElement element;
        boolean cacheOnly = false;

        // ------------------- In-Memory Cache ------------------------
        // fill cache
        while (isRunning()) {
            // take next currWriteBuffer from queue
            try {
                // 队列中获取 element
                element = this.queues.spill.take();
            } catch (InterruptedException iex) {
                throw new IOException("The spilling thread was interrupted.");
            }

            if (element == SPILLING_MARKER) {
                // 如果遇到 SPILLING_MARKER,就到下一阶段 Spilling（Memory Merge阶段需判断cacheOnly变量）
                break;
            } else if (element == EOF_MARKER) {
                // 如果遇到结束标记,则 到下一阶段 Memory Merge
                cacheOnly = true;
                break;
            }
            cache.add(element);
        }

        // check whether the thread was canceled
        if (!isRunning()) {
            return;
        }

        // ------------------- In-Memory Merge ------------------------
        if (cacheOnly) {
            // 直接在内存中做排序,之后释放内存通知调用方,不进入Spilling阶段
            List<MutableObjectIterator<BinaryRowData>> iterators = new ArrayList<>(cache.size());

            for (CircularElement cached : cache) {
                iterators.add(cached.buffer.getIterator());
            }

            // set lazy iterator
            List<BinaryRowData> reusableEntries = new ArrayList<>();
            for (int i = 0; i < iterators.size(); i++) {
                reusableEntries.add(serializer.createInstance());
            }
            setResultIterator(iterators.isEmpty() ? EmptyMutableObjectIterator.get() :
                    iterators.size() == 1 ? iterators.get(0) : new BinaryMergeIterator<>(
                            iterators, reusableEntries, comparator::compare));

            releaseEmptyBuffers();

            // signal merging thread to exit (because there is nothing to merge externally)
            this.queues.merge.add(FINAL_MERGE_MARKER);

            return;
        }

        // ------------------- Spilling Phase ------------------------

        final FileIOChannel.Enumerator enumerator =
                this.ioManager.createChannelEnumerator();

        // loop as long as the thread is marked alive and we do not see the final currWriteBuffer
        while (isRunning()) {
            try {
                // 如果 cache 为空,则获取spill 队列中数据（阻塞）,不为空则直接获取poll
                element = cache.isEmpty() ? queues.spill.take() : cache.poll();
            } catch (InterruptedException iex) {
                if (isRunning()) {
                    LOG.error("Spilling thread was interrupted (without being shut down) while grabbing a buffer. " +
                            "Retrying to grab buffer...");
                    continue;
                } else {
                    return;
                }
            }

            // check if we are still running
            if (!isRunning()) {
                return;
            }
            // check if this is the end-of-work buffer
            if (element == EOF_MARKER) {
                break;
            }

            if (element.buffer.getOccupancy() > 0) {
                // open next channel
                FileIOChannel.ID channel = enumerator.next();
                channelManager.addChannel(channel);

                AbstractChannelWriterOutputView output = null;
                int bytesInLastBuffer;
                int blockCount;

                try {
                    numSpillFiles++;
                    output = FileChannelUtil.createOutputView(ioManager, channel, compressionEnable,
                            compressionCodecFactory, compressionBlockSize, memorySegmentSize);
                    element.buffer.writeToOutput(output);
                    spillInBytes += output.getNumBytes();
                    spillInCompressedBytes += output.getNumCompressedBytes();
                    bytesInLastBuffer = output.close();
                    blockCount = output.getBlockCount();
                    LOG.info("here spill the {}th sort buffer data with {} bytes and {} compressed bytes",
                            numSpillFiles, spillInBytes, spillInCompressedBytes);
                } catch (IOException e) {
                    if (output != null) {
                        output.close();
                        output.getChannel().deleteChannel();
                    }
                    throw e;
                }

                // 将spill 的 文件元数据信息添加到 mergeThread
                this.queues.merge.add(new ChannelWithMeta(channel, blockCount, bytesInLastBuffer));
            }

            // pass empty sort-buffer to reading thread
            element.buffer.reset();
            this.queues.empty.add(element);
        }

        // clear the sort buffers, as both sorting and spilling threads are done.
        releaseSortMemory();

        // signal merging thread to begin the final merge
        this.queues.merge.add(FINAL_MERGE_MARKER);

        // Spilling thread done.
    }
```

**MergingThread**
```java

    @Override
    public void go() throws IOException {

        final List<ChannelWithMeta> spillChannelIDs = new ArrayList<>();
        List<ChannelWithMeta> finalMergeChannelIDs = new ArrayList<>();
        ChannelWithMeta channelID;

        while (isRunning()) {
            try {
                //从merge队列中获取channelID
                channelID = this.queues.merge.take();
            } catch (InterruptedException iex) {
                if (isRunning()) {
                    LOG.error("Merging thread was interrupted (without being shut down) " +
                        "while grabbing a channel with meta. Retrying...");
                    continue;
                } else {
                    return;
                }
            }

            if (!isRunning()) {
                return;
            }
            if (channelID == FINAL_MERGE_MARKER) {
                //判断该channelID是否是最终MERGE标记
                finalMergeChannelIDs.addAll(spillChannelIDs);
                spillChannelIDs.clear();
                // 依据block数量排序channel
                finalMergeChannelIDs.sort(Comparator.comparingInt(ChannelWithMeta::getBlockCount));
                break;
            }

            spillChannelIDs.add(channelID);
            // 如果异步merge禁用,我们只会做最终merge,否则直到 maxFanIn 数量后才开始merge
            // maxFanIn = table.exec.sort.max-num-file-handles = 128(default)
            if (!asyncMergeEnable || spillChannelIDs.size() < maxFanIn) {
                continue;
            }

            // 执行中间合并
            // 将多channel 合并为较少 channel
            finalMergeChannelIDs.addAll(merger.mergeChannelList(spillChannelIDs));
            spillChannelIDs.clear();
        }

        // check if we have spilled some data at all
        if (finalMergeChannelIDs.isEmpty()) {
            if (iterator == null) {
                // notify 调用方获取 Iterator
                setResultIterator(EmptyMutableObjectIterator.get());
            }
        } else {
            // merge channels until sufficient file handles are available
            while (isRunning() && finalMergeChannelIDs.size() > this.maxFanIn) {
                finalMergeChannelIDs = merger.mergeChannelList(finalMergeChannelIDs);
            }

            // Beginning final merge.

            // no need to call `getReadMemoryFromHeap` again,
            // because `finalMergeChannelIDs` must become smaller

            List<FileIOChannel> openChannels = new ArrayList<>();
            // finalMergeChannelIDs转为Iterator
            BinaryMergeIterator<BinaryRowData> iterator = merger.getMergingIterator(
                finalMergeChannelIDs, openChannels);
            channelManager.addOpenChannels(openChannels);
            // notify 调用方获取 Iterator
            setResultIterator(iterator);
        }
    }
```

write方法较为简单,先获取空的**buffer**,把**RowData**写到**buffer**里面,判断阈值大小,往spill队列中添加 *SPILLING_MARKER* 标记

**SortMergeJoinOperator**
```java
    @Override
    public void open() throws Exception {
        // 初始化 两个sorter
        // 分别对left 和 right streamTask开启两个sorter（BinaryExternalSorter）
    }
    @Override
	public void processElement1(StreamRecord<RowData> element) throws Exception {
		this.sorter1.write(element.getValue());
	}

	@Override
	public void processElement2(StreamRecord<RowData> element) throws Exception {
		this.sorter2.write(element.getValue());
	}

    @Override
	public void endInput(int inputId) throws Exception {
		isFinished[inputId - 1] = true;
		if (isAllFinished()) {
			doSortMergeJoin();
		}
	}
    
    private void doSortMergeJoin() throws Exception {
        MutableObjectIterator iterator1 = sorter1.getIterator();
        MutableObjectIterator iterator2 = sorter2.getIterator();
        // 依据join类型匹配数据
        if (type.equals(FlinkJoinType.INNER)) {
            if (!leftIsSmaller) {
                // right更小
                try (SortMergeInnerJoinIterator joinIterator = new SortMergeInnerJoinIterator(
                        serializer1, serializer2, projection1, projection2,
                        keyComparator, iterator1, iterator2, newBuffer(serializer2), filterNulls)) {
                    // 把两个流封装为一个 Iterator
                    innerJoin(joinIterator, false);
                }
            } else {
                // left更小
                try (SortMergeInnerJoinIterator joinIterator = new SortMergeInnerJoinIterator(
                        serializer2, serializer1, projection2, projection1,
                        keyComparator, iterator2, iterator1, newBuffer(serializer1), filterNulls)) {
                    innerJoin(joinIterator, true);
                }
            }
        } else if(...){...}
    }

    private void innerJoin(
    			SortMergeInnerJoinIterator iterator, boolean reverseInvoke) throws Exception {
        // nextInnerJoin方法如下图
        while (iterator.nextInnerJoin()) {
            // 获取 probe 表数据
            RowData probeRow = iterator.getProbeRow();
            // 获取 buffer 的 Iterator
            ResettableExternalBuffer.BufferIterator iter = iterator.getMatchBuffer().newIterator();
            while (iter.advanceNext()) {
                RowData row = iter.getRow();
                // 按条件匹配数据并输出到下游节点
                joinWithCondition(probeRow, row, reverseInvoke);
            }
            iter.close();
        }
    }
```
**SortMergeInnerJoinIterator**
![nextInnerJoin.png](http://ww1.sinaimg.cn/large/b3b57085gy1glxvch32b9j217m114wvj.jpg)




