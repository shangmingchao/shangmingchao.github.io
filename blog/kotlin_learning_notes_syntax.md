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

### 类

大部分类都不是基类，所以都不能被继承，如果想要作为基类被继承，需要使用 `open` 关键字修饰  
大部分成员函数都不需要被覆写，如果想要被覆写，需要使用 `open` 关键字修饰  
非空类型的属性必须直接初始化，或者在构造器中初始化，或者使用 `lateinit` 修饰（`isInitialized` 检测是否初始化完成），或者使用 `by` 进行委托  
`by lazy` 默认是线程安全的（`LazyThreadSafetyMode.SYNCHRONIZED`），如果不需要线程同步可以使用 `LazyThreadSafetyMode.PUBLICATION`，如果初始化和使用肯定在相同的线程所以为了避免同步操作可以使用 `LazyThreadSafetyMode.NONE`  

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

### 集合

Kotlin 把集合分为只读的（read-only）和可变的（mutable），只读的集合是不能进行增删改操作的  
集合默认是只读的  
如果想使用可变集合，有对应 `Mutable` 前缀的，如 `mutableListOf(1, 2, 3)`  
大部分列表都是读操作更频繁一点，所以 List 的默认实现是 `ArrayList`  
大部分数学集合有个读写顺序更方便一点，所以 Set 的默认实现是 `LinkedHashSet`  
大部分字典有个读写顺序更方便一点，所以 Map 的默认实现是 `LinkedHashMap`  
创建集合最简单最推荐的方式是:  

```kotlin
val numbersSet = setOf("one", "two", "three", "four")
val emptySet = mutableSetOf<String>()
```

只读空集合可以使用 `emptyList<String>()` 这样的函数  
使用 Map 的 `"one" to 1` 构造 Map 时如果想提升性能（避免创建临时 Pair），可以这样 `mutableMapOf<String, String>().apply { this["one"] = "1"; this["two"] = "2" }`  
每个集合操作符都会产生新的集合，不会对源集合产生影响  
测试断言 `all()` 对于空集合总是返回 `true`  
`any` 和 `none` 可以用来检测集合是不是空的  
`groupBy()` 可以返回一个 Map，key 是 lambda 表达式的值，value 是对应的列表  
取值时为了避免越界检查可以使用 `getOrNull(5)` 或者 `numbers.getOrElse(5, {0})`  
只取前 2 个 `list.take(2)`，如果小于 2 个会返回整个集合，所以很安全  

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

#### 作用域函数的实际使用

对一个可空对象执行操作，可以使用 `let`:  

```kotlin
val length = str?.let { 
    println("let() called on $it")        
    processNonNullString(it)
    it.length
}
```

对一个对象进行配置，可以使用 `apply`:  

```kotlin
val adam = Person("Adam").apply {
    age = 32
    city = "London"        
}
```

对一个对象访问的同时执行一些操作，可以使用 `also`:  

```kotlin
numbers
    .also { println("The list elements before adding new one: $it") }
    .add("four")
```

用这个上下文对象做一些操作，可以使用 `with`:  

```kotlin
with(numbers) {
    println("'with' is called with argument $this")
    println("It contains $size elements")
}
```
