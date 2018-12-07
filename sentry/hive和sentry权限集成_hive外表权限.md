## hive和sentry权限集成 - hive外表权限


### sentry hdfs acl sync URI privilege

- 在hdfs创建外部表数据存储目录（此操作需要hdfs用户权限），例如：

	```
	hadoop fs -mkdir -p /data/hive/externaldir
	```

- 修改外表目录权限（此操作需要hdfs用户权限）

	```
	hadoop fs -chown -R hive:hive /data/hive/externaldir
	```

- 设置在hdfs参数 `sentry.hdfs.integration.path.prefixes` ，设置完成后重启集群，cdh会同步hdfs配置信息
	
	```
	<property>
     <name>sentry.authorization-provider.hdfs-path-prefixes</name>
	  <value>/user/hive/warehouse,/data/hive/externaldir</value>
   </property>
	```
	
	>
	HDSF启用Sentry ACL同步时，默认同步目录时/user/hive/warehouse（即hive内表目录）

### 测试创建外部表


- 登录hive shell分配外部表uri权限（管理员用户操作）

	```
	 grant all on uri "/data/hive/externaldir/db2.db/tb_ext_part" to role etl;
	```
	
	- `/data/hive/externaldir/db2.db/tb_ext_part` 将读写权限分配给etl角色

- 创建外部表（普通用户，例如：etl用户）
	
	```
	 CREATE  external TABLE db2.tb_ext_part(
  		`id_pay` int,                                    
   		`idcard` string,                                 
   		`no_card` string
 		)  PARTITIONED BY (                                   
   `dt` string) 
   STORED AS ORC  
   LOCATION '/data/hive/externaldir/db2.db/tb_ext_part';
	```

- 测试插入数据

	```
	insert into tb_ext_part PARTITION (dt = '2018-01-01')(id_pay,idcard,no_card) values(1,"1000xxx","10000x");
	
	insert into tb_ext_part PARTITION (dt = '2018-01-02')(id_pay,idcard,no_card) values(2,"1000xxx","10000x");
	```

- 查看数据

	```
	select * from tb_ext_part ;
	+---------------------+---------------------+----------------------+-----------------+--+
    | tb_ext_part.id_pay  | tb_ext_part.idcard  | tb_ext_part.no_card  | tb_ext_part.dt  |
    +---------------------+---------------------+----------------------+-----------------+--+
    | 1                   | 1000xxx             | 10000x               | 2018-01-01      |
    | 2                   | 1000xxx             | 10000x               | 2018-01-02      |
    +---------------------+---------------------+----------------------+-----------------+--+
	
	```
	
- 查看hdfs acl 

	```
	hadoop fs -getfacl  /data/hive/externaldir/db2.db/tb_ext_part
	
	# file: /data/hive/externaldir/db2.db/tb_ext_part
	# owner: hive
	# group: hive
	user::rwx
	group::---
	user:hive:rwx
	group:etl:rwx
	group:hive:rwx
	mask::rwx
	other::--x

	```
	group etl已经具有这个目录的rwx(读、写、可执行权限)，etl用户可以直接通过hdfs访问这个目录：
	
	```
	hadoop fs -ls  /data/hive/externaldir/db2.db/tb_ext_part
	Found 2 items
    drwxrwx--x+  - hive hive          0 2018-12-07 13:50 /data/hive/externaldir/db2.db/tb_ext_part/dt=2018-01-01
    drwxrwx--x+  - hive hive          0 2018-12-07 13:51 /data/hive/externaldir/db2.db/tb_ext_part/dt=2018-01-02
	
	```	
	
	- 删除表后acl权限消失
		
		```
		jdbc:hive2://datanode03:10000/default> drop table tb_ext_part;
		
		hadoop fs -getfacl  /data/hive/externaldir/db2.db/tb_ext_part
		
		# file: /data/hive/externaldir/db2.db/tb_ext_part
		# owner: hive
		# group: hive
		user::rwx
		group::r-x
		other::r-x
	
		```
	

### 触发hdfs acl同步的条件 [官方文档](https://www.cloudera.com/documentation/enterprise/latest/topics/sg_hdfs_sentry_sync.html#acl_changes)


> Prompting HDFS ACL Changes
URIs do not have an impact on the HDFS-Sentry plugin. Therefore, you cannot manage all of your HDFS ACLs with the HDFS-Sentry plugin and you must continue to use standard HDFS ACLs for data outside of Hive.

- HDFS ACL changes are triggered on:
	- Hive DATABASE object LOCATION (HDFS) when a role is granted to the object
	- Hive TABLE object LOCATION (HDFS) when a role is granted to the object
- HDFS ACL changes are not triggered by:
	- Hive URI LOCATION (HDFS) when a role is granted to a URI
	- Hive SERVER object when a role is granted to the object. HDFS ACLs are not updated if a role is assigned to the SERVER. The privileges are inherited by child objects in standard Sentry interactions, but the plugin does not trickle the privileges down.



### 外表路径管路规范

**【创建外表时严格按照此规范创建，这样的目的方便数据管理】**

```
/data/hive/externaldir/<用户名/部门组织名称>/<库名称>/<表名称>
```



### 参考资料

- [Sentry HDFS sync will NOT sync URI privilege](https://www.ericlin.me/2016/07/sentry-hdfs-sync-will-not-sync-uri-privilege/)

- [Synchronizing HDFS ACLs and Sentry Permissions] (https://www.cloudera.com/documentation/enterprise/latest/topics/sg_hdfs_sentry_sync.html)

- [How to Verify that HDFS ACLs are Synching with Sentry
](https://www.cloudera.com/documentation/enterprise/latest/topics/sentry_howto_verify_hdfs_sync.html#sentry_howto_verify_hdfs_acl_sync)




