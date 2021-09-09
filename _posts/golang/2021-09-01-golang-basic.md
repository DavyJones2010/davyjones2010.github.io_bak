---
layout: post
title: golang入门笔记之基础语法
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [golang, golang-lib]
lang: zh
---


# 变量
局部变量单纯地给 a 赋值也是不够的，这个值必须被使用, 否则会编译错误
全局变量是允许声明但不使用
---

# 常量


---
# 交叉编译
目标是linux64
GOOS=linux GOARCH=amd64 go build -o app.linux
目标是win64
GOOS=windows GOARCH=amd64 go build
---

类型的别名&定义
// 将NewInt定义为int类型
type NewInt int
// 将int取一个别名叫IntAlias
type IntAlias = int


-- 声明数组
intArray := []int{1, 2, 3, 4}

数组的长度必须是常量表达式，因为数组的长度需要在编译阶段确定。


-- make的作用:
slice1 := []int{1, 2, 3, 4, 5}
slice2 := []int{3, 3, 3}
slice3 := make([]int, 3)

copy(slice3, slice2)
copy(slice2, slice1)
copy(slice1, slice3)
fmt.Println(slice2)
fmt.Println(slice1)
fmt.Println(slice3)



在引用某个包时，如果只是希望执行包初始化的 init 函数，而不使用包内部的数据时，可以使用匿名引用格式，如下所示：
import _ "fmt"
包初始化的 init 函数



---
如何写ut?


GO是没有方法重载的!
https://www.zhihu.com/question/40661108

---
rune 类型:
// rune is an alias for int32 and is equivalent to int32 in all ways. It is
// used, by convention, to distinguish character values from integer values.
//int32的别名，几乎在所有方面等同于int32
//它用来区分字符值和整数值

type rune = int32


---
关于 Go 语言的类（class）
Go 语言中没有“类”的概念，也不支持“类”的继承等面向对象的概念。
Go 语言的结构体与“类”都是复合结构体，但 Go 语言中结构体的内嵌配合接口比面向对象具有更高的扩展性和灵活性。
Go 语言不仅认为结构体能拥有方法，且每种自定义类型也可以拥有自己的方法。


http://c.biancheng.net/view/83.html

---
defer 关键词
类似java里的finally, 用于资源安全地释放
~~~
defer func() {
  pc.workqueue.ShutDown()
  pc.deletequeue.ShutDown()
}()
~~~

---
- 如何快速赋值?
例如: 
~~~
flavor, _ := cmd.Flags().GetString("flavor")
request.InstanceType = flavor
~~~
怎么样能变成一行? 
~~~
// 如下有编译错误, 怎么消除编译错误? 
request.InstanceType = cmd.Flags().GetString("flavor")
~~~


- 如何快速给结构体中部分field赋值? 
例如: 
~~~
request := ecs.CreateRunInstancesRequest()
request.RegionId = region
request.ZoneId = zone
request.InstanceType = flavor
request.InstanceChargeType = payType
~~~
怎样才能快速变成例如:
~~~
request {
    RegionId = region
    ZoneId = zone
    InstanceType = flavor
    InstanceChargeType = payType
}
~~~




- 

---
# 感悟&吐槽
- 函数可以返回多值, 太爽啦, 可以用匿名变量来屏蔽不需要的字段, 节省内存
- 不显式声明类之间的继承关系, 实现关系, 代码看起来困难很多? 有没有更简便的方式?
- 

