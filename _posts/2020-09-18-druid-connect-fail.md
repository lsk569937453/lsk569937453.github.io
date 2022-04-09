---
title: Druid unable to connect to mysql database after failback
description: After the mysql machine fails to recover, druid needs to be restarted to return to normal
categories:
 - tutorial
tags:
- Mysql
- Java
---
## Problem Description
**The bottom layer of the application connects to the mysql database through druid. When the fault occurs, the mysql process is abnormal. The on-duty operation and maintenance executes the mysql process restart and the network card reset.
Waiting for the operation, it is found that only half of the 8 machines on the application side have returned to normal, and the other machines need to restart the application to return to normal.**  
The normal machine reports the following error before recovery：
```
... 101 more\nCaused by: com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 3000, active 1, maxActive 100, creating 1
	at com.alibaba.druid.pool.DruidDataSource.getConnectionInternal(DruidDataSource.java:1510)
	at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:1255)
	at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1235)
	at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1225)
	at com.ly.dal.manager.TransactionContextManager.doGetRealConnection(TransactionContextManager.java:319)
	... 107 more\nCaused by: com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Could not create connection to database server. Attempted reconnect 3 times. Giving up.
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:404)
	at com.mysql.jdbc.Util.getInstance(Util.java:387)
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:917)
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:896)
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:885)
	at com.mysql.jdbc.SQLError.createSQLException(SQLError.java:860)
	at com.mysql.jdbc.ConnectionImpl.connectWithRetries(ConnectionImpl.java:2165)
	at com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2090)
	at com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:795)
	at com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:44)
	at sun.reflect.GeneratedConstructorAccessor84.newInstance(Unknown Source)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:404)
	at com.mysql.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:400)
	at com.mysql.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:327)
	at com.alibaba.druid.pool.DruidAbstractDataSource.createPhysicalConnection(DruidAbstractDataSource.java:1558)
	at com.alibaba.druid.pool.DruidAbstractDataSource.createPhysicalConnection(DruidAbstractDataSource.java:1623)
	at com.alibaba.druid.pool.DruidDataSource$CreateConnectionThread.run(DruidDataSource.java:2468)\nCaused by: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
	at sun.reflect.GeneratedConstructorAccessor115.newInstance(Unknown Source)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
	at com.mysql.jdbc.Util.handleNewInstance(Util.java:404)
	at com.mysql.jdbc.SQLError.createCommunicationsException(SQLError.java:981)
	at com.mysql.jdbc.MysqlIO.readPacket(MysqlIO.java:628)
	at com.mysql.jdbc.MysqlIO.doHandshake(MysqlIO.java:1014)
	at com.mysql.jdbc.ConnectionImpl.coreConnect(ConnectionImpl.java:2255)
	at com.mysql.jdbc.ConnectionImpl.connectWithRetries(ConnectionImpl.java:2106)
	... 12 more\nCaused by: java.io.EOFException: Can not read response from server. Expected to read 4 bytes, read 0 bytes before connection was unexpectedly lost.
	at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2957)
	at com.mysql.jdbc.MysqlIO.readPacket(MysqlIO.java:560)
	... 15 more
```
The abnormal machine has been reporting the following error:
```
Caused by: com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 3000, active 1, maxActive 100, creating 1
at com.alibaba.druid.pool.DruidDataSource.getConnectionInternal(DruidDataSource.java:1512)
at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:1255)
at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1235)
at com.alibaba.druid.pool.DruidDataSource.getConnection(DruidDataSource.java:1225)
at com.ly.dal.manager.TransactionContextManager.doGetRealConnection(TransactionContextManager.java:319)
... 101 more
```

The difference is：**The faulty machine has no socket exceptions. Whether it is a connection failure or a connection reset, an exception will be thrown, but the faulty machine does not.**
Since the business side restarted the application directly, there was no scene, so I searched the Internet to see if anyone else encountered this kind of error, and found a similar article
[blog](https://www.jianshu.com/p/3c85c9ddffd3)  。  
## Problem recurrence
The normal application server connects to mysql through druid, and the reproduction scene is as follows:
1.disconnected. Disconnect the application and mysql from the Internet, and then reconnect after a period of time, you can connect
2.Restart the mysql process.
3.The network card of the mysql machine is restarted.

None of the above three methods can reproduce the problem.
At this time, only the Chaos Tool of Ali, the big killer, can be sent.[chaosBlade](https://github.com/chaosblade-io/chaosblade)  。  
chaosBladeIt can simulate network disconnection operation, packet loss rate, network card packet damage, etc.[Tuturial](https://chaosblade-io.gitbook.io/chaosblade-help-zh-cn/blade-create-network-corrupt)  。  
I tried a lot of methods at the time, and finally when the packet loss rate was set to 90%, the error was reproduced. After we returned the packet loss rate to normal, the application did not return to normal.
```
blade create network loss --percent 90 --interface eth0 --remote-port 3022
```
Since I did the chaos experiment on the application machine, the packet loss rate of the remote port is set here, and 3022 is the port of my mysql machine.
Then we are using [arthas](https://github.com/alibaba/arthas/blob/master/README_CN.md)  Look at the specific thread status.
<!--![RUNOOB](../assets/images/druid-1.png) -->
![RUNOOB](https://lsk569937453.github.io/assets/images/druid-1.png)
You can see that there are two threads of Druid-ConnectionPool-Create, because we have two dbs that need to be connected, so there are two druid connection pools, so there are two threads that create connections.
  

Take a look at the thread stack information of the two threads:
<!--![RUNOOB](../assets/images/druid-2.png) -->
![RUNOOB](https://lsk569937453.github.io/assets/images/druid-2.png)

The first thread is the thread in question, you can see that the current thread state is Runnable. This is what the java doc says about the Runnable state


**A thread in the runnable state is executing in the Java virtual machine, but it may be waiting for other resources from the operating system, such as a processor.**  

That is, the current thread is running, always executing socket.read. We all know that socket is the implementation of the tcp protocol stack. In the case of abnormal shutdown of the server, the fin packet/reset packet will be sent, and the client will only close after receiving these two types of packets. Then the problem must be a malfunction



The second thread is a normal thread, waiting until a signal wakes it up to create a connection.

## Source code analysis

```
public class CreateConnectionThread extends Thread {

        public CreateConnectionThread(String name){
            super(name);
            this.setDaemon(true);
        }

        public void run() {
            initedLatch.countDown();

            long lastDiscardCount = 0;
            int errorCount = 0;
            for (;;) {
                // addLast
                try {
                    lock.lockInterruptibly();
                } catch (InterruptedException e2) {
                    break;
                }

                long discardCount = DruidDataSource.this.discardCount;
                boolean discardChanged = discardCount - lastDiscardCount > 0;
                lastDiscardCount = discardCount;

                try {
                    boolean emptyWait = true;

                    if (createError != null
                            && poolingCount == 0
                            && !discardChanged) {
                        emptyWait = false;
                    }

                    if (emptyWait
                            && asyncInit && createCount < initialSize) {
                        emptyWait = false;
                    }

                    if (emptyWait) {
                        // There must be a thread waiting to create a connection
                        if (poolingCount >= notEmptyWaitThreadCount //
                                && !(keepAlive && activeCount + poolingCount < minIdle)) {
                            empty.await();
                        }

                       // Prevent the creation of more than maxActive connections
                        if (activeCount + poolingCount >= maxActive) {
                            empty.await();
                            continue;
                        }
                    }

                } catch (InterruptedException e) {
                    lastCreateError = e;
                    lastErrorTimeMillis = System.currentTimeMillis();

                    if (!closing) {
                        LOG.error("create connection Thread Interrupted, url: " + jdbcUrl, e);
                    }
                    break;
                } finally {
                    lock.unlock();
                }

                PhysicalConnectionInfo connection = null;

                try {
                    connection = createPhysicalConnection();
                    setFailContinuous(false);
                } catch (SQLException e) {
                    LOG.error("create connection SQLException, url: " + jdbcUrl + ", errorCode " + e.getErrorCode()
                              + ", state " + e.getSQLState(), e);

                    errorCount++;
                    if (errorCount > connectionErrorRetryAttempts && timeBetweenConnectErrorMillis > 0) {
                        // fail over retry attempts
                        setFailContinuous(true);
                        if (failFast) {
                            lock.lock();
                            try {
                                notEmpty.signalAll();
                            } finally {
                                lock.unlock();
                            }
                        }

                        if (breakAfterAcquireFailure) {
                            break;
                        }

                        try {
                            Thread.sleep(timeBetweenConnectErrorMillis);
                        } catch (InterruptedException interruptEx) {
                            break;
                        }
                    }
                } catch (RuntimeException e) {
                    LOG.error("create connection RuntimeException", e);
                    setFailContinuous(true);
                    continue;
                } catch (Error e) {
                    LOG.error("create connection Error", e);
                    setFailContinuous(true);
                    break;
                }

                if (connection == null) {
                    continue;
                }

                boolean result = put(connection);
                if (!result) {
                    JdbcUtils.close(connection.getPhysicalConnection());
                    LOG.info("put physical connection to pool failed.");
                }

                errorCount = 0; // reset errorCount
            }
        }
    }
```
As you can see from the source code of CreateConnectionThread, in order to avoid opening a new thread/thread pool when creating a connection, an infinite loop is used to do this. If there is a signal to wake it up, it will create a new thread. After the creation, it enters the next loop, that is, waiting for a new signal to wake it up to create a connection.

Our program must be restarted to solve the problem, because CreateConnectionThread is stuck in the step of creating a connection. You can see the socket.read step from the reproduction of the problem to see that our program has been stuck here. So the question is don't we set a timeout when creating the connection? The answer is indeed not set.

```
jdbc:mysql://xx.xx.xx.xx:3027/xx?useUnicode=true&characterEncoding=utf8&autoReconnect=true
```
## Solution
Set it when setting up the druid connection pool **connectTimeout,socketTimeout**。
