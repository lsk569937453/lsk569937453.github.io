---
title: Druid getConnection 卡住  

description: 高并发下druid getConnection 卡住

categories:
 - tutorial  
 
tags:
 - Druid  
 - Mysql  
 - Java  
  
---
## 故障复原  
  业务方在低并发的时候能正常通过druid访问mysql,并发上来后访问mysql会出现卡住的情况。
  通过[阿尔萨斯](https://arthas.aliyun.com/doc/en/quick-start.html)   查看一下线程情况   
  ![Runable](https://lsk569937453.github.io/assets/images/druid-2020-09-23.png)
可以看到我们消费MQ用的四个线程有两个已经Blocked了,我们在使用
```
thread threadId -b 
```
查看一下我们的线程究竟是卡在哪里了。下面是我们的线程栈：
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
## 源码分析
1407的代码是这个，
```
 if (valid && isMySql)  // unexcepted branch
                    long lastPacketReceivedTimeMs = MySqlUtils.getLastPacketReceivedTimeMs(conn);
```
继续往下跟进一下，看一下getLastPacketReceivedTimeMs 这个方法
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
可以看到如果要进这个Utils.loadClass的话，必须要class_connectionImpl==null,并且class_connectionImpl_Error为false。我又跟进了一下，发现
Utils.loadClass这行代码一直返回null，所以每次都会进getLastPacketReceivedTimeMs这个逻辑。为什么会每次都为空呢？
继续跟进看一下Utils.loadClass这个方法
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
可以看到如果这个class在当前包下面找不到，异常就会被吞，则该方法返回空。众所周知，mysql的driver class有两个:  
   
   mysql-connector-java 5:com.mysql.jdbc.Driver   
   
   mysql-connector-java 6:com.mysql.cj.jdbc.Driver  

我们程序中使用的是com.mysql.cj.jdbc.Driver这个驱动，从而这个类com.mysql.jdbc.MySQLConnection就一直加载不到。每次获取连接都会进这个方法去loadClass。
而loadClass这个方法天生就是同步的，所以并发上来后，性能就会急剧下降。所以解决方案就是将驱动的className以及驱动jar包换一下即可,即降级到com.mysql.jdbc.Driver
这个驱动。

## 解决方案
mysql-driver-class: com.mysql.jdbc.Driver 
驱动jar换成下面的驱动即可 
```
<dependency>
	<groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
	<version>5.1.49</version>
</dependency>
```