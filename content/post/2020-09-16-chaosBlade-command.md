---
title: Instructions of ChaosBlade
description: Instructions of Blade
date: 2020-09-16T07:07:07+01:00
categories:
 - tutorial
tags:
- Chaos
---
## Simulate network failure
##### Forbid access to remote port 3022

``` 
blade create network drop --remote-port 3022
``` 
##### Disable access to local port 3022

``` 
blade create network drop --local-port 3022
``` 
##### When calling the remote port 3022, set the packet loss rate to 90

``` 
blade create network loss --interface eth0 --percent 90 --remote-port 3022
```  
##### The network delay of the network card is 3s (all networks, including ssh)

``` 
blade create network delay --interface eth0 --time 3000
```

## Simulate io failure

##### Simulate io failure

``` 
blade create disk burn --write --read  --size 10 --count 1024  --timeout 300
```

## Other

##### delete an experiment

``` 
blade destroy uid
``` 

##### Querying Chaos Experiments Created

``` 
blade status --type create
``` 