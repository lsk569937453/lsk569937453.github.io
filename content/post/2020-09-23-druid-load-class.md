---
title: Druid getConnection stuck

date: 2020-09-23T07:07:07+01:00

description: Druid getConnection stuck under high concurrency

categories:
 - tutorial  
 
tags:
 - Druid  
 - Mysql  
 - Java  
  
---
## Background
  The business side can normally access mysql through druid when the concurrency is low, and the access to mysql will be stuck after the concurrency is up.
  With [Arthas](https://arthas.aliyun.com/doc/en/quick-start.html) Check out the thread
  
  ![Runable](https://lsk569937453.github.io/assets/images/druid-2020-09-23.png)
It can be seen that two of the four threads we use to consume MQ have been Blocked, and we are using
```
thread threadId -b 
```
See where exactly our thread is stuck. Here is our thread stack:
```
"ConsumeMessageThread_2" Id=73 BLOCKED on java.lang.Object@491bf991 owned by "ConsumeMessageThread_4" Id=75
    at java.lang.ClassLoader.loadClass(ClassLoader.java:404)
    -  blocked on java.lang.Object@491bf991
    at org.springframework.boot.loader.LaunchedURLClassLoader.loadClass(LaunchedURLClassLoader.java:93)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    at com.alibaba.druid.util.Utils.loadClass(Utils.java:211)
    at com.alibaba.druid.util.MySqlUtils.getLastPacketReceivedTimeMs(MySqlUtils.java:351)
    at com.alibaba.druid.pool.DruidAbstractDataSource.testConnectionInternal(DruidAbstractDataSource.java:1407)
    at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:1268)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1235)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1225)
    at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:90)
```
## Source code analysis
The code for 1407 is below,

```
 if (valid && isMySql)  // unexcepted branch
                    long lastPacketReceivedTimeMs = MySqlUtils.getLastPacketReceivedTimeMs(conn);
```
Continue to follow up and take a look at the method getLastPacketReceivedTimeMs
```
public static long getLastPacketReceivedTimeMs(Connection conn) throws SQLException {
        if (class_connectionImpl == null && !class_connectionImpl_Error) {
            try {
                class_connectionImpl = Utils.loadClass("com.mysql.jdbc.MySQLConnection");
            } catch (Throwable error){
                class_connectionImpl_Error = true;
            }
        }
```

It can be seen that if you want to enter this Utils.loadClass, you must have class_connectionImpl==null, and class_connectionImpl_Error is false. I followed up again and found  Utils.loadClass always returns null, so the logic of getLastPacketReceivedTimeMs is entered every time. Why is it null every time?
Continue to follow up and take a look at the method Utils.loadClass
```
public static Class<?> loadClass(String className) {
        Class<?> clazz = null;

        if (className == null) {
            return null;
        }

        try {
            return Class.forName(className);
        } catch (ClassNotFoundException e) {
            // skip
        }

        ClassLoader ctxClassLoader = Thread.currentThread().getContextClassLoader();
        if (ctxClassLoader != null) {
            try {
                clazz = ctxClassLoader.loadClass(className);
            } catch (ClassNotFoundException e) {
                // skip
            }
        }

        return clazz;
    }
```
It can be seen that if the class is not found under the current package, the exception will be swallowed, and the method will return empty. As we all know, there are two mysql driver classes:

```
   mysql-connector-java 5:com.mysql.jdbc.Driver   
   
   mysql-connector-java 6:com.mysql.cj.jdbc.Driver  
```
The driver com.mysql.cj.jdbc.Driver is used in our program, so the class com.mysql.jdbc.MySQLConnection cannot be loaded all the time. Every time you get a connection, you will enter this method to loadClass.

The loadClass method is inherently synchronous, so after the concurrency comes up, the performance will drop sharply. So the solution is to change the className of the driver and the driver jar package, that is, downgrade to com.mysql.jdbc.Driver
this driver.


## Solution
mysql-driver-class: com.mysql.jdbc.Driver 
The driver jar can be replaced with the following driver
```
<dependency>
	<groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
	<version>5.1.49</version>
</dependency>
```