# R&D 备考 2025

## 介绍一下项目？

- 项目的难点：广告组件的滑动请求、业务解耦、开屏广告策略模式，可视化合规工具（appOpsManager.setOnOpNotedCallback）
- 使用到的设计模式：策略模式，模板方法模式

## Kotlin 协程实现原理是什么？

协程的核心是挂起（suspend）和恢复（resume），实现原理就是CPS：把 `suspend` 函数转换成 Continuation 和状态机实现协程的挂起和恢复  

```kotlin
fun testCoroutine(completion: Continuation<Any?>): Any? {
    class TestContinuation(completion: Continuation<Any?>?) : ContinuationImpl(completion) {
        override fun invokeSuspend(_result: Result<Any?>): Any? {
            result = _result
            label = label or Int.Companion.MIN_VALUE
            return testCoroutine(this)
        }
    }
}
```

挂起：执行到挂起函数时由调度器负责停止执行并调度其他协程去执行  
恢复：调度器通过 resumeWith 调度 continuation 继续执行  

### 协程调度器有哪些？

- `Dispatchers.Default` 后台线程池调度，线程数最大为 CPU 核心数，适合 CPU 密集型任务
- `Dispatchers.IO` 复用 `Dispatchers.Default` 线程池以提高资源利用率，线程最多 64，适合 IO 密集型任务
- `Dispatchers.Main` 在 Android 上的实现是通过主线程 `Handler` 的 `post(block)` 实现的，如果已经在主线程了，不想要被 post 延迟调度，可以使用 `Dispatchers.Main.immediate`
- `Dispatchers.Unconfined` 不限制执行的线程，不建议使用

## 内存优化

### 什么情况下会发生内存泄漏？

生命周期长的对象持有生命周期短的对象，导致本该被回收的短生命周期对象没办法回收  

- 静态变量持有 `Activity`
- 单例持有 `Activity`
- 非静态内部类或匿名内部类隐式持有外部类的引用（包括非静态 Handler 延迟消息和 TimerTask）
- 广播、监听或回调在销毁时未取消

### 怎么检测内存泄漏？

- AS 自带的 Profiler 工具手动触发 GC 观察内存变化
- 使用 LeakCanary 自动检测
- 使用弱引用手动检测
- 代码审查（lint）

### 降低内存占用的措施有哪些？

- 用 RGB_565 代替 RGB_8888
- 图片压缩、采样、分片处理（inJustDecodeBounds，inSampleSize）
- 及时 recycle 临时 Bitmap
- 避免 onDraw 中创建对象


### LeakCanary 的实现原理是怎样的？

1. 通过 `ContentProvider` 自动初始化
2. 注册 `ActivityLifecycleCallbacks` 和 `FragmentLifecycleCallbacks` 以便在销毁时检测泄漏
3. 为销毁的 Activity 和 Fragment 创建弱引用并关联引用队列
4. 手动触发 GC
5. 检测泄漏

## App 启动过程

1. Launcher 向系统发送启动请求 Intent
2. AMS 接收到请求后如果应用未运行就请求 Zygote fork 一个应用进程
3. 应用进程启动后会创建 ActivityThread 实例（初始化主线程 Looper 并进入循环）
4. AMS 发送启动入口 Activity 的请求
5. ActivityThread 收到请求后通过 Instrumentation 加载 Activity
6. 依次调用 onCreate, onStart, onResume 方法显示应用页面

## HashMap

**HashMap** 使用 数组 + 链表 + 红黑树实现，允许 key 为 null，非线程安全（put非原子性，可见性，扩容，迭代失效），默认容量 16，负载因子 0.75，两倍扩容  
**HashTable** 使用 数组 + 链表实现，不允许 key 为 null，线程安全，默认容量 11，2n+1 扩容  
**ConcurrentHashMap‌** 使用 CAS + synchronized 保证线程安全，不允许 key 为 null  
**Collections.synchronizedMap** 包装 map 实例实现同步安全  

## 启动模式

singleTask 一般用于 MainActivity 或 SettingActivity，表示任务栈中唯一，singleInstance 表示该 Activity 系统中唯一，会独占一个任务栈  

## Kotlin

### object 关键字的用法有哪些

- 单例（线程安全、访问再初始化）
- 伴生对象（可以实现接口）
- 匿名对象
- 匿名内部类

## 设计模式

### 单例

`object` 关键字，双检锁单例

### 简单工厂，工厂方法，抽象工厂

```kotlin
/**
 * 简单工厂
 */
class SimpleFactory {
    companion object {
        fun createProduct(type: String): Product? {
            return when (type) {
                "A" -> ConcreteProductA()
                "B" -> ConcreteProductB()
                else -> null
            }
        }
    }
}
```

简单工厂：一个工厂生产不同的产品  
工厂方法：不同工厂生产不同产品  
抽象工厂：不同工厂生产一系列不同产品  

实例：`addConverterFactory()`，`BitmapFactory`  

### 建造者模式

`AlertDialog#build()`，`Retrofit#build()`

### 装饰器模式

`FileInputStream`

### 观察者模式

`LifecycleObserver`，`LiveData`  

### 策略模式

`animator.setInterpolator()`，不同图片库加载图片  

### 责任链模式

OkHttp 的 `Interceptor.Chain`  

### 模板方法模式

`AsyncTask`，广告的请求、展示、跳过

## 卡顿优化

- 利用 AS 自带的 Profiler 看主线程的耗时和卡顿
- 利用 `BlockCanary` 监听主线程 `dispatchMessage` 耗时
- 利用 `ArgusAPM` 监听 Choreographer 两次 VSync 耗时

```kotlin
Choreographer.getInstance().postFrameCallback { frameTimeNanos ->
    val frameTimeMs = frameTimeNanos / 1_000_000
    Choreographer.getInstance().postFrameCallback(this)
}

Choreographer.getInstance().postCallback(
    // 触摸输入
    Choreographer.CALLBACK_INPUT‌,
    // 动画
    Choreographer.CALLBACK_ANIMATION‌,
    // 布局绘制
    Choreographer.CALLBACK_TRAVERSAL,
)
```

## 显示相关

### Activity 中获取 View 宽高的方式？

- `ViewTreeObserver` 监听
- `view.post()`
- `onWindowFocusChanged()`

### Activity 显示过程是怎样的？

- Activity attach 创建 PhoneWindow
- onCreate + setContentView 获取或创建 DecorView
- Activity resume： PhoneWindow ViewRootImpl.setView
- onAttachedToWindow, measure, layout, draw
- 将 PhoneWindow 添加到 WMS 上

也就是说第一次 resume 时不能获取宽高，因为 resume 之后才通过 Handler 去通知 measure  

### `getMeasuredWidth()` 和 `getWidth()` 什么情况下不一样？

布局完成前可能不一样，布局完成后一样  
onMeasure() 之后才会确定测量宽度，onLayout() 后才会确定实际宽度  

## Android 事件分发机制

事件产生时，会首先 dispatch:  

* `true`. 只分发到这里，本函数直接消费
* `false`. 不分发也不消费，逐层返回给父 View 消费

`super.dispatchTouchEvent(ev)` 系统默认处理：如果有子 view 就分发给子 view，如果自己是最小颗粒的 view 了，就直接调用 `onTouchEvent()` 消费

`onInterceptTouchEvent()` 时:  

* `true`. ACTION_DOWN 返回 true 时当前 `onTouchEvent()` 消费
* `false`/`super`. 不拦截，继续分发

到达 `onTouchEvent()` 时:  

* `true`. 消费，事件结束
* `false`/`super`. 不消费，逐层返回给父 View 消费

默认点击事件:

```text
Activity: dispatchTouchEvent---ACTION_DOWN
ViewGroup: dispatchTouchEvent---ACTION_DOWN
ViewGroup: onInterceptTouchEvent---ACTION_DOWN
View: dispatchTouchEvent---ACTION_DOWN
View: onTouchEvent---ACTION_DOWN
ViewGroup: onTouchEvent---ACTION_DOWN
Activity: onTouchEvent---ACTION_DOWN
Activity: dispatchTouchEvent---ACTION_MOVE
Activity: onTouchEvent---ACTION_MOVE
Activity: dispatchTouchEvent---ACTION_UP
Activity: onTouchEvent---ACTION_UP
```

**DOWN 事件的分发是为了寻找消费者，找到了，后续的事件直接交给消费者去处理**  

上面的 ACTION_DOWN 只有 Activity 消费了，所以后续的 ACTION_MOVE 和 ACTION_UP 直接交给 Activity 处理了  

如果 ViewGroup 的 dispatch 返回了 true，那么直接消费，不会经过 `onInterceptTouchEvent()` 和 `onTouchEvent()`  

```text
Activity: dispatchTouchEvent---ACTION_DOWN
ViewGroup: dispatchTouchEvent---ACTION_DOWN
Activity: dispatchTouchEvent---ACTION_MOVE
ViewGroup: dispatchTouchEvent---ACTION_MOVE
Activity: dispatchTouchEvent---ACTION_UP
ViewGroup: dispatchTouchEvent---ACTION_UP
```

如果返回了 `false`  

```text
Activity: dispatchTouchEvent---ACTION_DOWN
ViewGroup: dispatchTouchEvent---ACTION_DOWN
Activity: onTouchEvent---ACTION_DOWN
Activity: dispatchTouchEvent---ACTION_MOVE
Activity: onTouchEvent---ACTION_MOVE
Activity: dispatchTouchEvent---ACTION_UP
Activity: onTouchEvent---ACTION_UP
```

分发是自上而下的，消费是自下而上的

如果 A { B { C }} 中，B 的 `onInterceptTouchEvent` 在 ACTION_MOVE 时返回 true，C 的 `onTouchEvent` 返回 true，那么点击 C 的事件传递顺序为：  

```text
A dispatchTouchEvent ACTION_DOWN
A onInterceptTouchEvent ACTION_DOWN
B dispatchTouchEvent ACTION_DOWN
B onInterceptTouchEvent ACTION_DOWN
C dispatchTouchEvent ACTION_DOWN
C onTouchEvent ACTION_DOWN
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onInterceptTouchEvent ACTION_MOVE
C dispatchTouchEvent ACTION_CANCEL
C onTouchEvent ACTION_CANCEL
A dispatchTouchEvent ACTION_MOVE
A onInterceptTouchEvent ACTION_MOVE
B dispatchTouchEvent ACTION_MOVE
B onTouchEvent ACTION_MOVE
A dispatchTouchEvent ACTION_UP
A onInterceptTouchEvent ACTION_UP
B dispatchTouchEvent ACTION_UP
B onTouchEvent ACTION_UP
```

也就是说 `onInterceptTouchEvent` 如果后续事件返回了 true，那么原来的目标子 View 将会收到 ACTION_CANCEL，并且后续事件将不会再走 `onInterceptTouchEvent`，而是直接走 `onTouchEvent`  




















