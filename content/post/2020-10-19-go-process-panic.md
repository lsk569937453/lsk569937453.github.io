---
title: Go Process Panic  

description: Go Process Panic
date: 2020-10-19T07:07:07+01:00

categories:
 - tutorial  
 
tags:
 - Go  
  
---
## Background  
  Running processes, occasionally panic, about once every few months. Early this morning, the process paniced again, and the process used to start the process was

```
  nohup xxxx /dev/null 2>&1
```
Think of /dev/null as a "black hole". It is equivalent to a write-only file. Everything written to it is lost forever. Trying to read from it will read nothing.
/dev/null 2>&1 means that the standard output and error output are put into this "black hole", which means nothing is output.
So the console error is not entered in the log file, so the command to start the process is replaced.


I use the following command simply and rudely, and the log of the process in the console will be entered into our log file.

```
  nohup xxxx >main.nohup
```
So after the process panic, let's check main.nohup.

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
## Source code analysis

ReturnError is below,
```
 func ReturnError() {
 	if e := recover(); e != nil {
 		errMessage := e.(error).Error() + string(debug.Stack())
 		fmt.Println(errMessage)
 	}
 }
```
And ResultConvert is below,
``` 
defer ReturnError()

if err = rows.Err(); err != nil {
	panic(err.Error())
}
```
Prior to the error panic, the process of panic recovering through defer can ensure that the process will not panic, so why does our process panic?
The specific problem lies in this code

```
e.(error).Error()
```
This place uses golang's coercion conversion, e is converted to error, but panic(err.Error()), err.Error() in our ResultConvert, this is a string type.

So we made an error when converting the string type to the error type, and our log also showed an error,
```
panic: interface conversion: string is not error: missing method Error
```

## Solution
Problematic code：
```
panic(err.Error())
```
The recover() method in the go language returns an interface{} type, which is converted from the interface type to the Error type by e.(Error), which is also called type inference in golang.
Type inference may cause the process to panic, so it is not very safe to use. Therefore, it is officially recommended to use the [ok-idiom](https://golang.org/doc/effective_go.html) mode.
That is, the code is changed to the following:

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