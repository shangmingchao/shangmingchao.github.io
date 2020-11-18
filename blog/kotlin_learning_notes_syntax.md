# Kotlin 学习笔记语法篇

## 常见用法

### 遍历索引和值

```kotlin
for ((index, value) in list.withIndex()) {
    println("$index $value")
}
```

### 返回

`return` 直接返回到最近的 `fun` 处（包括匿名函数）  

```kotlin
fun foo() {
    listOf(1, 2, 3, 4, 5).forEach {
        if (it == 3) return // 直接返回到 foo()
        print(it)
    }
    println("this point is unreachable")
}
```

```text
12
```

可见，对于简单的 lambda 表达式，常规 `return` 并不能返回到 `forEach`  
可以使用带标签的 `return`，如:  

```kotlin
listOf(1, 2, 3, 4, 5).forEach lit@{
    if (it == 3) return@lit // 局部返回到该 lambda 表达式的调用者，即 forEach 循环，return@forEach 也可以
    print(it)
}
```

如果不想取名字可以直接 `return@forEach`  
或者只能用匿名函数替代 lambda 表达式了  

### 对象声明和对象表达式

对象声明的初始化过程是线程安全的并且是在首次访问时进行的  
也就是说，对象声明可以方便地创建一个线程安全的延迟初始化的单例  
一个类最多可以有一个伴生对象 `companion object`  

对象表达式是立即执行的，可以访问局部变量  
如果匿名对象作为公共函数返回值，那么它是其超类型或者 `Any` 类型的  

### 空安全

```kotlin
person?.department?.head = managersPool.getManager()
```

只有 `person` 和 `department` 都不为空时才会执行右侧的表达式和赋值  
如果 `?:` 左侧表达式非空，就返回左侧表达式的值，否则返回右侧表达式的值。只有左侧为空时才会对右侧表达式求值  

### 作用域函数

在一个 **对象上** 执行一个代码块（lambda 表达式）  
这个对象在代码块中如何使用（**上下文**），代码块的结果是什么（**返回值**）  

#### let also [it]

上下文对象作为 lambda 表达式的 **参数** ，就用 `it`  
`it` 可以换成自定义名字  

```kotlin
val length = str?.let {
    println("let() called on $it")
    processNonNullString(it)      // 编译通过：'it' 在 '?.let { }' 中必不为空
    it.length
}
```

#### run with apply [this]

上下文对象作为 lambda 表达式的 **接收者**，就用 `this`  
可以 **省略** `this`  

```kotlin
val adam = Person("Adam").apply {
    age = 20 // 即 this.age = 20
    city = "London"
}
```

#### [上下文对象] apply also  

`apply` 和 `also` 返回上下文对象，所以可以方便的对一个对象进行链式操作  

#### [lambda 表达式的值] let run with

`let`，`run`，`with` 返回表达式的值，所以可以在对对象操作后返回一个想要的结果，或者不需要结果只是想多执行一些操作  
