# Golang方法集的理解


## 先看一段代码

```go
// notifier 是一个定义了notify行为的接口
type notifier interface {
	notify()
}

// user 使用指针语义实现了notifier接口
type user struct {
	name  string
	email string
}

// receiver为指针语义
func (u *user) notify() {
	fmt.Printf("Sending User Email To %s<%s>\n", u.name, u.email)
}

// receiver为值语义
func (u user) showUserInfo() {
	fmt.Printf("Name:%s\n Email:<%s>\n", u.name, u.email)
}

func main() {

	u := user{"Han", "han@email.com"}

	// 发生编译错误！！！！！！！！！！！
	sendNotification(u)
}

// sendNotification函数接收一个实现notifier接口的变量
func sendNotification(n notifier) {
	n.notify()
}
```

代码在**25行**发生了编译错误

```go
// ./prog.go:32:19: cannot use u (variable of type user) as type notifier in argument to sendNotification:
//   user does not implement notifier (notify method has pointer receiver)
```



## 为什么会发生编译错误？

### 1. 什么是方法集

类型的 *方法集* 确定了该类型的 [操作数](https://bitbili.net/golang_spec.html#操作数) 所可以 [调用](https://bitbili.net/golang_spec.html#调用) 的方法。每一个类型都有一个（可能为空的）方法集与之关联

1. 类型 **T** （值语义）的方法集，包含全部receiver为 **T** 的方法
2. 类型 **\*T** （指针语义）的方法集，包含全部receiver为 **T** + **\*T** 的方法

> 注：当类型调用自己声明的方法时，不需要考虑receiver是**值语义**还是**指针语义**，可以调用全部方法。因为在调用时，编译器自动做了转换。
>
> 例如：
>
> ```go
> // 使用值语义声明变量u
> u := user{}
> // notify方法的receiver为指针语义
> u.notify() 	// 编译器自动转换为 (&u).notify()
> 
> // 使用指针语义声明变量u2
> u2 := &user{}
> // showUserInfo方法的receiver为值语义
> u2.showUserInfo() 	// 编译器自动转换为 (*u2).showUserInfo()
> ```

### 2. 什么是接口

go语言接口是一个或多个方法签名的集合。任何**类型**的方法集中只要拥有该接口 **对应的全部方法签名**（指有相同名称、参数列表 (不包括参数名) 以及返回值列表） ，就表示该**类型**实现了该**接口**，无须在该**类型**上显式声明实现了哪个接口。

## 解决

理解了上面两个概念后，回到文章开头的代码中：

```go
func main() {

	// 使用值语义声明的user
	u := user{"Han", "han@email.com"}
	// 此时 user 类型变量的方法集中仅包含：
	// 值语义receiver的 showUserInfo() 方法

	sendNotification(u)
	// 发生编译错误！！！！！！！！！！！
	// user类型的变量 u 没有实现 notifier 接口
	// 因为拥有指针语义receiver的 notify 方法不属于 u 变量的方法集
}
```

修改后：

```go
func main() {

	// 依然使用值语义声明的user
	u := user{"Han", "han@email.com"}

	// 对u取指针，此时传递给 sendNotification 方法的参数是 *user 类型
	sendNotification(&u)
	// 此时 *user 类型变量的方法集中包含了：
	// 拥有值语义receiver的 showUserInfo() 方法
	// 拥有指针语义receiver的 notify() 方法
	// 实现了 notifier 接口，编译器不报错
}
```

可以得到以下结论：

1. 若以值语义 **T** 作为receiver实现接口，不管是**T类型的值**，还是**T类型的指针**，都实现了该接口
2. 若以指针语义 **\*T** 作为receiver实现接口，只有**T类型的指针**实现了该接口






