---
title: chaosBlade的相关指令
description: blade的相关指令
categories:
 - tutorial
tags:
- Chaos
---
## 模拟网络故障
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