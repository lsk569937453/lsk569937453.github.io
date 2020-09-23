---
title: Druid geConnection 卡住  

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
  
##### 禁止访问远端3022端口
```
blade create network drop --remote-port 3022
```
##### 禁止访问本地3022端口
```
blade create network drop --local-port 3022
```
##### 调用远端3022端口时，设置丢包率90
```
blade create network loss --interface eth0 --percent 90 --remote-port 3022
```
##### 网卡的网络延时3s(所有网络，包括ssh)
```
blade create network delay --interface eth0 --time 3000
```

## 模拟io故障
##### 模拟io故障
```
blade create disk burn --write --read  --size 10 --count 1024  --timeout 300
```

## 其他

##### 删除某个实验
```
blade destroy uid
```

##### 查询创建过的混沌实验
```
blade status --type create
```