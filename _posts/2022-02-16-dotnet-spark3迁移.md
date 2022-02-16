---
categories: technology
layout: post
---

上两周我在迁移spark2上跑的dotnet job到spark3上去。但是遇到了一个问题，在我替换了依赖项后，原本正常运行的job失败了，日志如下：

```
Caused by: org.apache.spark.api.python.PythonException: System.IO.FileLoadException: Mixed mode assembly is built against version 'v2.0.50727' of the runtime and cannot be loaded in the 4.0 runtime without additional configuration information.
   at [myjobName].<>c__DisplayClass0_0.<Run>b__0(String market, String offerBonds)
   at Microsoft.Spark.Sql.PicklingUdfWrapper`3.Execute(Int32 splitIndex, Object[] input, Int32[] argOffsets)
   at Microsoft.Spark.Worker.Command.PicklingSqlCommandExecutor.ExecuteCore(Stream inputStream, Stream outputStream, SqlCommand[] commands)
   at Microsoft.Spark.Worker.TaskRunner.ProcessStream(Stream inputStream, Stream outputStream, Version version, Boolean& readComplete)
	at org.apache.spark.api.python.BasePythonRunner$ReaderIterator.handlePythonException(PythonRunner.scala:503)
	at org.apache.spark.sql.execution.python.PythonUDFRunner$$anon$2.read(PythonUDFRunner.scala:81)
	at org.apache.spark.sql.execution.python.PythonUDFRunner$$anon$2.read(PythonUDFRunner.scala:64)
	at org.apache.spark.api.python.BasePythonRunner$ReaderIterator.hasNext(PythonRunner.scala:456)
	at org.apache.spark.InterruptibleIterator.hasNext(InterruptibleIterator.scala:37)
	at scala.collection.Iterator$$anon$11.hasNext(Iterator.scala:489)
	at scala.collection.Iterator$$anon$10.hasNext(Iterator.scala:458)
	at scala.collection.Iterator$$anon$10.hasNext(Iterator.scala:458)
	at org.apache.spark.sql.catalyst.expressions.GeneratedClass$GeneratedIteratorForCodegenStage2.processNext(Unknown Source)
	at org.apache.spark.sql.execution.BufferedRowIterator.hasNext(BufferedRowIterator.java:43)
	at org.apache.spark.sql.execution.WholeStageCodegenExec$$anon$1.hasNext(WholeStageCodegenExec.scala:729)
	at scala.collection.Iterator$$anon$11.hasNext(Iterator.scala:489)
	at scala.collection.Iterator$ConcatIterator.hasNext(Iterator.scala:222)
	at scala.collection.Iterator$$anon$10.hasNext(Iterator.scala:458)
	at org.apache.spark.sql.execution.datasources.v2.DataWritingSparkTask$.$anonfun$run$7(WriteToDataSourceV2Exec.scala:438)
	at org.apache.spark.util.Utils$.tryWithSafeFinallyAndFailureCallbacks(Utils.scala:1411)
	at org.apache.spark.sql.execution.datasources.v2.DataWritingSparkTask$.run(WriteToDataSourceV2Exec.scala:477)
	at org.apache.spark.sql.execution.datasources.v2.V2TableWriteExec.$anonfun$writeWithV2$2(WriteToDataSourceV2Exec.scala:385)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:90)
	at org.apache.spark.scheduler.Task.run(Task.scala:127)
	at org.apache.spark.executor.Executor$TaskRunner.$anonfun$run$3(Executor.scala:446)
	at org.apache.spark.util.Utils$.tryWithSafeFinally(Utils.scala:1377)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:449)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

上面的[myjobName]是我们内部的包名。

网上的说法是需要向App.config文件增加额外的配置项：

```xml
  <startup useLegacyV2RuntimeActivationPolicy="true">
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.6.1" />
  </startup>
```

但是我这边的项目内部本来就有这段代码，所以这个方法并不是解决方案。

之后我通过二分法找到了错误的来源，一个C++/CLR dll，目标dotnet framework 3.5。

现在回到上面的方法，`useLegacyV2RuntimeActivationPolicy="true"`表示的是用V2 dotnet运行环境来运行目标dotnet framework低于4.0的C++/CLR dll。

因此这个方法应该是可以生效的，但是为啥就失败了呢。注意到这个dll是在一个UDF调用的，我尝试在main函数中调用这个dll，发现main函数过程中不会抛出异常。

现在问题确定了，只有UDF中，App.config文件会失效。实际上当我们要执行某个dotnet编译的exe文件`X.exe`时，它也会去加载对应的config文件，这个文件的名称是`X.exe.config`。但是很显然调用UDF的入口并不是我们的main入口，因为UDF会被分发到不同的机器上执行，当然不可能重新执行一次完整的spark job，所以光修改我们main入口的config文件是不够的。

查询了一些关于spark和dotnet spark的资料后发现，存在一个名叫dotnet spark worker的项目（位置由环境变量`DOTNET_WORKER_DIR`确定），它会负责驱动我们的UDF，因此我们需要修改`Microsoft.Spark.Worker.exe.config`，并在其中加入上面提到的`useLegacyV2RuntimeActivationPolicy="true"`。