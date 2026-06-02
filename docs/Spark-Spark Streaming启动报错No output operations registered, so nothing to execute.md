错误信息
```
java.lang.IllegalArgumentException: requirement failed: No output operations registered, so nothing to execute
	at scala.Predef$.require(Predef.scala:224)
	at org.apache.spark.streaming.DStreamGraph.validate(DStreamGraph.scala:168)
	at org.apache.spark.streaming.StreamingContext.validate(StreamingContext.scala:513)
	at org.apache.spark.streaming.StreamingContext.liftedTree1$1(StreamingContext.scala:573)
	at org.apache.spark.streaming.StreamingContext.start(StreamingContext.scala:572)
	at com.canaan.data.miner.job.StreamingWithKafka$.main(StreamingWithKafka.scala:50)
	at com.canaan.data.miner.job.StreamingWithKafka.main(StreamingWithKafka.scala)
Exception in thread "main" java.lang.IllegalArgumentException: requirement failed: No output operations registered, so nothing to execute
	at scala.Predef$.require(Predef.scala:224)
	at org.apache.spark.streaming.DStreamGraph.validate(DStreamGraph.scala:168)
	at org.apache.spark.streaming.StreamingContext.validate(StreamingContext.scala:513)
	at org.apache.spark.streaming.StreamingContext.liftedTree1$1(StreamingContext.scala:573)
	at org.apache.spark.streaming.StreamingContext.start(StreamingContext.scala:572)
	at com.canaan.data.miner.job.StreamingWithKafka$.main(StreamingWithKafka.scala:50)
	at com.canaan.data.miner.job.StreamingWithKafka.main(StreamingWithKafka.scala)
```

产生的原因：
没有触发DStream需要的aciton 

解决方法:
使用以下方法之一触发：

print()
foreachRDD()
saveAsObjectFiles()
saveAsTextFiles()
saveAsHadoopFiles()
