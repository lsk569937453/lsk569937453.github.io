---
title: Go进程panic  

description: Go进程panic

categories:
 - tutorial  
 
tags:
 - Go  
  
---
## 故障复原  
  运行中的进程，偶尔会panic,大概会几个月一次。今天一大早进程又panic了，以前启动进程使用的是
```
  nohup xxxx /dev/null 2>&1
```
可以把/dev/null 可以看作"黑洞". 它等价于一个只写文件. 所有写入它的内容都会永远丢失. 而尝试从它那儿读取内容则什么也读不到.
/dev/null 2>&1则表示吧标准输出和错误输出都放到这个“黑洞”，表示什么也不输出。  
所以控制台的错误没有输入到日志文件里，所以就把这个启动进程的指令替换掉。  


我直接简单粗暴的用下面的指令，进程在控制台的日志就会输入到我们的日志文件里。
```
  nohup xxxx >main.nohup
```
所以进程panic后我们去查看一下main.nohup。

```
panic: invalid connection [recovered]
	panic: interface conversion: string is not error: missing method Error

goroutine 1947932716 [running]:
ddamain/impl.ReturnError(0xc0005206f0)
	/Users/user/xxxx/impl/ErrorUtils.go:16 +0x85
panic(0x9e2540, 0xc0002fdb00)
	/Users/user/computer/go/src/runtime/panic.go:969 +0x175
ddamain/entity.ConverResult(0xc0008fab00, 0xc0002747e0)
	/Users/user/xxxx/ResultConvert.go:69 +0x785
```
## 源码分析
ReturnError的代码如下,
```
 func ReturnError() {
 	if e := recover(); e != nil {
 		errMessage := e.(error).Error() + string(debug.Stack())
 		fmt.Println(errMessage)
 	}
 }
```
而ResultConvert的代码如下,
``` 
defer ReturnError()

if err = rows.Err(); err != nil {
	panic(err.Error())
}
```
在错误panic之前优先通过defer将panic recover能保证进程不会panic，那我们的进程为什么会panic呢？  
具体的问题出在这句代码上
```
e.(error).Error()
```
这个地方使用了golang的强制转换，e转换为error，但是我们ResultConvert中panic(err.Error()),err.Error()，这个是string类型
所以我们在将string类型转为error类型的时候出错了，我们的日志中也显示了报错，
```
panic: interface conversion: string is not error: missing method Error
```

## 解决方案
有问题代码：
```
panic(err.Error())
```
go语言中recover()方法返回一个interface{}类型，由e.(Error)这种把interface类型转为Error类型的，在golang中又称为类型推断。
类型推断是有可能导致进程panic的，所以使用的时候不是很安全。因此官方推荐使用 [ok-idiom](https://golang.org/doc/effective_go.html) 模式。
即代码改为如下：
```
func checkType(i interface{}) string {
	var res string
	switch v := i.(type) {
	case int:
		res = "int type"
	case string:
		return v
	case error:
		res = v.Error()
	default:
		res = "not found type"
	}
	return res
}
func ReturnError() {
	if e := recover(); e != nil {
		prefix := checkType(e)
		errMessage := prefix + string(debug.Stack())
		fmt.Println(errMessage)
	}
}
```