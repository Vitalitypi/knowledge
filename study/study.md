# 学习-5-30

## go语言

### Filecoin介绍

git clone https://github.com/filecoin-project/lotus.git

Filecoin是一个基于区块链机制的分布式存储网络。系统中的矿工可以为网络提供存储空间，其通过周期性地提供加密证明（证明他们正在提供一定的空间）来挣到加密货币（FIL）。此外，Filecoin也可以使不同党派通过记录在Filecoin区块链的共享分类账簿上的交易来交换FIL货币。这个系统不使用中本聪方式的工作量证明来维护区块链的共识，而是使用它特有的存储量证明：矿工的共识权利与它提供的存储数量成正比。

Filecoin不仅维护了FIL交易以及账户的分类账簿，也实现了FilecoinVM——一个执行加密合约以及网络成员之间交易机制变量的复制状态机。这些合约包括存储交易，即客户端向矿工支付FIL货币来交换一定的空间用于存储客户请求的特定文件数据。通过FilecoinVM的分布式实现，记录在区块链上的存储交易以及其他的合约机制将会随着时间继续执行，不需要来自原始部分（例如请求存储数据的客户）的更深一步的相互交流

### 相关源码学习

#### option模块

优点：

1. 支持传递多个参数，在参数个数、类型发生变化时保持兼容
2. 任意顺序传递参数
3. 支持默认值
4. 方便拓展

缺点：

1. 新增字段就需要添加一个方法

##### 例1

option/api/option.go

```go
package api
import "fmt"
type Api struct{
	id int
	ak string
	bk string
	ck int
}
type Option func(api *Api)
func SetBk(bk string)Option{
    return func(api *Api){
        api.bk=bk
    }
}
func SetCk(ck int)Option{
    return func(api *Api){
        api.ck=ck
    }
}
func NewApi(id int, ak string, options ...Option) *Api {
	api:=&Api{id: id,ak: ak}
	for _,option:=range options{
		fmt.Println(option)
		fmt.Printf("%T\n",option)
		option(api)
	}
	return api
}

func (api *Api) GetName() {
	fmt.Println("GetName:",api)
}

func (api *Api) GetInfo() {
	fmt.Println("GetInfo:",api)
}
```

option/main.go

```go
package main

import "Knowledge/option/api"

func main() {
	api.NewApi(1,"ak",api.SetCk(123),api.SetBk("bk")).GetName()
	api.NewApi(2,"ak2").GetInfo()
}
```

#### context模块

##### context结构

context：上下文；语境；环境

```go
//一个context包含截止时间、中止信号以及其他的值
//context的方法可能会被多个goroutine同时调用
type Context interface {
    /*
    当任务完成时（表示这个context应该被取消）Deadline返回时间
    当没有设置deadline时，Deadline返回ok==false
    */
	Deadline() (deadline time.Time, ok bool)
    /*
    Done返回一个管道，当任务做完时（表示context被取消了），这个管道将会关闭
    Done返回一个nil，当这个context从未被取消
    连续调用Done将会获得相同的值
    Done的管道将会异步关闭，在返回取消函数之后
    */
	Done() <-chan struct{}
    /*
    如果Done还未被关闭，err返回nil
    如果Done已经被关闭，err返回non-nil的error
    在err返回一个non-nil的error后，多次调用err将获得相同的error
    */
	Err() error
    /*
    Value返回一个与这个context的key有关的值
    如果没有值与这个key有关，将返回一个nil。
    连续传入相同的key来调用Value将获得相同的结果
    */
	Value(key interface{}) interface{}
}
//background返回一个non-nil，空的Context，它从来不能被取消，没有值，没有deadline。它通常用于main函数、初始化、测试以及对于即将到来的请求的高层Context
func Background() Context {
	return background
}
type CancelFunc func()
//withcancel使用一个新的DoneCancel返回一个parent的副本。当返回的cancel函数被调用或父context的DoneCancel被关闭，返回的context的DoneCancel将会被关闭。
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
// WithTimeout returns WithDeadline(parent, time.Now().Add(timeout)).
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
//withdeadline返回一个Context的副本（继承），这个副本的deadline不晚于d，如果父context的deadline比d早，那么这两个context在语义上是相等的，当到达deadline后或cancel函数被调用后或者父context的done管道被关闭后（无论哪一个发生），返回值context的Done管道关闭
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
//withvalue返回一个父context的副本，这个副本的value与key有关
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
//调用CancelFunc会取消child以及child生成的context，取出父context对这个child的引用，停止相关的计数器
```





## English

### 单词

<img src="C:\Users\77995\AppData\Roaming\Typora\typora-user-images\image-20220530152648490.png" alt="image-20220530152648490" style="zoom: 67%;" />





## 相关参考

[1]: https://spec.filecoin.io/	"Filecoin介绍"
[2]: http://t.csdn.cn/1pPsG	"GoLang option设计模式"
[3]: http://t.csdn.cn/60WEH	"go context用法详解"



# 学习-5-31

## Linux

### 相关命令

1. go语言编译 go build main.go
2. 直接运行go程序 go run main.go
3. 开启超级用户 sudo -s
4. 退出超级用户 exit
5. 更改某文件或目录权限 sudo chmod 777 文件或目录名

### Makefile工程管理

#### 认识make

target：dependency_files

​		command

在一个Makefile文件中，通常包含以下内容

- 需要由make工具创建的目标体（target），通常是目标文件或可执行文件
- 要创建的目标体所依赖的文件（dependency_files）
- 创建每个目标体时所需要执行的命令（command）

##### 例2

1. Makefile文件

   ```makefile
   main:main.o mytool1.o mytool2.o
   	gcc -o main main.o mytool1.o mytool2.o
   main.o:main.c
   	gcc -c main.c
   mytool1.o:mytool1.c mytool1.h
   	gcc -c mytool1.c
   mytool2.o:mytool2.c mytool2.h
   	gcc -c mytool2.c
   ```

2. main.c

   ```c
   #include "mytool1.h"
   #include "mytool2.h"
   int main(int argc,char *argv)
   {
       mytool1_print("hello");
       mytool2_print("hello");
   }
   ```

3. mytool1.h

   ```c
   #ifndef _MYTOOL_1_H
   #define _MYTOOL_1_H
   void mytoo1_print(char *print_str);
   #endif
   ```

4. mytool1.c

   ```c
   #include "mytool1.h"
   void mytool1_print(char *print_str)
   {
       printf("This is mytool1 print:%s\n",print_str);
   }
   ```

5. mytool2.h

   ```c
   #ifndef _MYTOOL_2_H
   #define _MYTOOL_2_H
   void mytoo2_print(char *print_str);
   #endif
   ```

6. mytool2.c

   ```c
   #include "mytool2.h"
   void mytool2_print(char *print_str)
   {
       printf("This is mytool2 print:%s\n",print_str);
   }
   ```

#### Makefile变量

make中的变量无论采用哪种方式定义，使用时格式均为：$(VAR)

##### Makefile常见的自动变量

| 命令格式 |                             含义                             |
| :------: | :----------------------------------------------------------: |
|    $*    |                  不包含扩展名的目标文件名称                  |
|    $+    | 所有的依赖文件，以空格分开，并以出现的先后为序，可能包含重复的依赖文件 |
|    $<    |                     第一个依赖文件的名称                     |
|    $?    |        所有时间戳比目标文件晚的依赖文件，并以空格分开        |
|    $@    |                      目标文件的完整名称                      |
|    $^    |              所有不重复的依赖文件，并以空格分开              |
|    $%    |      如果目标是归档成员，则该变量表示目标的归档成员名称      |

#### Makefile规则

##### 隐式规则

例2的Makefile文件可修改为：

```makefile
OBJS=main.o mytool1.o mytool2.o
CC=gcc
main:$(OBJS)
	$(CC) $^ -o $@
```

所有的.o文件都可以自动由.c文件使用命令$(CC) -c <file.c> -o <file.o>生成。main.o、mytool1.o、mytool2.o就会调用$(CC) -c $< -o $@来生成

##### 模式规则

模式规则规定，在定义目标文件时需要%字符。%表示一个或多个任意字符，与文件名匹配。

### Linux Shell编程

#### shell的语法基础

##### shell脚本文件

1. shell脚本文件结构

   shell脚本文件结构格式是固定的，通常的格式如下：

   ```shell
   #! /bin/bash
   echo "Hello world!"
   ```

   将文件保存为hello.sh。

   符号#!告诉系统它后面的参数是用来执行该文件的解释器。

2. 添加shell脚本文件可执行权限

   chmod +x [文件名]

   例如：chmod +x hello.sh

3. 执行shell脚本文件

   ./hello.sh或sh hello.sh

4. shell脚本文件的注释语句

   以#开头来进行注释，#!说明shell脚本文件的解释器

##### shell的变量及配置文件

1. 用户变量

   - 变量赋值

     一般shell脚本文件的变量都是用户变量。

     a="Hello World!"（赋值号=的两侧不准有空格）

   - 获取变量的值

     使用$来获取变量的值，例如：

     echo $a

   使用例子：

   a="hello world"

   echo $a

2. 环境变量

   有关键字export说明的变量叫做环境变量

   export abc=/mnt/shell

   echo $abc

3. 系统变量

   - $0:当前程序的名称
   - $n:当前程序的第n个参数
   - $*:当前程序的所有参数
   - $#:当前程序的参数个数
   - $$:当前程序的PID
   - $!:执行上一个指令的PID
   - $?:执行上一个指令的返回值

#### shell的流程控制语句

##### 条件语句

```shell
if((条件表达式))
then
	#语句块
fi
```

注意：条件表达式要用双括号括起来

##### 循环语句

```shell
for((循环变量=初值;循环条件表达式;循环变量增量))
do
	#循环体语句
done
```

注意：for循环的条件要用双括号括起来

#### shell编程示例

##### 例3 

编写显示20以内能被3整除的数

```shell
#!/bin/bash
for((i=1;i<20;i++))
do
	if((i%3==0))
	then
		echo $i
	fi
done
```

##### 例4 

编写shell程序实现计算1+2+3+……+100的和，并输出所有能被13整除的数。

```shell
#!/bin/bash
sum=0
for((i=1;i<=100;i++))
do
 if((i%13==0))
 then
  echo $i
 fi
 ((sum=sum+i))
done
echo $sum
```

## go语言

### 逃逸分析

通俗来讲，当一个对象的指针被多个方法或线程引用时，我们称这个指针发生了逃逸。

go中逃逸分析最基本的原则是：如果一个函数返回对一个变量的引用，那么它就发生逃逸

简单来说，编译器会根据变量是否会被外部引用来决定是否逃逸：

1. 如果函数外部没有引用，则优先放到栈中；
2. 如果函数外部存在引用，则必定放到堆中。

##### 例5

```go
package main
type S struct {}
func main() {
  var x S
  _ = identity(x)
}
func identity(x S) S {
  return x
}
```

不存在逃逸，go语言中函数是值传递，在栈中copy一份

##### 例6



## English

### 单词-复习200个

![image-20220531150506159](C:\Users\77995\AppData\Roaming\Typora\typora-user-images\image-20220531150506159.png)

## 相关参考

[1]: http://t.csdn.cn/gFaCV	"Go中的逃逸分析"

