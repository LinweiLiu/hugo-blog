---
title: "Golang程序结构"
date: 2018-02-28T15:17:07+08:00
categories: ["golang"]
draft: true
---


## 程序示例详解

```go
/* FileName: helloworld.go
 * Author: cijliu
 */
/* 打包，这里main包是一个特殊包，定义了一个独立可执行的程序，而不是一个库或模块 */
package main 
/* 导入包， 这里导入fmt包，这是个Golang官方提供的输入输出包 */
import (
    "fmt"
)
/* 声明变量，Golang声明变量的结构方式：
 *     var 变量名 变量类型 = 变量值
 * 这里把声明统一放在一个地方。
 */
var (
	StringHelloWorld string = "Hello World!"
    i int = 0 //未使用
)
/* main函数作为程序执行的入口函数 */
func main(){
    /* 打印一行，fmt是包名，Println是fmt包内函数 */
    fmt.Println(StringHelloWorld)
}
```

整个Golang程序的结构大致如此，进行编译，这里可以先运行试试看结果，在cmd下执行

> go run helloworld.go

**输出结果**

> Hello World!

若想编译生成二进制可执行文件，在cmd下执行

> go build helloworld.go

同目录下生成对应的helloworld可执行程序。