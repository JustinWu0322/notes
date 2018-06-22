## goldengate 实现mysql到kafka同步

> Oracle GoldenGate 提供异构环境间事务数据的实时、低影响的捕获、路由、转换和交付

###goldengate架构


<img src="https://cdn.app.compendium.com/uploads/user/e7c690e8-6ff9-102a-ac6d-e4aebca50425/f4a5b21d-66fa-4885-92bf-c4e81c06d916/Image/817446762d2fc432d0eb9520cfc63fa3/image.png" width = "700" height = "500" alt="图片名称" align=center />

<img src="https://1.bp.blogspot.com/-J7PemBpYmqA/WaMj94nrdkI/AAAAAAAADLA/eUDxIK-ihMYFHvionc6Di-cbGWfWo0VzwCEwYBhgL/s1600/Oraclegg.png"  width = "700" height = "400" alt="图片名称" align=center >


#### goldengate相关概念

- Manager进程是GoldenGate的控制进程，运行在源端和目标端上。它主要作用有以下几个方面：启动、监控、重启Goldengate的其他进程，报告错误及事件，分配数据存储空间，发布阀值报告等。在目标端和源端有且只有一个manager进程
- Extract运行在数据库源端，负责从源端数据表或者日志中捕获数据
- Data Pump进程运行在数据库源端，其作用是将源端产生的本地trail文件，把trail以数据块的形式通过TCP/IP 协议发送到目标端。
- Collector进程与Data Pump进程对应 的叫Server Collector进程，这个进程不需要引起我的关注，因为在实际操作过程中，无需我们对其进行任何配置，所以对我们来说它是透明的。它运行在目标端，其 任务就是把Extract/Pump投递过来的数据重新组装成远程ttrail文件。
- Replicat进程，通常我们也把它叫做应用进程。运行在目标端，是数据传递的最后一站，负责读取目标端trail文件中的内容。

- 关于OGG的Trail文件
	- 为了更有效、更安全的把数据库事务信息从源端投递到目标端。GoldenGate引进trail文件的概念。前面提到extract抽取完数据以 后 Goldengate会将抽取的事务信息转化为一种GoldenGate专有格式的文件。然后pump负责把源端的trail文件投递到目标端，所以源、 目标两端都会存在这种文件。
	- trail文件存在的目的旨在防止单点故障，将事务信息持久化，并且使用checkpoint机制来记录其读写位置，如果故障发生，则数据可以根据checkpoint记录的位置来重传 。

### 系统环境配置（源端和目标端）

- JDK 版本
	
	```
	jdk1.8
	```

- 添加环境变量	

	```
	export GGS_HOME=/opt/ggs
	export PATH=$PATH:$GGS_HOME
	```
	
	*本篇安装文档goldengate安装目录为：/opt/ggs*
	
- 环境

	```
	yp-data02 (源端，mysql所在机器)
	
	yp-data01 （目标端）
	```
	
- goldengate版本

	```
	Oracle GoldenGate 12.3.0.1.1 for MySQL on Linux x86-64   (源端版本)
	
	Oracle GoldenGate for Big Data 12.3.1.1.1 on Linux x86-64 （目标端版本）
	```
	- 下载地址
		
		```
		http://www.oracle.com/technetwork/middleware/goldengate/downloads/index.html
		```
- 配置mysql binlog,配置后需重启mysql实例 (源端)

	```
	# vim /etc/my.cnf
	
	log-bin=mysql-bin
	binlog_format=row
	server-id=1
	``` 
- mysql 创建测试表(源端数据库)

	```
	CREATE TABLE `test`.`wms_test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `first_name` varchar(60) DEFAULT NULL,
  `last_name` varchar(60) DEFAULT NULL,
  `sex` varchar(45) DEFAULT NULL,
  `address` varchar(200) DEFAULT NULL,
  `flag` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8
	```
	
- kafka环境（目标端）
	
### 源端配置（mysql）

- 配置命令 
	
	```
	$GGS_HOME/ggsci
	```
- mysql用户登录
	
	```
	dblogin sourcedb test@localhost:3306,userid root,password 123456  
	```
-  配置manager

	 - `CREATE SUBDIRS` 创建目录
	 
	 - 配置命令
	 	
	 	```
	 	# edit param mgr
		port 17809
		dynamicportlist 17800-18000
		purgeoldextracts ./dirdat/*,usecheckpoints, minkeepdays 7
	 	```
	 -  启动mgr
	 	
	 	```
	 	#  start mgr
		查看启动进程
		#  info all
	 	```
	 	
-  配置extract
	 	
	```
 	 # edit param ext_wpkg  
 	
 	extract ext_wpkg
	setenv (MYSQL_HOME="/var/lib/mysql")
	tranlogoptions altlogdest /var/lib/mysql/mysql-bin.index
	dboptions host localhost,connectionport 3306
	sourcedb test, userid root,password 123456
	exttrail /opt/ggs/dirdat/W3
	dynamicresolution
	gettruncates
	GETUPDATEBEFORES
	NOCOMPRESSDELETES
	NOCOMPRESSUPDATES
	table test.wms_test;
	```
	 	
 	```
 	#      ADD EXTRACT ext_wpkg, tranlog,begin now
	#      ADD EXTTRAIL /opt/ggs/dirdat/W3, EXTRACT ext_wpkg
 	```
	 	
-  配置pump
	
	```
	# edit param pum_wpkg  
	
	extract pum_wpkg
	rmthost yp-data01,mgrport 17809
	rmttrail /opt/ggs/dirdat/W3
	passthru
	gettruncates
	table test.wms_test;
	```	 	
	
	```
	 # ADD EXTRACT pum_wpkg,EXTTRAILSOURCE /opt/ggs/dirdat/W3;
	 # ADD RMTTRAIL /opt/ggs/dirdat/W3, EXTRACT pum_wpkg
	```

`备注：/opt/ggs/dirdat目标端ogg路径数据存放路径`

-  配置defgen

	```
	# edit param defgen_wpkg
	defsfile /opt/ggs/dirdef/defgen_wpkg.prm
	sourcedb test@localhost:3306,userid root,password 123456
	table test.wms_entry_warehouse_wpkg;
	
	```

	
	*备注：用于生成表字段映射*

- 生成defgen表字段映射
 进入ogg根目录
 
 ```
 ./defgen paramfile /opt/ggs/dirprm/defgen_wpkg.prm
 ```
	 *备注：拷贝dirdef/defgen_wpkg.prm文件到目标端dirdef/目录下*
 
-  启动extract和pump

	```
	#      start ext_wpkg
	#      start pum_wpkg
	#      info all
	``` 
	
- 查看EXTRACT进程统计信息

	```
	# stats EXT_WPKG
	
	
	Sending STATS request to EXTRACT EXT_WPKG ...

	Start of Statistics at 2017-12-12 14:55:43.
	
	Output to /opt/ggs/dirdat/W3:
	
	Extracting from test.wms_test to test.wms_test:
	
	*** Total statistics since 2017-12-12 11:08:40 ***
		Total inserts                   	           1.00
		Total updates                   	           1.00
		Total befores                   	           1.00
		Total deletes                   	           1.00
		Total discards                  	           0.00
		Total operations                	           3.00

	*** Daily statistics since 2017-12-12 11:08:40 ***
		Total inserts                   	           1.00
		Total updates                   	           1.00
		Total befores                   	           1.00
		Total deletes                   	           1.00
		Total discards                  	           0.00
		Total operations                	           3.00


	```
	
- 查看PUMP进程统计信息
	
	```
	Sending STATS request to EXTRACT PMP ...

	Start of Statistics at 2017-12-12 15:00:05.
	
	Output to /opt/ggs/dirdat/W3:
	
	Extracting from test.wms_test to test.wms_test:
	
	*** Total statistics since 2017-12-12 11:08:43 ***
		Total inserts                   	           1.00
		Total updates                   	           1.00
		Total befores                   	           1.00
		Total deletes                   	           1.00
		Total discards                  	           0.00
		Total operations                	           3.00

	*** Daily statistics since 2017-12-12 11:08:43 ***
		Total inserts                   	           1.00
		Total updates                   	           1.00
		Total befores                   	           1.00
		Total deletes                   	           1.00
		Total discards                  	           0.00
		Total operations                	           3.00
			
	```
	
	

#### goldengate配置

- 登录ggs
	
	```
	$GGS_HOME/ggsci
	```
- 创建目录




### 目标端配置

- `$GGS_HOME/ggsci`

- `CREATE SUBDIRS` 创建目录

- 配置mgr

	```
	# edit param mgr 
	
	PORT 17809
	DYNAMICPORTLIST 17810-17909
	AUTORESTART EXTRACT *,RETRIES 5,WAITMINUTES 3
	PURGEOLDEXTRACTS /opt/ggs/dirdat/*,usecheckpoints, minkeepdays 3
	
	```
- 配置REPLICAT

	```
	# edit param REP_WPKG
	
	REPLICAT rep_wpkg
	TARGETDB LIBFILE libggjava.so SET property=dirprm/kafka.props
	REPORTCOUNT EVERY 1 MINUTES, RATE
	GROUPTRANSOPS 10000
	MAP test.*,TARGET test.*;
	```


	```
	add replicat rep_wpkg, exttrail /opt/ggs/dirdat/W3,begin now
	```
	
- 配置kafka

	- `dirprm/kafka.props`
	
		```
		gg.handlerlist = kafkahandler
		gg.handler.kafkahandler.type=kafka
		gg.handler.kafkahandler.KafkaProducerConfigFile=custom_kafka_producer.properties
		#The following resolves the topic name using the short table name
		gg.handler.kafkahandler.topicMappingTemplate=${tableName}
		#The following selects the message key using the concatenated primary keys
		gg.handler.kafkahandler.keyMappingTemplate=${primaryKeys}
		gg.handler.kafkahandler.format=json_row
		gg.handler.kafkahandler.SchemaTopicName=mySchemaTopic
		gg.handler.kafkahandler.BlockingSend =false
		gg.handler.kafkahandler.includeTokens=false
		gg.handler.kafkahandler.mode=op
	
	
		goldengate.userexit.timestamp=utc
		goldengate.userexit.writers=javawriter
		javawriter.stats.display=TRUE
		javawriter.stats.full=TRUE
		
		gg.log=log4j
		gg.log.level=INFO
		
		gg.report.time=30sec
		
		#Sample gg.classpath for Apache Kafka
		gg.classpath=dirprm/:/opt/kafka_2.10-0.10.0.1/libs/*
		#Sample gg.classpath for HDP
		#gg.classpath=/etc/kafka/conf:/usr/hdp/current/kafka-broker/libs/*
		
		javawriter.bootoptions=-Xmx512m -Xms32m -Djava.class.path=/opt/ggs/ggjava/ggjava.jar
		```
		
		- gg.handler.kafkahandler.topicMappingTemplate  kafka topic名称的映射，指定topic名称，也可以通过占位符的方式，例如`${tableName}`,每一张表对应一个topic
		- gg.handler.kafkahandler.SchemaTopicName 表的Schema信息对应的topic名称
		
	- 配置kafka连接信息`custom_kafka_producer.properties`
		
		```
		bootstrap.servers=localhost:9092
		acks=1
		reconnect.backoff.ms=1000
		
		value.serializer=org.apache.kafka.common.serialization.ByteArraySerializer
		key.serializer=org.apache.kafka.common.serialization.ByteArraySerializer
		# 100KB per partition
		batch.size=16384
		linger.ms=0
		
		```
	
		
- 启动mgr/replicat

	```
	start mgr 
	start replicat REP_WPKG
	```
	
- 查看replicat统计信息
	
	 ```
	 # stats REP_WPKG
	 
	 	Sending STATS request to REPLICAT REP_WPKG ...

		Start of Statistics at 2017-12-12 15:02:14.
		
		Replicating from test.wms_test to test.wms_test:
		
		*** Total statistics since 2017-12-12 11:10:41 ***
			Total inserts                   	           1.00
			Total updates                   	           1.00
			Total deletes                   	           1.00
			Total discards                  	           0.00
			Total operations                	           3.00
		
		*** Daily statistics since 2017-12-12 11:10:41 ***
			Total inserts                   	           1.00
			Total updates                   	           1.00
			Total deletes                   	           1.00
			Total discards                  	           0.00
			Total operations                	           3.00

	 ``` 
- 启动kafka查看消息
	
	kafka上会创建两个topic
	- mySchemaTopic  接收Schema信息的topic
	
		- 从kafka接收Schema信息
		
		```
		 # kafka-console-consumer.sh --zookeeper localhost:2181 --topic  mySchemaTopic  --from-beginning
		 
		 {
		    "$schema":"http://json-schema.org/draft-04/schema#",
		    "title":"TEST.WMS_TEST",
		    "description":"JSON schema for table TEST.WMS_TEST",
		    "definitions":{
		        "tokens":{
		            "type":"object",
		            "description":"Token keys and values are free form key value pairs.",
		            "properties":{
		
		            },
		            "additionalProperties":true
		        }
		    },
		    "type":"object",
		    "properties":{
		        "table":{
		            "description":"The fully qualified table name",
		            "type":"string"
		        },
		        "op_type":{
		            "description":"The operation type",
		            "type":"string"
		        },
		        "op_ts":{
		            "description":"The operation timestamp",
		            "type":"string"
		        },
		        "current_ts":{
		            "description":"The current processing timestamp",
		            "type":"string"
		        },
		        "pos":{
		            "description":"The position of the operation in the data source",
		            "type":"string"
		        },
		        "primary_keys":{
		            "description":"Array of the primary key column names.",
		            "type":"array",
		            "items":{
		                "type":"string"
		            },
		            "minItems":0,
		            "uniqueItems":true
		        },
		        "tokens":{
		            "$ref":"#/definitions/tokens"
		        },
		        "id":{
		            "type":[
		                "integer",
		                "null"
		            ]
		        },
		        "first_name":{
		            "type":[
		                "string",
		                "null"
		            ]
		        },
		        "last_name":{
		            "type":[
		                "string",
		                "null"
		            ]
		        },
		        "sex":{
		            "type":[
		                "string",
		                "null"
		            ]
		        },
		        "address":{
		            "type":[
		                "string",
		                "null"
		            ]
		        },
		        "flag":{
		            "type":[
		                "integer",
		                "null"
		            ]
		        }
		    },
		    "required":[
		        "table",
		        "op_type",
		        "op_ts",
		        "current_ts",
		        "pos"
		    ],
		    "additionalProperties":false
		}
		```
		
	
	- wms_test  每个table对应一个topic,也可以指定多个table对应一个topic
		
		- 从topic读取事务日志
		
			```
			kafka-console-consumer.sh --zookeeper localhost:2181 --topic wms_test  --from-beginning
			```
			
			- 新增一条记录
			
			```
			INSERT INTO `test`.`wms_test` (`first_name`, `last_name`, `sex`, `address`, `flag`) VALUES ('a', 'b', 'c', 'd', '1');
			
			
			#kafka接收到消息格式
			
			{
			    "table":"TEST.WMS_TEST",
			    "op_type":"I",
			    "op_ts":"2017-12-12 09:47:44.349517",
			    "current_ts":"2017-12-12T17:47:50.415000",
			    "pos":"00000000000000003880",
			    "id":16,
			    "first_name":"a",
			    "last_name":"b",
			    "sex":"c",
			    "address":"d",
			    "flag":1
			}

			``` 	 
			
			- 修改一条记录

			```
			UPDATE `test`.`wms_test` SET `flag`='2' WHERE `id`='16';
			
			#kafka接收到消息格式
			{
			    "table":"TEST.WMS_TEST",
			    "op_type":"U",
			    "op_ts":"2017-12-12 09:49:27.349051",
			    "current_ts":"2017-12-12T17:49:32.457000",
			    "pos":"00000000000000004268",
			    "id":16,
			    "first_name":"a",
			    "last_name":"b",
			    "sex":"c",
			    "address":"d",
			    "flag":2
			}

			```
			
			- 删除一条记录
			
			```
			# DELETE FROM `test`.`wms_test` WHERE `id`='16';
			
			#kafka接收到消息格式
			{
			    "table":"TEST.WMS_TEST",
			    "op_type":"D",
			    "op_ts":"2017-12-12 09:51:00.348680",
			    "current_ts":"2017-12-12T17:51:06.491000",
			    "pos":"00000000000000004376",
			    "id":16,
			    "first_name":"a",
			    "last_name":"b",
			    "sex":"c",
			    "address":"d",
			    "flag":2
			}

			```
	
	
	




### 问题列表

- `Error loading Java VM runtime library: (2 No such file or directory)`
	
	- 解决方式
	
		在环境变量中添加如下`LD_LIBRARY_PATH`
	
		```
		export LD_LIBRARY_PATH=$JAVA_HOME/jre/lib/amd64:$JAVA_HOME/jre/lib/amd64/server:$JAVA_HOME/jre/lib/amd64/libjsig.so:$JAVA_HOME/jre/lib/amd64/server/libjvm.so
		
		export LD_LIBRARY_PATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib/amd64/server
		```
	 
- db2安装报错 `/ggsci: error while loading shared libraries: libdb2.so.1: cannot open shared object file: No such file or directory`
	
	- 解决方式
	 
	添加db2的lib依赖
	 
	```
	export LD_LIBRARY_PATH=$DB2_HOME/lib64/:$LD_LIBRARY_PATH
	
	``` 
	
### kafka常用命令

- 启动服务
	
	```
	zookeeper-server-start.sh config/zookeeper.properties &
	```
- 启动Kafka

	```
	bin/kafka-server-start.sh config/server.properties &
	```

- 创建队列
	
	```
	kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
	```  
	
- 运行producer

	```
	in/kafka-console-producer.sh --broker-list localhost:9092 --topic test  
	```
	
- 运行consumer

	```
	bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning  
	```
- 查看创建的topic
	
	```
	kafka-topics.sh --list --zookeeper localhost:2181
	```
	
### 参考资料
- [ Introducing the Java Adapter](https://docs.oracle.com/goldengate/bd1221/gg-bd/GBDIN/GUID-2D084859-E83F-44A7-B3E6-1F24798862E9.htm#GBDIN111)
- [https://docs.oracle.com/goldengate/bd123110/gg-bd/GADBD/using-kafka-handler.htm#GADBD451](https://docs.oracle.com/goldengate/bd123110/gg-bd/GADBD/using-kafka-handler.htm#GADBD451)
- [深入理解 Oracle GoldenGate 检查点机制](http://www.aiuxian.com/article/p-1508684.html)
- [GoldenGate常用操作](http://www.traveldba.com/archives/777)
- [Oracle goldengate 实现mysql到kafka同步配置](http://news.seecoder.com/news/detail/2703)
- [https://blogs.oracle.com/dataintegration/goldengate-for-big-data-123-is-released](https://blogs.oracle.com/dataintegration/goldengate-for-big-data-123-is-released)
- [http://dbaoracle4hire.blogspot.com/2017/08/nueva-version-de-oracle-goldengate-para.html](http://dbaoracle4hire.blogspot.com/2017/08/nueva-version-de-oracle-goldengate-para.html)
- [Learn GoldenGate](http://www.vitalsofttech.com/goldengate-extract-pump-replicat-ggsci-logdump/)
- [GoldenGate 基础架构](https://www.cnblogs.com/polestar/p/4252123.html)
- [ Oracle GoldenGate for db2 ](https://docs.oracle.com/goldengate/c1221/gg-winux/GIDBL/GUID-03895C0F-8754-4FDA-89DD-663DA95F0DD8.htm#GIDBL165)