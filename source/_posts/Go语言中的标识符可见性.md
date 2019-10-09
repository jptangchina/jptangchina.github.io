---
title: Go语言中的标识符可见性
author: 鹏徙南冥
top: false
mathjax: false
categories:
  - Golang
tags:
  - Golang
date: 2019-10-09 08:54:54
updated: 2019-10-09 08:54:54
img:
summary:
---

## Go语言中的标识符可见性

Go语言中变量首字母大写，代表其对其他包可见，反之代表不可见。具体到实际使用中，有如下一些体现。

### 1. 同包的可见性

同一个包内代码的标识符始终具有可见性。例如在`main`包下面有两个go文件，分别为`computer.go`与`mian.go`，代码分别如下：

```go
// computer.go
package main

type computer struct {
	brand string
}
```

```go
// main.go
package main

import "fmt"

func main() {
	computer := computer{"华硕"}
	fmt.Println(computer.brand)
}
```

上述代码的目的是打印出电脑的品牌信息，测试执行不会有任何问题，可见同包下的变量是始终可见的。

### 2. 不同包的可见性

现在，将**1**中代码的computer.go转移至computer包下，代码如下：

```go
// computer.go
package computer

type computer struct {
	brand string
}
```

此时即便是`main.go`文件中引入了computer包也无法使用computer结构及其属性，编译器提示`Unexported type 'computer' usage`异常，因为此处computer及其属性都没有大写首字母，因此无法再被访问，修改后的代码应该如下：

```go
// computer.go
package computer

type Computer struct {
	Brand string
}
```

```go
// main.go
package main

import (
	"fmt"
	"tests/computer"
)

func main() {
	computer := computer.Computer{"华硕"}
	fmt.Println(computer.Brand)
}
```

事实上，还可以在`computer.go`中创建工厂函数返回computer对象，并且computer对象及其属性并不需要暴露出去，代码如下：

```go
//computer.go
package computer

type computer struct {
	brand string
}

func New(brand string) computer {
	return computer{brand}
}
```

尽管这样做编译并没有问题，但是在main函数中实际仍不能通过computer.brand获取computer的值，对于属性而言，这里的`brand`仍旧需要对外暴露。同时需要注意的是，此种写法IDE会有警告：`Exported function with unexported type`。

这里其实主要思考的是这样一个问题：为什么没有暴露的对象可以在其他类中被获取到？

对比Java语言，其实效果是一样的，对于包内可见的类与属性，虽然可以通过其他类获取到，但是仍然不能调用类中的方法对类进行操作。

按照`《Go语言实战（中文版）》`一书中给出的解释是，要让这个行为可行，需要满足两个条件：

> 第一，公开或者未公开的标识符，不是一个值
>
> 第二，短变量声明操作符，有能力捕获引用的类型，并创建一个未公开的类型的变量。
>
> 永远不能显示创建一个未公开的变量，不过短变量声明操作符可以这么做。

### 3. 内嵌对象的可见性

内嵌对象如果本身可见，那跟之前的描述差不多，就不再赘述。这里主要描述下内嵌对象本身不可见的情况。例如，在上述`computer`结构中增加一个类型为`screen`的结构：

```go
// computer.go
package computer

type Computer struct {
	screen
	Brand string
}

type screen struct {
	Brand string
	Size int
}
```

这里的screen是没有对外暴露的，也就是说，我们不能像下面这样来初始化一个computer对象：

```go
	computer := computer.Computer {
		Brand: "华硕",
		screen: computer.screen {
			Size: 24,
		},
	}
```

但是利用Go语言的特性，尽管screen本身没有对外暴露，但是其属性却以对外暴露的形式被提升到了computer中，所以可以这样对其赋值：

```go
package main

import (
	"fmt"
	"tests/computer"
)

func main() {
	computer := computer.Computer {
		Brand: "华硕",
	}
	computer.Size = 24
	fmt.Println(computer.Size)
}
```

### 总结

Go语言中通过字母大小写来判定是否可以对外访问。但不可访问不等于不可获取，同样的，结构不可访问不等于属性不可访问，在实际使用中需要仔细甄别。



## 参考

《Go语言实战》