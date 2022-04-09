---
title: The connection pool(Mysql) is full and cannot be released

description: The application connects to the database middleware, and the database middleware connects to mysql. After mysql fails, it returns to normal, but the business side cannot recover.

categories:
 - tutorial
tags:
- Mysql
- Java

---
## Problem Description
**The application connects to the database middleware, and the database middleware connects to mysql. After mysql fails, it returns to normal, but the business side cannot recover. The error is as follows**   
```
### Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection; nested exception is org.apache.tomcat.jdbc.pool.PoolExhaustedException: [Thread-1391] Timeout: Pool empty. Unable to fetch a connection in 0 seconds, none available[size:30; busy:30; idle:0; lastwait:20].
	at org.apache.ibatis.exceptions.ExceptionFactory.wrapException(ExceptionFactory.java:23)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:107)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:98)
	at com.elong.dda.test.test.MybatisTest$1.run(MybatisTest.java:71)
	at java.lang.Thread.run(Thread.java:748)
Caused by: org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection; nested exception is org.apache.tomcat.jdbc.pool.PoolExhaustedException: [Thread-1391] Timeout: Pool empty. Unable to fetch a connection in 0 seconds, none available[size:30; busy:30; idle:0; lastwait:20].
	at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:80)
	at org.mybatis.spring.transaction.SpringManagedTransaction.openConnection(SpringManagedTransaction.java:80)
	at org.mybatis.spring.transaction.SpringManagedTransaction.getConnection(SpringManagedTransaction.java:66)
	at org.apache.ibatis.executor.BaseExecutor.getConnection(BaseExecutor.java:271)
	at org.apache.ibatis.executor.SimpleExecutor.prepareStatement(SimpleExecutor.java:69)
	at org.apache.ibatis.executor.SimpleExecutor.doQuery(SimpleExecutor.java:56)
	at org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(BaseExecutor.java:259)
	at org.apache.ibatis.executor.BaseExecutor.query(BaseExecutor.java:132)
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:105)
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:81)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:104)
	... 3 more
Caused by: org.apache.tomcat.jdbc.pool.PoolExhaustedException: [Thread-1391] Timeout: Pool empty. Unable to fetch a connection in 0 seconds, none available[size:30; busy:30; idle:0; lastwait:20].
	at org.apache.tomcat.jdbc.pool.ConnectionPool.borrowConnection(ConnectionPool.java:674)
	at org.apache.tomcat.jdbc.pool.ConnectionPool.getConnection(ConnectionPool.java:188)
	at org.apache.tomcat.jdbc.pool.DataSourceProxy.getConnection(DataSourceProxy.java:128)
	at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:111)
	at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:77)
	... 13 more
``` 
## Solution  
Set param when building the connection pool
**removeAbandoned,removeAbandonedTimeout**。
``` 
#Whether to automatically recycle timeout connections
dataSource.removeAbandoned=true 

#timeout (in seconds)
dataSource.removeAbandonedTimeout=180
``` 