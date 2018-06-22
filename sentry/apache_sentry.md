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
	 GRANT ALL ON DATABASE db3 TO ROLE etl;
	 
	 
	```
	
- 修改 hive-site.xml文件,关掉 `HiveServer2 impersonation`



- 创建用户并赋权

	```
	 jdbc:hive2://10.205.58.36:10000> CREATE ROLE admin;
	 jdbc:hive2://10.205.58.36:10000> GRANT ROLE admin TO GROUP hive;
	 jdbc:hive2://10.205.58.36:10000> GRANT ALL ON server SentryHostname to role admin;
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