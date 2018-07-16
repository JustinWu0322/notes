# Hadoop集群权限测试的列表
> Hadoop集群集成Kerberos和Sentry需要测试的列表


## Kerberos

| 测试项				|测试结果	|备注|
|:------------------|--------|-----|
|新增/删除principal	|已测试（通过）||
|新增客户端授权/取消授权|已测试（通过）||




## Hadoop

> HDFS、YARN的使用


| 测试项				|测试结果	|备注|
|:------------------|--------|-----|
|kerberos环境运行hadoop mapreduce|已测试（通过）||
|kerberos环境 hadoop rest api|已测试|使用org.springframework.security.kerberos KerberosRestTemplate|
|hadoop acl测试|||
|Synchronizing HDFS ACLs and Sentry Permissions|已测试||
|使用SPNEGO和Kerberos配置浏览器进行身份验证|||
|Java Clinet Kerberos环境下测试|||




## Hive
| 测试项				|测试结果	|备注|
|:------------------|--------|-----|
|Hive on Spark   	|已测试（通过）|data01环境变量紊乱导致，hive on spark 无法启动|
|Hive on MR   		|已测试（通过）||
|Hive Sentry   		|已测试|默认角色admin,拥有超级管理员权限，测试是将admin赋权给ypdata时，ypdata无法获取最高权限，需要查找原因|
|spark/mapreduce程序直接读/user/hive/warehouse测试   |||
|Hive基于角色/组权限测试   |||

## Spark
| 测试项				|测试结果	|备注|
|:------------------|--------|-----|
| Spark集成kerberos测试 |||
| Spark Hive  |已测试||
| Spark直接读hdfs文件测试 |||



## Livy
| 测试项				|测试结果	|备注|
|:------------------|--------|-----|
|kerberos环境下Livy使用   ||目前能跑通|


## Hbase

## Hue




## 问题列表


## 1. rm rest api kill application (kerberos)
 
 > Yarn 启用 HTTP Web 控制台的 Kerberos 身份验证后如何使用rest api kill  application
 
 - 使用`org.springframework.security.kerberos`调用rm rest api kill application 
 	
 	- 依赖要求(Spring Security Kerberos 1.0.1.RELEASE)
		- jdk 1.7+
		- Spring Security 3.2.7.RELEASE
		- Spring Framework 4.1.6.RELEASE
 	
 	- pom 依赖

 		```
 		   <dependency>
            <groupId>org.springframework.security.kerberos</groupId>
            <artifactId>spring-security-kerberos-client</artifactId>
            <version>1.0.0.RC2</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.security.kerberos</groupId>
            <artifactId>spring-security-kerberos-core</artifactId>
            <version>1.0.0.RC2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.3.6</version>
         </dependency>
 		```
	- 实现示例

		```
			import org.json.JSONObject;
			import org.springframework.http.HttpEntity;
			import org.springframework.http.HttpHeaders;
			import org.springframework.http.MediaType;
			import org.springframework.http.ResponseEntity;
			import org.springframework.security.kerberos.client.KerberosRestTemplate;
			import org.springframework.util.LinkedMultiValueMap;
			import org.springframework.util.MultiValueMap;

			public class YarnKerberosApi {
			    public static String KERBEROS_KEYTAB = "/Users/yp-tc-m-7161/yp-tc-m-7161_yp-wuwx.keytab";
			    public static String USER_PRINCIPAL = "yp-tc-m-7161/yp-wuwx@HADOOP.COM";
			
			    public static void main(String[] args) {
			        String rmUrl = "http://datanode03:8088";
			        String appId = "application_1531463687304_0008";
			        KerberosRestTemplate restTemplate = new KerberosRestTemplate(KERBEROS_KEYTAB, USER_PRINCIPAL);
			        String token = getDelegationToken(restTemplate, rmUrl, "ypdata");
			        killApplicationById(restTemplate, rmUrl, appId, token);
			    }
			
			    public static String getDelegationToken(KerberosRestTemplate restTemplate, String rmUrl, String renewer) {
			        String url = String.format("%s/ws/v1/cluster/delegation-token", rmUrl);
			        HttpHeaders headers = new HttpHeaders();
			        headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
			        MultiValueMap<String, String> map = new LinkedMultiValueMap<>();
			        map.add("renewer", renewer);
			        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(map, headers);
			        ResponseEntity<String> response = restTemplate.postForEntity(url, request, String.class);
			        JSONObject jsonObject = new JSONObject(response.getBody());
			        String token = jsonObject.getString("token");
			        System.out.println("token : " + jsonObject.getString("token"));
			        return token;
			    }
			
			    public static void killApplicationById(KerberosRestTemplate restTemplate, String rmUrl, String appId, String token) {
			        String url = String.format("%s/ws/v1/cluster/apps/%s/state", rmUrl, appId);
			        HttpHeaders headers = new HttpHeaders();
			        headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
			        headers.add("X-Hadoop-Delegation-Token", token);
			        MultiValueMap<String, String> map = new LinkedMultiValueMap<>();
			        map.add("state", "KILLED");
			        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(map, headers);
			        restTemplate.put(url, request);
			    }
			
			}		
		```
		
		- 资料

			- [Cluster_Delegation_Tokens_API](https://hadoop.apache.org/docs/r2.8.0/hadoop-yarn/hadoop-yarn-site/ResourceManagerRest.html#Cluster_Delegation_Tokens_API)

			- [Spring-security-kerberos](https://docs.spring.io/spring-security-kerberos/docs/current/reference/html/requirements.html)