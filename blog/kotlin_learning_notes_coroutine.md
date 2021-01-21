# Kotlin 学习笔记协程篇

> Coroutines are computer program components that generalize subroutines for non-preemptive multitasking, by allowing execution to be suspended and resumed
> Donald Knuth, Melvin Conway  1958, 1963

- 核心: 协作，积极性，自觉
- 一个协程 **让（yielding）** 给另一个协程去执行 和 简单调用另一段程序执行 的本质区别是 两个协程的关系不是调用者和被调用者的关系，而是完全对等的关系（symmetric）
- 协程是 **协作式** 多任务，线程是抢占式多任务，这也就意味着协程只提供并发性而不提供并行性
- 协程非常适合硬实时（hard-realtime）的需求，适合分时处理的 IO 密集型操作
- 协程完全由程序自己管理和调度，理论上是串行的，所以不需要锁，但可以借助多线程或多处理器实现并行，协程间通信一般使用管道
- 还有一个叫 Fiber 的概念，Fiber 和协程基本是一样的，不过协程是语言级的概念，不需要操作系统和硬件的支持，而 Fiber 是操作系统级的概念
- 虽然多个线程可以共享进程的内存资源，但并不意味着可以随便无限制地创建大量线程进行异步操作，因为每个线程也需要维护自己栈和计数器等内存资源，不断地创建线程可能会耗尽内存资源，线程的创建和调度需要大量时间，线程越多时间就越多，操作系统的负担就越大，切换线程时需要保存和恢复所有寄存器的状态，即使是使用线程池技术在面对大量的请求和较高的吞吐量要求时也会变得无能为力，此时协程就变得非常有用了，协程不需要操作系统参与，创建和销毁一个协程代价很小，切换协程时只需要保存和恢复 PC、SP 和 DX 三个寄存器的状态就行了，而且协程调度器不关心阻塞的协程，你可以创建成千上万的协程，而且数量再多也不会影响调度时间
- 协程调度器负责将协程放到线程中去执行，所以一个协程阻塞了那么调度器可以马上调度另一个协程来执行，不会造成线程的阻塞  

`launch` 作为一个协程 builder，可以在一个 CoroutineScope 中建造并启动一个协程  
`runBlocking` 也是一个协程 builder，但是它比较特殊，它会中断地阻塞当前线程，直到它完成，所以它一般只用于 `main` 函数和协程测试中

```kotlin
fun main() = runBlocking {
    val job = GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join()
}
```

用 `GlobalScope.launch` 启动的那些协程是顶级的协程，不好控制，它们就像守护线程一样，不会保持进程存活，所以为了控制它们还得时刻记着保存这些协程的引用，还得时刻记得把它们 join 进来，还得考虑有一个挂起了怎么办等等问题，这是一个很麻烦很容易出错的事情

## 结构化并发

有一种解决办法，那就是结构化，有层级关系的协程才好控制，所以每个协程 builder 都会添加一个 `CoroutineScope` 实例到它的代码块中，在这些 scope 中启动的协程就方便控制了，**一个协程会等到它 scope 中启动的所有协程都完成才算完成**

```kotlin
fun main() = runBlocking {
    launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

除了协程 builder 定义的 scope，还可以通过 `coroutineScope` 这个 scope builder 定义自己的 scope

```kotlin
fun main() = runBlocking {
    launch {
        delay(1000L)
        println("World!")
    }
    coroutineScope {
        launch {
            delay(2000L)
            println("Foo")
        }
    }
    println("Hello,")
}
```

结构化并发（Structured concurrency）是控制多任务协作的简单有效的方式之一，是 Kotlin 团队极力倡导的

## 取消协程

`kotlinx.coroutines` 中所有的挂起函数都是可取消的，因此可以调用 `launch` 函数返回 `Job` 的 `cancel()` 或 `cancelAndJoin()` 函数取消一个协程，当协程执行到一个挂起函数时这个挂起函数发现协程已经 cancel 了就会马上抛出一个 `CancellationException` 异常从而终止协程的执行，但是如果协程里没有挂起函数时就没办法自动终止执行了，所以这就来到了协程最基本的问题，那就是 **协作**，每个协程应该积极地检测自己的状态，并且在适当的时候主动 **让** 给其它协程执行，所以可以调用 `yield` 函数，也可以使用 `isActive` 这个扩展属性来主动终止协程里逻辑的执行：

```kotlin
val startTime = System.currentTimeMillis()
val job = launch(Dispatchers.Default) {
    var nextPrintTime = startTime
    var i = 0
    while (i < 5) {
        if (System.currentTimeMillis() >= nextPrintTime) {
            println("job: I'm sleeping ${i++} ...")
            nextPrintTime += 500L
        }
    }
}
delay(1300L)
println("main: I'm tired of waiting!")
job.cancelAndJoin()
println("main: Now I can quit.")
```

```text
job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
job: I'm sleeping 3 ...
job: I'm sleeping 4 ...
main: Now I can quit.
```

说明在这种情况下 `cancelAndJoin()` 并没有真正终止协程的执行，想要在 `cancel` 时终止执行需要把 `while (i < 5)` 换成 `while (isActive)`。或者在 `while` 循环里加一个挂起函数如 `delay(1)`  
当然，可以在挂起函数抛出 `CancellationException` 异常后利用 `try {...} finally {...}` 做一些资源回收工作，而如果 `finally` 里也有挂起函数怎么办，也会抛异常啊，可以使用 `withContext(NonCancellable) {...}` 来保证里面的逻辑是不能被取消的  
`withTimeout` 函数可以用来检测协程的执行的超时，一旦协程执行超时会马上抛出 `TimeoutCancellationException` 异常，它是 `CancellationException` 的子类，所以也是表明协程完成的正常异常  
`withTimeoutOrNull` 不会抛异常，只会返回 `null`  

## 并发协程

协程理论上是串行的，不需要锁，也就是说协程是协作式的，只提供并发性不提供并行性  
并发（Concurrency）指的是多任务的并发，一个任务不需要等另一个任务执行完就可以交给系统去执行，但这两个任务在一个时间点上是不是同时执行的就是另外一个事了  
协程理论上是串行的，所以正常情况下，协程里的代码，后面的肯定是在前面的代码执行完后执行，就像传统的同步代码一样：  

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L)
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L)
    return 29
}
...
val time = measureTimeMillis {
    val one = doSomethingUsefulOne()
    val two = doSomethingUsefulTwo()
    println("The answer is ${one + two}")
}
println("Completed in $time ms")
```

肯定是 `doSomethingUsefulOne()` 函数先执行完，`doSomethingUsefulTwo()` 再执行，如果前者花了 1s，后者也花了 1s，那个 time 肯定就超过 2s 了  
但如果非要两个函数同时执行怎么办？可以分别再开一个协程去执行，但是 `launch` 函数只能返回 `Job` 引用没法直接返回结果，我们就需要另一个可以直接返回结果的协程 builder，那就是 `async`，它会返回一个 `Deferred`，就跟 future 或 promise 意思一样，可以延迟拿结果，`Deferred` 继承的 `Job`，所以也可以随时取消它：

```kotlin
val time = measureTimeMillis {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    println("The answer is ${one.await() + two.await()}")
}
println("Completed in $time ms")
```

由于 `async` 没有指定调度器，这两个协程，或者说这两个函数，还是在同一个线程上执行的  
如果我们想要它们在需要用到函数的执行结果时才执行呢？可以指定 `async` 函数的 `start` 参数为 `CoroutineStart.LAZY`，这样就只会在 `await` 结果或者 `start` Job 的时候才会启动协程了  
这里的 `async` 函数和传统的 `async` 关键字是不一样的，传统的 `async` 关键字用来标记一个函数是异步函数，是可以异步执行的，所以传统的 `async` 关键字跟 Kotlin 的 `suspend` 关键字是一样的，而 Kotlin 的 `async` 函数会启动一个协程去执行  
注意，同步执行也要像这样结构化，不要滥用 `GlobalScope`：

```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}
```

只有这样的程序才是可控的，如果 `doSomethingUsefulTwo()` 出现异常，整个层级包括 `doSomethingUsefulOne()` 也会跟着取消，这才是符合逻辑的，因为一旦一个环节出错，其它环节就没有进行下去的必要了

## 调度协程

协程上下文（CoroutineContext）其实就是一系列协程相关的元素（element），主要的 element 是协程的 Job，还有就是协程的调度器（CoroutineDispatcher），调度器决定协程在哪个或哪些线程上运行，也就是说，调度器可以限制协程在指定的线程中运行，也可以直接把它放到线程池中运行，也可以不对它进行限制  

> `Element` 继承了 `CoroutineContext`，`Job` 和 `CoroutineDispatcher` 都继承了 `Element`，`CoroutineDispatcher` 属于间接继承，它直接继承 `ContinuationInterceptor`，所以它们都是协程上下文

`launch` 和 `async` 都接受可选的 `CoroutineContext` 参数，所以可以利用这个参数指定协程的调度器

```kotlin
launch(Dispatchers.Default) {
    println("Unconfined: I'm working in thread ${Thread.currentThread().name}")
}
```

如果没指定上下文，那么会继承所在的上下文  
如果指定了 `Dispatchers.Unconfined`，表示不作特别的线程限制  
如果指定了 `Dispatchers.Default`，会使用一个共享的后台线程池来运行协程  
如果指定了 `newSingleThreadContext`，会创建一个专用线程来运行协程  

> `launch(Dispatchers.Default) { ... }` 和 `GlobalScope.launch { ... }` 使用了相同的调度器  
专用线程是个非常昂贵的资源，一旦使用，最好记得调用 `close` 函数关闭这个调度器和执行器，或者把它定为全局可重用的  
谨慎使用 `Dispatchers.Unconfined` 这个无限制的调度器，它虽然不限制协程在哪些线程上运行，但是它并不直观，因为它最开始让协程运行在调用它的线程上，但是到第一个挂起点后就不一定了，挂起后继续在哪个线程上执行取决于这个挂起函数，所以它适合那些既不占 CPU 又不更新共享数据的协程，总之这只在一些极端情况下适用，尽量不要在正常代码中使用它  

可以使用 `withContext` 函数让协程在几个线程中来回跳跃（切换所在的线程）：  

```kotlin
runBlocking(ctx1) {
    log("Started in ctx1")
    withContext(ctx2) {
        log("Working in ctx2")
    }
    log("Back to ctx1")
}
```

Job 是上下文的一部分，所以可以上下文中检索出来  

```kotlin
println("My job is ${coroutineContext[Job]}")
```

当父协程被取消后，它所有的子协程都会被递归地取消  
父协程会等待所有的子协程完成  
可以使用 `CoroutineName` element 给协程命名以便进行调试，如果同时指定了名字和调度器等多个 element，可以使用 `+` 操作符把它们结合起来：  

```kotlin
launch(Dispatchers.Default + CoroutineName("test")) {
    println("I'm working in thread ${Thread.currentThread().name}")
}
```

为了结构化并发，协程都是从一个 `CoroutineScope` 中启动的，像 `launch`，`async` 等协程 builder 也都是 `CoroutineScope` 的扩展函数，会继承上下文并自动传播上下文的元素和取消，但是像 Android 的 `Activity` 不是一个 scope 怎么办呢？怎么创建一个 scope 呢？  
创建 scope 实例最简单的方式就是使用 `CoroutineScope` 和 `MainScope` 工厂方法，`CoroutineScope` 会创建通用的 scope，`MainScope` 会创建适合 UI 应用的 scope（使用 `Dispatchers.Main` 作为默认的调度器）  

```kotlin
class MyActivity : AppCompatActivity() {
    private val mainScope = MainScope()
    override fun onDestroy() {
        mainScope.cancel()
    }
    fun showSomeData() = mainScope.launch {
        draw(data)
    }
}
```

还有一种方式就是让一个类（如 `Activity`）直接手动实现 `CoroutineScope` 接口，但是不推荐这么做，可以通过代理来实现，如：

```kotlin
class MyActivity : AppCompatActivity(), CoroutineScope by MainScope() {
    override fun onDestroy() {
        cancel()
    }
    fun showSomeData() = launch {
        draw(data)
    }
}
```

## Flow

挂起函数只能异步地返回单个值，如果我想异步返回多个值咋办，传统集合这个数据结构还能用吗？能用倒是能用，但是得一次性返回：

```kotlin
suspend fun foo(): List<Int> {
    delay(1000)
    return listOf(1, 2, 3)
}
```

这不叫能用好吧，还得看 `Flow<Int>`，它就像同步代码的 `Sequence<Int>` 一样可以异步动态交付多个数据：

```kotlin
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}
...
fun main() = runBlocking<Unit> {
    foo().collect { value -> println(value) }
}
```

Flow 只有在收集的时候才会运行，像个 cold stream  
这么一看，怎么跟 Reactive Streams 这么像啊，确实，它跟 RxJava，project Reactor 等一样，但是它想要更加简单，更加 Kotlin 友好，更加挂起友好，更加尊重结构化并发，同时它也提供了对应的转换器可以和其它的 Stream 相互转换  
Flow 本身没有取消的机制，只能借助里面的挂起函数实现取消  
`flow { ... }` 块中的代码本来就是可挂起的，所以不需要 `suspend`  
`flowOf(1, 2, 3)` 这样可以构造 Flow  
普通集合和 Sequence 都可以通过 `.asFlow()` 转换成 
Flow  
用来收集的 `终止操作符` 除了 `collect()`，还有 `toList`，`toSet` 等收集成集合的，有 `first()` 确定收集一个的，还有 `reduce()` 和 `fold()`  
和 Rx 一样，Flow 在哪个上下文（如线程）收集就在哪个上下文发射数据。但是有一点需要注意，`flow {}` 代码块中不允许切换上下文，如果想指定发射数据的上下文，可以使用 `flowOn` 操作符  
`conflate` 操作符会在上次收集还没完成时直接丢弃本次收集  
`collectLatest` 操作符会在收集到新数据时自动终止之前的收集  
`combine` 操作符和 `zip` 不同的是它不会等到两个 Flow 都发射了数据再组合，而是只要有一个发射了就尝试组合最近的  
`flatMapConcat` 和 `flatMapMerge` 见名思意就是保证顺序的和不保证顺序的 map + flatten  
`flatMapLatest` 和 `collectLatest` 一样会自动终止之前的 Flow  
在收集的时候可以通过 `try/catch` 代码块捕获整个过程中出现的任何异常，但是不要在发射数据的地方加 `try/catch` 代码块，要保证异常的透明，可以使用 `catch` 操作符声明式地捕获异常并进行处理  
`onCompletion` 操作符可以声明式地处理 **上游** 发射完成后的操作（没有取消或失败），它的参数如果不为空，那表示发生了异常，否则就表示成功完成了。要注意的是，如果下游收集发生了异常是会收到异常通知的  
`launchIn` 操作符等价于 `scope.launch { flow.collect() }`，就是启动一个协程去收集，而不用阻塞其后代码的执行  
`flow{}` builder 会在自动在每次发射值时检测 `isActive` 状态，所以它可以及时取消。其它 builder 不会加这个自动检测  
`cancellable` 操作符等价于 `.onEach { currentCoroutineContext().ensureActive() }` 可以自动检测 `isActive` 状态  

## Channel

协程间传递单个数据用 `Deferred` 可以实现，如果传递数据流的话可以用 `Channel`，它类似于阻塞队列，只不过使用可挂起的 `send()` 和 `receive()` 发送接收数据而已，`close()` 可以关闭  
`produce()` 可以启动一个新的生产数据流的协程，数据都会被发送到 `ReceiveChannel` 中  

```kotlin
fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

fun main() = runBlocking {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```

所以可以使用 `produce()` 实现生产者消费者模式或者 Pipeline 管道模式  

## 参考

- [Coroutines Guide](https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html)
