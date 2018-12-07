# Hadoop 权限


## Sentry


## Hive


- beeline连接Hive

	```
	beeline> !connect jdbc:hive2://datanode03:10000/default 
	scan complete in 1ms
	Connecting to jdbc:hive2://datanode03:10000/default
	Enter username for jdbc:hive2://datanode03:10000/default: hive
	Enter password for jdbc:hive2://datanode03:10000/default: ****
	```
	用户名和密码：`hive/hive`


	```
	beeline --verbose=true -u "jdbc:hive2://data01:10000/default;principal=hive/_HOST@HADOOP.COM"
	```


	```
	 GRANT ALL ON DATABASE db3 TO ROLE etl;
	 
	 
	```
	
- 修改 hive-site.xml文件,关掉 `HiveServer2 impersonation`



- 创建用户并赋权

	```
	 jdbc:hive2://10.205.58.36:10000> CREATE ROLE admin;
	 jdbc:hive2://10.205.58.36:10000> GRANT ROLE admin TO GROUP hive;
	 jdbc:hive2://10.205.58.36:10000> GRANT ALL ON server server1 to role admin;
	 jdbc:hive2://10.205.58.36:10000> 
	 jdbc:hive2://10.205.58.36:10000> CREATE ROLE etl； 
	 jdbc:hive2://10.205.58.36:10000> GRANT ROLE etl TO GROUP etl;
	 jdbc:hive2://10.205.58.36:10000>GRANT SELECT ON DATABASE app TO ROLE etl;GRANT SELECT ON DATABASE web TO ROLE etl;
......
	```
	
	```
	[databases]
	# Defines the location of the per DB policy file for the customers DB/schema
	#db1 = hdfs://cdh1:8020/user/hive/sentry/db1.ini
	
	[groups]
	admin = any_operation
	hive = any_operation
	test = select_filtered
	
	[roles]
	any_operation = server=server1->db=*->table=*->action=*
	select_filtered = server=server1->db=filtered->table=*->action=SELECT
	select_us = server=server1->db=filtered->table=events_usonly->action=SELECT
	
	[users]
	test = test
	hive= hive
	
	$ hdfs dfs -rm -r /user/hive/sentry/sentry-provider.ini
	$ hdfs dfs -put /tmp/sentry-provider.ini /user/hive/sentry/
	$ hdfs dfs -chown hive:hive /user/hive/sentry/sentry-provider.ini
	$ hdfs dfs -chmod 640 /user/hive/sentry/sentry-provider.ini
	```

### Hive SQL Syntax for Use with Sentry

- 创建和删除角色
	- 创建角色: create role ROLE_NAME
	- 删除角色: droop role ROLE_NAME
	
- 角色的授权和撤销
	
	```
	GRANT ROLE role_name [, role_name] TO GROUP <groupName> [,GROUP <groupName>]
REVOKE ROLE role_name [, role_name] FROM GROUP <groupName> [,GROUP <groupName>]
	``` 	

- 权限的授予和撤销

	```
	GRANT <PRIVILEGE> [, <PRIVILEGE> ] ON <OBJECT> <object_name> TO ROLE <roleName> [,ROLE <roleName>]
REVOKE <PRIVILEGE> [, <PRIVILEGE> ] ON <OBJECT> <object_name> FROM ROLE <roleName> [,ROLE <roleName>]
	```
	
-  查看角色/组权限

	```
	SHOW ROLES;
SHOW CURRENT ROLES;
SHOW ROLE GRANT GROUP <groupName>;
SHOW GRANT ROLE <roleName>;
SHOW GRANT ROLE <roleName> on OBJECT <objectName>;
	```

## Hbase权限控制

- HBase grant permission

|HBase shell Commands          		|	Description|
|-----------------------------------|-------------|
|grant 'boopathi', 'RW', 'table'    |	User with this permission can manage data on the specified table only.|
|grant 'boopathi', 'RW', 'namespace:table'|	Granting permission Read and Write permission for user on table, which is present inside namespace. Here you will not give '@' prefix with namesapce.|
|grant 'boopathi', 'RWCA', '@namespace'	|Grant permission for user boopathi on specified 'namespace' only. In this case user can perform all operation on the given namespace.|
|grant 'boopathi', 'RWCA'|	Grant permission for user 'boopathi' with all access globally.|
|grant '@grp-name', 'RWXC', '@namespace'|	Grant permission for groups on specified namespace.|
|grant '@grp-name', 'RWXC'|	Grant permission for groups here. It will be easy to manage, in case of groups. This is given on global scope.|
|grant '@grp-name', 'RW', 'namespace:table'|Grant permission for group on table in namespace.|


- HBase get permission details

|HBase shell Commands          		|	Description|
|-----------------------------------|-------------|
|user_permission						   |List all the user and the permission on the global scope.|
|user_permission '@namespace'	      |List all the user in the specified namespace.|
|user_permission 'namespace:table'  |	List all users, who have permissions on the table in the namespace|
|user_permission 'table'             |	List all the users, who have permission on the table.|

- HBase Revoke Access

|HBase shell Commands          		|	Description|
|-----------------------------------|-------------|
|revoke 'boopathi'   |	Revoke all the access of the user on the global level.|
|revoke 'boopathi', 'table'	|Revoke all the access of the user on the table he has.|
|revoke 'boopathi', '@namespace'	|Revoke permissions on the specified namespace level.|
|revoke 'boopathi', 'namespace:table'|	Revoke permission on table in namespace.|




## Hadoop 



```
curl --negotiate -u wuwx -b ~/cookiejar.txt -c ~/cookiejar.txt -H "Accept: application/json" -H "Content-Type: application/json"   http://datanode03:8088/ws/v1/cluster/delegation-token -d '{"renewer":"data03"}'
```

```
{
    "token":"MAAWYWRtaW4vYWRtaW5ASEFET09QLkNPTQZkYXRhMDMAigFkktHwOooBZLbedDoBAhQ-CKd9LYnqkKQ1b4N37pTlMR_kmBNSTV9ERUxFR0FUSU9OX1RPS0VOAA",
    "renewer":"data03",
    "owner":"admin/admin@HADOOP.COM",
    "kind":"RM_DELEGATION_TOKEN",
    "expiration-time":1531557989434,
    "max-validity":1532076389434
}
```

### 生产环境


- 获取Token

```
curl --negotiate -u wuwx -b ~/cookiejar.txt -c ~/cookiejar.txt -H "Accept: application/json" -H "Content-Type: application/json"   http://name01:8088/ws/v1/cluster/delegation-token -d '{"renewer":"ypdata"}'
```


- Kill Application 

```
curl --negotiate -u ypdata  -XPUT -H "X-Hadoop-Delegation-Token:MgAYeXBkYXRhL2RhdGEwMUBIQURPT1AuQ09NBnlwZGF0YQCKAWSS6JlligFktvUdZQMCFDnJ2voi8nvwJmEGr1oWkAET_71VE1JNX0RFTEVHQVRJT05fVE9LRU4A"  -H "Accept: application/json" -H "Content-Type: application/json"     http://name01:8088/ws/v1/cluster/apps/application_1531472489843_0005/state -d '{"state":"KILLED"}
```



- [Cluster_Delegation_Tokens_API](https://hadoop.apache.org/docs/r2.8.0/hadoop-yarn/hadoop-yarn-site/ResourceManagerRest.html#Cluster_Delegation_Tokens_API)

- [Hadoop Auth, Java HTTP SPNEGO - Examples](https://hadoop.apache.org/docs/r2.8.0/hadoop-auth/Examples.html)



## 异常信息

- `can't be none in non-testing mode `
	- 异常信息
		
		```
		Error: Error while compiling statement: FAILED: InvalidConfigurationException hive.server2.authentication can't be none in non-testing mode (state=42000,code=40000)
		```
		
	- 解决方式
		
		```
		<property>
		  <name>sentry.hive.testing.mode</name>
		  <value>true</value>
		</property>
		```	
	
	
	
	

- `Error: Could not open client transport with JDBC Uri: jdbc:hive2://data01:10000/default: Peer indicated failure: Unsupported mechanism type PLAIN (state=08S01,code=0)`


- 解决方式
 
	 `beeline --verbose=true -u "jdbc:hive2://data01:10000/default;principal=hive/_HOST@HADOOP.COM"`
		
## 参考资料

### hive
- [CDH启用 sentry](https://blog.csdn.net/u010368839/article/details/75313337)

- [Managing the Sentry Service](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_sg_sentry_service.html#concept_bj1_ykt_lr)

- [hive集成sentry](https://my.oschina.net/Yumikio/blog/894556)

- [cloudera documentation](https://www.cloudera.com/documentation/enterprise/latest/topics/cdh_sg_hdfs_ext_acls.html)

- [Apache Sentry Guide](https://www.cloudera.com/documentation/enterprise/5-14-x/topics/sentry.html)

### hbase

- [HBase授权](https://www.alibabacloud.com/help/zh/doc-detail/62705.htm)

- [Securing Access To Your Data](http://hbase.apache.org/book.html#_securing_access_to_your_data)

- [HBase Authorization Cheat sheet](http://boopathi.me/blog/hbase-authorization-cheat-sheet/)

- [How to create a role admin user / priviledge](https://community.cloudera.com/t5/Interactive-Short-cycle-SQL/How-to-create-a-role-admin-user-priviledge/m-p/58721)