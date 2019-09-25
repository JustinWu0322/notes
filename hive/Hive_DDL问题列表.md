# Hive DDL问题列表


## 环境信息

- CDH-5.14.0
- SPARK2-2.3.0.cloudera4-1
- HIVE-1.1.0-cdh5.14.0
- HADOOP-2.6.0-cdh5.14.0

## Hive创建parquet格式表后，Spark SQL无法写入数据

### 背景

- 在hive终端执行 `create table  table_name(....) stored as parquet;`
	
	```
		 CREATE TABLE `test_wwx.test_hive`(                
	   `id_pk` string,                                  
	   `uid` string,                                    
	   `card_id` string,                                
	   `owner` string,                                  
	   `card_no` string,                                
	   `idcard` string)                                 
	 PARTITIONED BY (                                   
	   `dt` string) 
	   stored as parquet;
	
	```
- Spark SQL

	```
		 val sql ="select id_pk,uid,card_id,owner,card_no,idcard,dt from test_wwx.MEMBER_BANKCARD_UID_IDCARD_ANA_test where dt>='2019-09-09' and dt<'2019-09-10'"

		 val df=spark.sql(sql)
		
		 df.write.partitionBy("dt").mode("append").format("parquet").saveAsTable("test_wwx.test_hive")
		 
	```

### 异常信息

- 异常1

```
	scala> df.write.partitionBy("dt").mode("append").saveAsTable("test_wwx.test09");
	
	
	org.apache.spark.sql.AnalysisException: The format of the existing table test_wwx.test09 is `HiveFileFormat`. It doesn't match the specified format `ParquetFileFormat`.;
	  at org.apache.spark.sql.execution.datasources.PreprocessTableCreation$$anonfun$apply$2.applyOrElse(rules.scala:117)
	  at org.apache.spark.sql.execution.datasources.PreprocessTableCreation$$anonfun$apply$2.applyOrElse(rules.scala:76)
	  at org.apache.spark.sql.catalyst.trees.TreeNode$$anonfun$2.apply(TreeNode.scala:267)
	  at org.apache.spark.sql.catalyst.trees.TreeNode$$anonfun$2.apply(TreeNode.scala:267)
	  at org.apache.spark.sql.catalyst.trees.CurrentOrigin$.withOrigin(TreeNode.scala:70)
	  at org.apache.spark.sql.catalyst.trees.TreeNode.transformDown(TreeNode.scala:266)
	  at org.apache.spark.sql.catalyst.trees.TreeNode.transform(TreeNode.scala:256)
	  at org.apache.spark.sql.execution.datasources.PreprocessTableCreation.apply(rules.scala:76)
	  at org.apache.spark.sql.execution.datasources.PreprocessTableCreation.apply(rules.scala:72)
	  at org.apache.spark.sql.catalyst.rules.RuleExecutor$$anonfun$execute$1$$anonfun$apply$1.apply(RuleExecutor.scala:87)
	  at org.apache.spark.sql.catalyst.rules.RuleExecutor$$anonfun$execute$1$$anonfun$apply$1.apply(RuleExecutor.scala:84)
	  at scala.collection.IndexedSeqOptimized$class.foldl(IndexedSeqOptimized.scala:57)
	  at scala.collection.IndexedSeqOptimized$class.foldLeft(IndexedSeqOptimized.scala:66)
	
```


- 异常2

```
scala> df.write.partitionBy("dt").mode("append").format("parquet").saveAsTable("test_wwx.test_hive")

java.lang.IllegalArgumentException: Expected exactly one path to be specified, but got:
  at org.apache.spark.sql.execution.datasources.DataSource.planForWritingFileFormat(DataSource.scala:456)
  at org.apache.spark.sql.execution.datasources.DataSource.writeAndRead(DataSource.scala:516)
  at org.apache.spark.sql.execution.command.CreateDataSourceTableAsSelectCommand.saveDataIntoTable(createDataSourceTables.scala:216)
  at org.apache.spark.sql.execution.command.CreateDataSourceTableAsSelectCommand.run(createDataSourceTables.scala:166)
  at org.apache.spark.sql.execution.command.DataWritingCommandExec.sideEffectResult$lzycompute(commands.scala:104)
  at org.apache.spark.sql.execution.command.DataWritingCommandExec.sideEffectResult(commands.scala:102)
  at org.apache.spark.sql.execution.command.DataWritingCommandExec.doExecute(commands.scala:122)
  at org.apache.spark.sql.execution.SparkPlan$$anonfun$execute$1.apply(SparkPlan.scala:131)
  at org.apache.spark.sql.execution.SparkPlan$$anonfun$execute$1.apply(SparkPlan.scala:127)
  at org.apache.spark.sql.execution.SparkPlan$$anonfun$executeQuery$1.apply(SparkPlan.scala:155)
  at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:151)
  at org.apache.spark.sql.execution.SparkPlan.executeQuery(SparkPlan.scala:152)
  at org.apache.spark.sql.execution.SparkPlan.execute(SparkPlan.scala:127)
  at org.apache.spark.sql.execution.QueryExecution.toRdd$lzycompute(QueryExecution.scala:80)
  at org.apache.spark.sql.execution.QueryExecution.toRdd(QueryExecution.scala:80)
  at org.apache.spark.sql.DataFrameWriter$$anonfun$runCommand$1.apply(DataFrameWriter.scala:656)
  at org.apache.spark.sql.DataFrameWriter$$anonfun$runCommand$1.apply(DataFrameWriter.scala:656)
  at org.apache.spark.sql.execution.SQLExecution$.withNewExecutionId(SQLExecution.scala:77)
  at org.apache.spark.sql.DataFrameWriter.runCommand(DataFrameWriter.scala:656)
  at org.apache.spark.sql.DataFrameWriter.createTable(DataFrameWriter.scala:458)
  at org.apache.spark.sql.DataFrameWriter.saveAsTable(DataFrameWriter.scala:437)
  at org.apache.spark.sql.DataFrameWriter.saveAsTable(DataFrameWriter.scala:393)
  ... 49 elided

```


### 解决方案

```
alter table test_hive set tblproperties ('spark.sql.sources.provider'='parquet');

alter table test_hive set  SERDEPROPERTIES ('path'='hdfs://name01:8020/user/hive/warehouse/test_wwx.db/test_hive');
	
```


## Spark SQL自动创建parquet表后，表新增列后，无法写入数据，报列个数不匹配

### 背景


### 报错信息

```
	scala> df.write.mode("overwrite").format("parquet").insertInto("test_wwx.test09");
	
	org.apache.spark.sql.AnalysisException: `test_wwx`.`test09` requires that the data to be inserted have the same number of columns as the target table: target table has 9 column(s) but the inserted data has 10 column(s), including 0 partition column(s) having constant value(s).;
	  at org.apache.spark.sql.execution.datasources.PreprocessTableInsertion.org$apache$spark$sql$execution$datasources$PreprocessTableInsertion$$preprocess(rules.scala:341)
	  at org.apache.spark.sql.execution.datasources.PreprocessTableInsertion$$anonfun$apply$3.applyOrElse(rules.scala:373)
	  at org.apache.spark.sql.execution.datasources.PreprocessTableInsertion$$anonfun$apply$3.applyOrElse(rules.scala:368)
	  at org.apache.spark.sql.catalyst.trees.TreeNode$$anonfun$2.apply(TreeNode.scala:267)
	  at org.apache.spark.sql.catalyst.trees.TreeNode$$anonfun$2.apply(TreeNode.scala:267)
	  at org.apache.spark.sql.catalyst.trees.CurrentOrigin$.withOrigin(TreeNode.scala:70)
	  at org.apache.spark.sql.catalyst.trees.TreeNode.transformDown(TreeNode.scala:266)
	  at org.apache.spark.sql.catalyst.trees.TreeNode.transform(TreeNode.scala:256)
	  at org.apache.spark.sql.execution.datasources.PreprocessTableInsertion.apply(rules.scala:368)
	  at org.apache.spark.sql.execution.datasources.PreprocessTableInsertion.apply(rules.scala:328)
	  at org.apache.spark.sql.catalyst.rules.RuleExecutor$$anonfun$execute$1$$anonfun$apply$1.apply(RuleExecutor.scala:87)
	  at org.apache.spark.sql.catalyst.rules.RuleExecutor$$anonfun$execute$1$$anonfun$apply$1.apply(RuleExecutor.scala:84)
	  at scala.collection.IndexedSeqOptimized$class.foldl(IndexedSeqOptimized.scala:57)
	  at scala.collection.IndexedSeqOptimized$class.foldLeft(IndexedSeqOptimized.scala:66)
	  at scala.collection.mutable.ArrayBuffer.foldLeft(ArrayBuffer.scala:48)
	  at org.apache.spark.sql.catalyst.rules.RuleExecutor$$anonfun$execute$1.apply(RuleExecutor.scala:84)
	  at org.apache.spark.sql.catalyst.rules.RuleExecutor$$anonfun$execute$1.apply(RuleExecutor.scala:76)
	  at scala.collection.immutable.List.foreach(List.scala:381)
	
```


### 解决方案

```
spark saveAsTable不支持新增/修改列操作


* When the DataFrame is created from a non-partitioned `HadoopFsRelation` with a single input
* path, and the data source provider can be mapped to an existing Hive builtin SerDe (i.e. ORC
* and Parquet), the table is persisted in a Hive compatible format, which means other systems
* like Hive will be able to read this table. Otherwise, the table is persisted in a Spark SQL
* specific format.
```