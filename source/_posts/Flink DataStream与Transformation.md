---
title: Flink DataStream与Transformation
date: 2021-04-14 08:31:57
tags: flink
categories: flink
cover: /img/topimg/202106050950.png
---


# DataStream 如何转换为 Transformation ？

### 简介
本文就以中**WordCount**为例

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.fromElements(WordCountData.WORDS);

text.flatMap(new Tokenizer())
    .keyBy(value -> value.f0).sum(1)
    .addSink(new CustomSink());

env.execute("Streaming WordCount");
```

![flink-transformation.png](http://ww1.sinaimg.cn/large/b3b57085gy1gpjei4n9zmj20ms069t9f.jpg)
### DataStreamSource
```java
	public DataStreamSource(
			StreamExecutionEnvironment environment,
			TypeInformation<T> outTypeInfo,
			StreamSource<T, ?> operator,
			boolean isParallel,
			String sourceName,
			Boundedness boundedness) {
        // 传递给父类 DataStream, 并赋值, 后续调用的 flatMap, keyBy, aadSink 等算子都是调用 DataStream 的API
		super(environment, new LegacySourceTransformation<>(sourceName, operator, outTypeInfo, environment.getParallelism(), boundedness));

		this.isParallel = isParallel;
		if (!isParallel) {
			setParallelism(1);
		}
	}

```
* 调用 **fromElements** 算子,会构造出 *DataStreamSource*, *LegacySourceTransformation（SourceTransformation）*
* 最后传递给父类 **DataStream**, 并赋值, 后续调用的 **flatMap**, **keyBy**, **addSink** 等算子都是调用 **DataStream** 的API

### FlatMap
```java
	protected <R> SingleOutputStreamOperator<R> doTransform(
			String operatorName,
			TypeInformation<R> outTypeInfo,
			StreamOperatorFactory<R> operatorFactory) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		transformation.getOutputType();

		OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
				this.transformation,
				operatorName,
				operatorFactory,
				outTypeInfo,
				environment.getParallelism());

		@SuppressWarnings({"unchecked", "rawtypes"})
		SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);
        // 把这个 transform 添加到 env的 list中
		getExecutionEnvironment().addOperator(resultTransform);

		return returnStream;
	}
```
* **getExecutionEnvironment()** 方法就是上节传递给 **DataStream** 的 **env**
* **addOperator** 就是添加list元素

### keyBy
```java
	KeyedStream(
		DataStream<T> stream,
		PartitionTransformation<T> partitionTransformation,
		KeySelector<T, KEY> keySelector,
		TypeInformation<KEY> keyType) {

		super(stream.getExecutionEnvironment(), partitionTransformation);
		this.keySelector = clean(keySelector);
		this.keyType = validateKeyType(keyType);
	}
```
* **keyBy** 操作会重新构造构造 **DataStream**, 并把上游的 **env** 传递给新的 **DataStream**

### addSink
```java
	public DataStreamSink<T> addSink(SinkFunction<T> sinkFunction) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		transformation.getOutputType();

		// configure the type if needed
		if (sinkFunction instanceof InputTypeConfigurable) {
			((InputTypeConfigurable) sinkFunction).setInputType(getType(), getExecutionConfig());
		}

		StreamSink<T> sinkOperator = new StreamSink<>(clean(sinkFunction));

		DataStreamSink<T> sink = new DataStreamSink<>(this, sinkOperator);

		getExecutionEnvironment().addOperator(sink.getTransformation());
		return sink;
	}
```
* 与 **flatMap** 类似

### execute
* 调用 **execute** 方法后会构造**StreamGraph**