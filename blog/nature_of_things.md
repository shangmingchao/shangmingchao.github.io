# 格物

## 思考

为什么会存在这种技术？在没有它之前是怎么样的？它解决了哪些问题，又带来了哪些问题？它一定是最好的么？有没有比它更好的？  
有什么技术是学不会的？包括算法，有什么算法是你能学会别人学不会的？如果你看过 ACM 总决赛你就会知道什么才叫真正的差距。2019 年 4 月 4 号在葡萄牙波尔图举办的第 43 届 ACM-ICPC World Finals 中，莫斯科国立大学获得了两连冠，俄罗斯的大学完成了八连冠  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_1.png)  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_2.png)  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_3.png)  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_4.png)  
一个计算机由哪些基本的硬件组成？为什么需要这些硬件？存储元件有哪些？为什么这么多存储元件？为了充分使用和管理这些硬件一个操作系统应该至少具备哪些能力？对于多任务操作系统来说既然资源是有限的那资源怎么分配才算合理？会不会出现资源竞争的情况？出现资源竞争该怎么办？操作系统为什么有进程和线程两个概念？买 CPU 的时候它说的 6 核 和 12 线程都是什么意思？它说的 12MB 三级缓存又什么意思？怎么实现跨平台（跨操作系统）？怎么设计一个跨平台的虚拟机，他应该具备哪些能力？  

## 了解

- 计算机的存储系统受限于目前的技术水平和制作工艺，没办法设计制造体积小容量大读写快成本小的存储元件，所以按照速度从快到慢、容量从小到大、距离 CPU 从近到远的存储元件有: CPU 核心内部的各种寄存器，一般是 TTL 或 CMOS 芯片 → 缓存，就是集成到 CPU 内部的一些 SRAM 芯片 → 内存，就是俗称的内存条，一般是 DRAM 芯片 → 固态硬盘或机械硬盘等即使关机断电也不会丢失数据的大容量存储介质
- CPU，以 [Core i7-8700K CPU](https://ark.intel.com/content/www/us/en/ark/products/126684/intel-core-i7-8700k-processor-12m-cache-up-to-4-70-ghz.html) 为例，拥有 6 个核心，每个核心内部都有一级缓存 L1 和二级缓存 L2，一级缓存 L1 **32KB**，包括一级数据缓存 L1d 和 一级指令缓存 L1i。二级缓存 L2 **256KB**。而三级缓存 L3 **12MB**，是 **所有核心共享的**
- 并不是缓存层数越多缓存越大越好，缓存过大反而会增加缓存查找时间进而影响性能，试想一下，在小房间里找东西肯定比在大房间里找东西要快啊，所以一味地增加缓存容量对性能的提升并不明显
- 虽然 Core i7-8700K CPU 有 6 个核心，但是利用超线程技术每个核心可以同时处理 2 个线程，一共支持 12 个线程，也就是说这个 CPU 在操作系统和普通用户看来 **就像** 有 12 个核心可以同时工作一样。线程本来是软件层面的概念，硬件不应该关心和参与，所以硬件层面支持多线程并行其实是在没有其它更好提升硬件性能的办法时的无奈之举，而且这样一来 CPU 就是把 **性能提升的责任** 部分地推给了操作系统和软件开发者
- 进程（Process）: 表示一个正在执行的程序，或者说交给计算机执行的一个 **任务**。
- 多任务（Multitasking）: 指在 **一段时间内** 并发执行多个任务（进程），新的任务可以在正在执行的任务结束前打断它而不用等它结束。多任务操作系统最常用的实现方式就是分时（Time-sharing），多个进程分时使用 CPU 等资源交替执行，如果分时很短，快速地来回切换上下文，**给人的感觉** **就像** 同一个处理器能同时执行多个任务（运行多个进程）一样，这种 **看起来同时** 执行多个进程被称为 **并发**（Concurrency），也就是说大部分情况下并发指的是多个任务在短时间内交替执行，事实上一个时间点只有一个任务在执行  
- **正是因为多任务的需求才有了进程的概念，进程用来描述具体的一个任务**  
- 协作式多任务（Cooperative multitasking）: 每个进程自愿让出 CPU 时间片等资源给其它进程，如果某些进程工作量太大迟迟不愿放弃 CPU 等资源可能会引起一系列问题，所以只有 RISC OS 等少数操作系统采用这种策略
- 抢占式多任务（Preemptive multitasking）: 操作系统掌握各个进程的完全控制权，可以随时剥夺一个进程对 CPU 等资源的控制，也可以随时调度一个进程马上去执行，所以类 Unix 操作系统和 Windows 操作系统等大部分操作系统都是采用这种抢占式策略
- 为了保证 **安全性** 和 **可靠性** ，操作系统给每个进程分配独立的内存空间，严格限制进程间的通信交流，所以进程间没有办法通过共享内存的方式实现进程间通信。一个进程无法也不应该直接访问其它进程的内存空间，如果试图访问了，那么内存管理单元 MMU 硬件会拒绝执行并告知操作系统采取必要的措施
- 一个进程涉及到东西还是挺多的，包括它要执行的操作（机器码），当前的调度状态，当前的 CPU 寄存器信息，当前执行到了哪一步等等，所以创建维护一个进程代价还是挺大的，切换进程时需要保存和加载寄存器和内存映射，更新各种表和列表等等，所以切换进程的代价非常大
- 创建一个进程最简单最快速的方式就是直接复制一个已存在的进程，所以类 Unix 操作系统有一个 `fork` 系统调用可以这样创建一个子进程，基本所有进程都是通过这个系统调用创建的，子进程也是完整的进程，所以父进程的终止或者崩溃并不一定导致它的子进程终止，子进程变成孤儿进程后操作系统可能会给它重新指定父进程  
- 多线程（Multithreading）: 有些任务需要多个进程协作来完成，但是切换进程的代价太大，交换数据的过程太难，所以需要一些轻量级进程可以共享父进程的内存空间和其它资源，而这种轻量级进程就被抽象成了线程，因此线程的切换不会切换内存的上下文  
- **正是因为高效协作的需求才有了线程的概念**  
- 线程（Thread）: 表示一个最小的程序指令序列，是操作系统调度器调度的最小单元，线程一般属于进程的一部分，一个进程为了能同时进行多个操作可以有多个线程，多个线程共享进程的资源，包括分配的内存空间和非每个线程独有的公共资源
- 单核单线程的 CPU 上实现多个线程同时执行的方式也是通过短时间内交替执行的并发方式实现的，但是对于多核 CPU 或者支持多线程的 CPU 核心就可以实现真正意义上 **同一时刻** **并行**（Simultaneously/Execute in parallel）多个线程了
- 通过提高吞吐量的方式提高计算机性能的方式目前有两个，一个是多线程（multithreading），一个是多处理器（multiprocessing），硬件层面的多线程指的是一个 CPU 核心可以同时执行多个进程或线程，Intel 的超线程技术（Hyper-Threading）就是硬件层多线程的一种实现，这些进程或线程共享 CPU 的计算单元、CPU 缓存、TLB 等资源，试想一下，一个核心在为一个线程做读取变量操作时那它的计算单元肯定在闲着没事干，此时可以让另一个线程使用计算单元，也就是压榨 CPU 核心内部的各种资源，不能让它们闲着。而多处理器一般都是通过一个 CPU 中集成多个核心来实现，2005 年 4 月，英特尔仓促推出简单封装双核的奔腾 D 和奔腾四至尊版 840，从此开启了商用多核处理器的时代，到 2017 年推出的 Core i7-8700K CPU 时 CPU 内部已经集成了 6 个核心  

> 看到过最多的问答是进程和线程的区别是什么，回答是进程是内存分配的最小单元而线程是操作系统调度的最小单元，这没有错，但是这只是进程和线程应用场景的一小部分，重点应该是进程和线程存在的原因和由此带来的问题  
> 安全和可靠是个永恒的话题，每个进程都要有自己独立的内存空间，如果一个进程能直接访问其他进程的内存空间那将会出现无意或者恶意攻击其他进程的情况，所以为了从根本上保证整个系统的安全可靠，硬件和操作系统都不会允许任何跨进程内存访问的请求。同样，最常用的字符串类 String，数据库连接的密码、socket 地址、文件描述等等都是用字符串形式表示的，如果这些都可以被修改，那安全和可靠就无从谈起了，所以 String 类是 final 的，属性是 private final 的

## Java 虚拟机

![arch](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_5.png)  
![jmm](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_6.png)  
![gc](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_7.png)

- 每个操作系统都有自己对应的虚拟机，负责将 Class 文件编译成对应的机器码运行，有了这些平台相关的虚拟机，相同的 Class 文件就可以直接在这些虚拟机上运行了，这个 Class 文件可以是 Java 文件编译生成的也可以是 Kotlin 等其他语言文件编译生成的，只要符合规范（The Java® Language Specification）的 Class 文件就可以跨平台运行，所以 **Java 虚拟机是语言无关的**  
- 基本的回收过程: 标记（Marking） → 清除（Normal Deletion） → 紧凑整理（Compact），每一步的效率都很低，而且内存越大管理的对象越多那么回收的时间代价和性能代价就越大
- 分代回收算法: 实际的应用中的大部分对象的生命周期都很短，所以可以根据这个规律将对象按年龄进行分代。新创建的对象放到 Eden 区，Eden 区满了的话会触发 minor GC 将里面的幸存者对象都移动到 Survivor 0 区，下一次 Eden 区满了的时候会触发 minor GC 将里面的幸存者对象都移动到 Survivor 1 区，Survivor 0 区中幸存的对象会涨岁数并一起移动到Survivor 1 区。如此反复切换，当幸存对象的年龄到了指定阈值时会被移动到年老区，当年老区需要回收时会触发一次major GC，major GC 和 minor GC 一样都会暂停所有线程，不过代价更高。永久代中包含了 JVM 在运行时加载的类和方法等元数据，也可能被 GC 回收
- GC 的性能指标有两个: 吞吐量和延迟。吞吐量是指未 GC 的时间占总时间的比例，延迟是指 GC 暂停对程序响应能力的影响。加大年轻代的比例可以增加吞吐量，但是代价是占用空间、及时性变差、暂停时间变长。每个应用可以根据自身的情况权衡吞吐量和延迟选择适合自己的垃圾回收器以提高性能
- Serial collector: 单线程进行 GC，适用于单处理器上运行的小数据集的应用，`-XX:+UseSerialGC`
- Parallel(Throughput) collector: 多线程 GC，适用于多处理器或多线程硬件上运行的对暂停时间要求不高的中到大型数据集的应用。`-XX:+UseParallelGC`，使用 `-XX:-UseParallelOldGC` 可以禁用并行紧凑整理（Compact），这样主要的收集工作就会在单线程中执行
- Garbage-First (G1) garbage collector: 大概率满足暂停时间要求的同时最大化吞吐量，适用于多处理器大内存的服务器，`-XX:+UseG1GC`
- Concurrent Mark Sweep（CMS） collector: 最小化停顿时间同时分享处理器资源，`-XX:+UseConcMarkSweepGC`
- 类的生命周期：由对应类型的 ClassLoader **加载** 到内存中的方法区，在堆区创建 Class 对象 → 进行完整性校验，为静态成员分配空间并设置默认值，把常量池中的符号引用都解析成直接引用，即 **连接** 过程 → 如果主动直接引用了，就开始类的 **初始化** ：父类 static 成员及 static 代码块初始化，子类 static 成员及 static 代码块初始化，父类普通成员初始化，父类构造器，子类普通成员初始化，子类构造器 → **使用** 分为主动引用和被动引用，主动引用才会初始化类，被动引用（定义类数组，引用类中的常量，引用其父类的静态成员只会初始化父类）不会 → 使用完就可以 **卸载** 了

## Java 线程安全与锁优化

- 方法区和堆是所有线程共享的，也就是说实例域，静态域和数组元素是所有线程共享的，通过读写 `共享内存` 的方式就可以简单方便地实现线程间的协作与通信了，不过这些数据一般都是存储在 **主存** 上的，主存一般就是指内存条，而 CPU 不会直接去读写那么慢的内存条，而是直接在内部的寄存器或缓存上读写数据，而 Java 把这些存储器抽象为 **工作内存**，这就涉及到一个问题，本来共享的是主存，但每个线程都是直接读写的属于自己的工作内存，工作内存和主存之间数据的同步需要一个过程（`read` 将主存变量传输至工作内存，`load` 将 `read` 的变量值写入工作内存中的副本，`store` 将工作内存变量传输至主存，`write` 将 `store` 的变量值写入主存），这个过程可能随时被打断，不是原子操作，共享变量和它的副本们 **同步不及时** 会产生难以预料的问题，即线程安全问题  
![thread](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_8.png)

- 为了提高性能，指令可能并不是按照预期的顺序执行，可能会被 **重排** ，重排对单线程没有影响，但是多线程可能 **也会** 出现线程安全问题  
![rearrangement](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_9.png)

- 原子性（Atomicity）: 像原子一样不能分割。对于原子性操作，要么不做，要么一口气做完
- 可见性（Visibility）: 当一个线程修改了共享变量的值，其他线程能马上知道。指令重排可能会影响可见性。释放 monitor 锁 或者对 volatile 的变量的修改行为会导致变量的值马上写回主存并将它的所有缓存副本标记为无效
- 有序性（Ordering）: 一个操作一定在哪个操作前执行。先天有序性的 `happens-before` 规则: 程序语义上顺序执行; 对于同一个锁的解锁操作肯定先于加锁; 对于 volatile 变量的写操作肯定先于读操作; 线程的 `start()` 方法肯定先于内部的其它操作; A 先于 B，B 先于 C，那么 A 肯定先于 C; 线程内部的所有操作肯定在结束前（`join()` 返回前）执行; 线程的 `interrupt()` 肯定先于中断检测（Thread.interrupted()）; 对象的初始化肯定先于 `finalize()`
- `join()`: 等待加入的线程执行完成后再继续执行后面的操作
- `Thread` 的静态方法 `sleep()` 只会让出 CPU 从而进入休眠（TIMED_WAITING），不会释放已经获得对象锁。而对象的 `wait()` 方法只能在同步方法或同步块中调用表示虽然自己拿到了锁但是不满足某个条件从而释放对象锁并进入等待池（WAITING），等待被 `notift()` 或 `notifyAll` 通知的时候才有资格获取 CPU 时间片（RANNABLE）
- `Condition` 对象的 `await()` / `signal()` 和对象的 `wait()` / `notift()` 一样，线程池正是利用了条件对象的 `await()` 和 `signal()` 完成对线程的保活处理的  
- `yield()` 方法也可以主动让出 CPU，让 **相同优先级的线程** 可以和 **自己** 参与下一次时间片的竞争
- `setDaemon(true)` 可以在启动前将线程标记为守护线程 ，当 JVM 中的线程只有守护线程的时候，JVM 就会退出了，守护线程中的操作并不一定都会执行，所以 `finally` 子句并不一定执行，在里面释放资源并不一定安全
- 偏向锁 → 轻量级锁 → 自旋锁 → 重量级锁  
- **经验** ：大部分情况下，一个锁不会被很多线程竞争，而且可能多次被同一个线程申请。**优化** ：偏向锁，让这个锁偏向一个线程，如果这个线程多次申请这个锁，直接给，不用同步操作的过程  
- **经验** ：绝大部分锁，都不会存在竞争。**优化** ：轻量级锁，直接乐观地借助 CAS(CPU 的 cmpxchg 指令) 进行读写尝试，ABA 问题可以通过时间戳或者版本号来区分和规避  
- **经验** ：一个线程持有锁的时间不长，其它线程稍等一会很可能就能成功申请到锁了。**优化** ：自旋锁，让等待锁的线程先等几个循环再考虑让出 CPU，很可能自旋期间就拿到锁了  
- 如果自旋期间也没能成功拿到锁，就只能升级成重量级锁了，让出 CPU ，阻塞线程  
- JIT 编译时，会自动消除不需要加锁的锁操作，从而对不必要的锁进行消除  
- Java 提供了两个实用的并发控制工具类 CountDownLatch 和 CyclicBarrier  
- CountDownLatch 的 `await()` 方法会等到计数减到 0 时才能向下继续执行（countDown()）  
- CountDownLatch 是一次性的，减到 0 后就不能用了  
- CyclicBarrier 要更强大一点，可以等到所有线程都到达屏障后再继续执行，否则阻塞在 await() 处，重点是之后 CyclicBarrier 可以再设置一个屏障（reset()），继续当计数器使用  
- 信号量 Semaphore 可以控制资源访问，acquire() 申请，release() 释放  
- Exchanger 用于两个线程交换数据，阻塞在 exchange() 处，等待另一个线程准备好数据后交换，之后再继续向下执行  

## JDK 语言机制

Java8 之前创建的匿名内部类中不能访问局部变量，除非把局部变量声明成 `final` 的。因为局部变量只存在方法中，方法执行完了出栈后这个局部变量就没有了，所以匿名内部类会拷贝一份局部变量作为自己的成员变量才能继续使用它，由于变量传递的是引用（地址），所以如果局部变量被更改了，那么会产生数据不一致问题或安全问题，所以 java 直接强制这样的局部变量是不能更改的，即 `final` 的，Java8 之后只是不用显式声明成 `final` 的而已，编译器还是认为这个局部变量是 `final` 的  

`ThreadLocal<T>` 用来管理每个线程独有的变量。`Thread` 会持有 `ThreadLocalMap` 成员，key 是 `ThreadLocal`，value 是 `ThreadLocalMap.Entry`。所以 `ThreadLocal` 对象并不存储值，只是用来表明每个线程会持有各自 T 类型的变量:  

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
...
sThreadLocal.set(new Looper(quitAllowed));
...
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

`Thread` 持有的 `ThreadLocalMap` 成员是强引用，虽然 `ThreadLocalMap` 的 `Entry` 继承了 `WeakReference<ThreadLocal<?>>`，但是这只表明 `ThreadLocal` 对象可以被回收，但是 value 不能被回收，所以如果不及时 `remove()` 掉，还是可能会出现内存泄漏的  

## 协程

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
Processor (P)  OSThread (M)  Goroutines (G)  
![goroutine](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_10.png)  
![steal](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_11.png)

- 因此，开发者不需要知道线程的存在，操作系统不需要知道协程的存在，同步的方式写的异步操作就可以实现高并发  

## HTTPS

通信最重要的就是 **安全**，所以保证通信不被拦截，不被篡改，不被破解是重中之重。加密现在有两种方式，一种是对称加密，一个密钥完成对明文的加密和对密文的解密。还有一种是非对称加密，用公钥对明文进行加密，用对应的私钥才能对密文进行解密。非对称加密比对称加密理论上更安全，但是要更复杂，加解密过程更耗时。  
HTTPS 要解决的问题一个是加密问题，一个是确认通信双方身份的问题。对于加密问题，HTTPS 采用对称加密和非对称加密结合的方式实现。对于确认身份问题，HTTPS 采用公信机构颁发身份证书的方式实现  

1. 客户端向服务端发起请求：东风同志你好，我是白杨，请求通信  
2. 服务端返回证书：是白杨同志啊，这是组织给我发的证书，上边有签名，你鉴定一下吧  
3. 客户端鉴定证书，如果证书有效的话，就生成一个用于通信的密钥，并用证书中的公钥对这个密钥加密后传给服务端：证书我鉴定过了没有问题，新的暗号我已经封好了，你解封一下吧， 以后我们就用这个暗号通信吧  
服务端用与证书公钥配对的私钥对这个加密后的通信密钥进行解密，拿到用于通信的密钥，之后用这个密钥对通信内容进行加密：天王盖地虎

也就是说，客户端和服务端只需要交流 **三** 次就可以安全地确认身份并协商好加密密钥了  

## TCP/IP

物理层：介质。确保有网线或者无线电能够传播 **比特流(bit)** 数据。**网线（光纤, 双绞线, 同轴电缆）**，WIFI，GPRS(4G, 5G, ...)  
数据链路层：可靠传输。**帧(frame)** 同步，差错控制，流量控制，链路管理。**交换机，网卡** ，MAC 地址  
网络层：路由。**包(packet)** ，路由选择，分组交换，阻塞控制。**路由器** ，IP 地址  
传输层：进程通信。建立逻辑连接，传输层寻址，**段(segment)** 传输。TCP，UDP  
会话层：会话。建立、维持、取消会话  
表示层：数据格式化(语法语义)，加密，压缩  
应用层：应用服务。为各种应用程序提供网络服务。HTTP，FTP，SMTP  

TCP 的三次握手：  

1. 客户端请求建立连接：东风同志你好，我是白杨，请求建立连接。SYN = 1, seq = x  
2. 服务端确认并请求连接：我已收到，可以建立连接。SYN = 1, ACK = 1, seq = y, ack = x + 1  
3. 客户端确认连接：我已收到，可以通信了。ACK = 1, seq = x + 1, ack = y + 1  

TCP 的四次挥手：  

1. 客户端请求断开连接：东风同志你好，我是白杨，我要走了，不要联系了。FIN = 1, seq = x  
2. 服务端确认断开请求：可以断开，但是等我先销毁一下资料。ACK = 1, seq = y, ack = x + 1  
3. 服务端请求断开连接：资料已销毁，再见吧我的朋友。FIN = 1, ACK = 1, seq = z, ack = x + 1  
4. 客户端确认断开请求：好的，再见。ACK = 1, seq = x + 1, ack = z + 1  

## 算法

> 科学不会也不能处理奇迹。科学只能处理重复的事件，艺术却不同。艺术是“就是如此”。在一个创作诞生以前，它是 Nothing——它没有来由、毫无征兆；诞生之后，它就是存在，是合理，是自然和美。我们所谈论的算法，作为一门实用的科学，既有科学的一面，也有艺术的一面。作为科学，它的结构可以分析，它的行为可以预测，它的属性可以量化，它的正确性可以证明。作为艺术，在一个算法诞生之后，有时我们只能说“它能工作”，仅此而已；对于它是如何来到这个世界上的，我们一无所知——这里没有“因为……所以……”，也不是简单的从一般到特殊。创造，似乎和生命一般神秘。我们可以给造物穿上漂亮的科学外衣，欣赏它内在的一致性，但是，最让人着迷的创造性的那一部分，却完全无法加以描述。  
—— 白话算法(6) 散列表(Hash Table)从理论到实用（上）  

AND

> 张三丰道：“老道这路太极剑法能得八臂神剑指点几招，荣宠无量。无忌，你有佩剑么？”小昭上前几步，呈上张无忌从绿柳山庄取来的那柄木制假倚天剑。张三丰接在手里，笑道：“是木剑？老道这不是用来両符捏诀、驱邪捉鬼么？”站起身来，右手持剑，左手捏个剑诀，双手成环，缓缓抬起，这起手式一展，跟着三环套月、大魁星、燕子抄水、左拦扫、右拦扫……一招招地演将下来，使到第五十三式“指南针”，双手同时画圆，复成第五十四式“持剑归原”。张无忌不记招式，只细看剑招中“神在剑先、绵绵不绝”之意。  
张三丰一路剑法使完，竟无一人喝彩，各人竟皆诧异：“这等慢吞吞、软绵绵的剑法，如何用来对敌过招？”转念又想：“料来张真人有意放慢了招数，好让他瞧得明白。”  
只听张三丰问道：“孩儿，你看清楚了没有？”张无忌道：“看清楚了。”张三丰道：“都记得了没有？”张无忌道：“已忘记了一小半。”张三丰道：“好，那也难为了你。你自己去想想吧。”张无忌低头默想。过了一会，张三丰问道：“现下怎样了？”张无忌道：“已忘记了一大半。”
周颠失声叫道：“糟糕！越来越忘记得多了。张真人，你这路剑法十分深奥，看一遍怎记得了？请你再使一遍给我们教主瞧瞧吧。”  
张三丰微笑道：“好，我再使一遍。”提剑出招，演将起来。众人只看了数招，心下大奇，原来第二次所使，和第一次使的竟然没一招相同。周颠叫道：“糟糕，糟糕！这可更叫人糊涂啦。”张三丰画剑成圈，问道：“孩儿，怎样啦？”张无忌道：“还有三招没忘记。”张三丰点点头，放剑归座。  
张无忌在殿上缓缓踱了一个圈子，沉思半晌，又缓缓踱了半个圈子，抬起头来，满脸喜色，叫道：“这我可全忘了，忘得干干净净的了。”张三丰道：“不坏，不坏！忘得真快，你这就请八臂神剑指教吧！”说着将手中木剑递了给他。张无忌躬身接过，转身向方东白道：“方前辈请。”周颠抓耳搔头，满心担忧。  
方东白问道：“阁下使木剑吗？”张无忌道：“是，请指教。”方东白猱身进剑，说道：“有僭了！”一剑刺到，青光。处，发出嗤嗤声响，内力之强，实不下于那个秃头阿二。众人凛然而惊，心想他手中所持莫说是砍金断玉的倚天宝剑，便是一根废铜烂铁，在这等内力运使之下也必威不可当，“神剑”两字，果然名不虚传。  
张无忌左手剑诀斜引，木剑横过，画个半圆，平搭上倚天剑的剑脊，劲力传出，倚天剑登时一沉。方东白赞道：“好剑法！”抖腕翻剑，剑尖向他左臂刺到。张无忌回剑圈转，啪的一声，双剑相交，各自飞身而起。方东白手中的倚天宝剑这么一震，不住颤动，发出嗡嗡之声，良久不绝。  
—— 倚天屠龙记第二十四回太极初传柔克刚

### 【生涯】条件

**天赋** → **专业训练** → 思考总结 → 游刃有余

### 【马步】基础

- 边界条件，时刻注意边界条件，包括 **循环终止条件** ，**循环后的状态**  
- 考虑硬件的特征，用 **加法** 或 **移位** 代替乘除以提高效率，注意避免数据 **溢出**  
- 空间换时间，考虑借助 **多个辅助变量** 或 **队列栈哈希表等数据结构** ，尽量减少暴力操作  

![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_12.png)  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_13.png)  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_14.png)  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_15.png)

0000 | 0000  
128 64 32 16 | 8 4 2 1  
Sn = 2^n - 1 = a_n+1 - 1  
2 + 1 = 4 - 1  
4 + 2 + 1 = 8 - 1  

### 【拳脚功夫】二分枚举，暴力打表

#### 正常二分查

```java
int l = 0, r = a.length - 1;
while (l <= r) {
    int m = (l + r) >>> 1;
    if (a[m] == key) {
        return m;
    } else if (a[m] > key) {
        r = m - 1;
    } else {
        l = m + 1;
    }
}
return -1;
```

#### 查找和在数组中的位置

```java
private static int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>(16);
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[] { map.get(complement), i };
        }
        map.put(nums[i], i);
    }
    return null;
}
```

#### 求解数组任意区间的和

```java
static long[] sumMap;

private static void preprocess(int[] nums) {
    sumMap = new long[nums.length];
    sumMap[0] = nums[0];
    for (int i = 1; i < nums.length; i++) {
        sumMap[i] = sumMap[i - 1] + nums[i];
    }
}

private static long sum(int[] nums, int fromIndex, int toIndex) {
    return sumMap[toIndex] - (fromIndex == 0 ? 0 : sumMap[fromIndex - 1]);
}
```

#### 素数

- 2, 3, 5, 7, 11, 13, 17 ...  
- 素数/质数：大于 1 的自然数中，除了 1 和它本身外不再有其它因数的自然数  
- 孪生素数：相差 2 的素数对，(3, 5)，(11, 13)  
- 所有大于 10 的素数中，个位只可能是 1, 3, 7, 9
- 对正整数 n，如果用 2 到 sqrt (n) 之间的所有整数去除都无法整除，则 n 为质数  

```java
public boolean isPrime(int n) {
    if (n <= 3) {
        return n > 1;
    }
    for (int i = 2; i <= Math.sqrt(n); i++) {
        if (n % i == 0) {
            return false;
        }
    }
    return true;
}
```

#### Hash

- Hash 音译为哈希，直译为散列  
- 试想一个生活中的场景：一个朋友 A 问你另一个朋友 Z 的电话号码是多少，你把你的通讯录打开，直接点击 "Z" 就跳转到 Z 的那条电话号码信息了，这比你从电话本的第一条开始找要快得多，这就是哈希索引的好处，你只需要知道 Z 的信息在第 "Z" 个位置就可以快速地获取到 Z 的相关信息了  
- Hash 函数是将任意大小的数据映射到固定大小的数据的函数，函数的返回值通常被称为哈希值、哈希码、摘要、哈希。Hash 本意指的是将什么东西切碎或弄乱，所以 Hash 函数就是把输入的数据扰乱以获得它们的输出  
- Hash 最常见的应用就是应用于 Hash 表，对于给定的关键词可以快速地定位要查找的那条数据记录，因为 Hash 函数的值就是记录的索引。通常 Hash 函数都是不完美的，函数的的值域会小于定义域，所以有可能不同的几个 key 会映射到相同的索引，从而出现冲突/碰撞（collision），也正只是因为这样，Hash 表的每个槽（slot）可能对应几个记录，这个槽也被称为桶（bucket），Hash 值也被称为桶索引  
- Hash 函数大多是用来保护数据的，这就要求 Hash 函数必须是抗碰撞（collision-resistant）的，用于密码学领域的 Hash 函数有三个安全类型: Pre-image resistance 表示给你加密后的 h 你很 **难** 找到原始消息 m 满足 h = hash(m)，Second pre-image resistance 表示给你 m1 你很 **难** 找到 m2 满足 hash(m1) = hash(m2)，Collision resistance 表示你很 **难** 找到 m1 和 m2 满足 hash(m1) = hash(m2)。“ **难**” 有两种含义，一是表示几乎可以肯定系统的安全性超出了任何对手的攻击范围，二是表示从理论上来说计算的复杂度复杂到几乎不可能计算出来。用来保护数据的 Hash 函数分为两种，一种叫 Cryptographic hash functions，就是通过一些复杂的位运算等方式生成 Hash 码，生成的过程虽然通常很复杂但并不是基于纯粹的数学函数的，没有理论支撑，没法证明它具有抗碰撞性，MD5、SHA-1 系列、SHA-2 系列（如 SHA-224，SHA-256，SHA-384，SHA-512）等 Hash 函数都属于这种，还有一种叫 Provably secure hash functions，建立在纯粹的数学问题上，但是目前并不实用，因为设计这样的函数太难了，就算设计出来往往效率特别低，没办法满足性能的要求  
- Hash 函数的应用特别广泛，还可以用来进行缓存的处理；作为 Bloom filter 的一部分；用来快速寻找重复的记录；用来寻找相似的记录；用来寻找相似的子串；用于计算机图形学中的几何哈希；用于身份认证、消息完整性校验、消息指纹、数据损坏检测、和高效数字签名等密码学领域  
- Hash 是不可逆的，因为 Hash 本质上是个信息摘要算法，这些摘要和原本的信息之间没有本质上的相互关系  

#### 字符串哈希  

![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_hashing_strings.png)  

```java
// Java 字符串哈希
int h = 0, len = s.length();
for (int i = 0; i < len; i++) {
    h = 31 * h + s.charAt(i);
}
return h;
```

#### 字符串无序包含  

```java
int[] dic = new int[] {
        2, 3, 5, 7, 11, 13, 17,
        19, 23, 29, 31, 37, 41, 43,
        47, 53, 59, 61, 67, 71,
        73, 79, 83, 89, 97, 101
};
public boolean contains(String a, String b) {
    BigInteger m = BigInteger.ONE;
    for (char c : a.toCharArray()) {
        m = m.multiply(BigInteger.valueOf(dic[c - 'a']));
    }
    for (char c : b.toCharArray()) {
        if (!m.remainder(BigInteger.valueOf(dic[c - 'a'])).equals(BigInteger.ZERO)) {
            return false;
        }
    }
    return true;
}
```

#### 字符串匹配/字符串搜索

![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_string_searching.png)  

RK 算法（Rabin–Karp）使用 rolling hash 迅速过滤掉与 pattern 不匹配的位置，然后检查剩下的位置是否匹配。也就是说，长度为 m 的窗口放到 t 串上，然后每次向右移动一格，每次都比较窗口内的 hash 值是不是等于 pattern 的 hash 值，如果不相等那就继续向右移动，最多 n - m + 1 次比较。而第 2 步 计算窗口 hash 值的时候，只需要根据第一步的 hash 值减去最左边字符，然后剩下的都乘以一个基数，再加上最后一个字符，经过 “减乘加” 就可以算出第二步以及之后的窗口 hash 值了。所以 RK 第一次 hash 的复杂度是 O(m)，之后就是 O(1) 了  
RK 一般取 基数(base) 为 31，模(mod) 为 1e9 + 7  

```java
public static final int base = 31;
public static final int mod = 1000000007;
public int rk(String t, String p) {
    int n = t.length(), m = p.length(), mul = 1;
    int th = 0, ph = 0;
    for (int i = 0; i < m; i++) {
        ph = (ph * base + p.charAt(i) - 'a') % mod;
        th = (th * base + t.charAt(i) - 'a') % mod;
        mul = i == 0 ? 1 : (mul * base) % mod;
    }
    for (int i = 0; i < n - m + 1; i++) {
        if (th == ph) {
            return i;
        }
        if (i == n - m) {
            break;
        }
        th = (th - mul * (t.charAt(i) - 'a')) * base + t.charAt(i + m) - 'a';
    }
    return -1;
}
```

KMP（Knuth–Morris–Pratt）算法只是简化了回退的步骤：

```java
public int[] kmp(String pattern) {
    int[] next = new int[pattern.length()];
    int l = 0;
    for (int i = 1; i < pattern.length(); i++) {
        while (l > 0 && pattern.charAt(i) != pattern.charAt(l)) {
            l = next[l - 1];
        }
        if (pattern.charAt(i) == pattern.charAt(l)) {
            l++;
        }
        next[i] = l;
    }
    return next;
}
public List<Integer> search(String text, String pattern) {
    List<Integer> res = new ArrayList<>();
    int[] maxMatchLens = kmp(pattern);
    int j = 0;
    for (int i = 0; i < text.length(); i++) {
        while (j > 0 && text.charAt(i) != pattern.charAt(j)) {
            j = maxMatchLens[j - 1];
        }
        if (pattern.charAt(j) == text.charAt(i)) {
            j++;
        }
        if (j == pattern.length()) {
            res.add(i - j + 1);
            j = maxMatchLens[j - 1];
        }
    }
    return res;
}
```

### 【内功】动态规划

动态规划（dynamic programming，简称 **DP**）属于运筹学（Operational Research，简称 OR）的一个分支，是求解决策过程 **最优化** 的数学方法  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_16.png)  

> "Those who cannot remember the past are condemned to repeat it"  
—— George Santayana  

如果一个问题可以通过 **分解为子问题** 然后 **递归找到子问题的最优解** 来最优解决，那么说这个问题有 **最优子结构**  
如果一个问题可以 **分解为多次重用的子问题** 或者 **递归中一遍又一遍地解决相同的子问题**，那么就说这个问题有 **重叠子问题**  
一个问题可以用动态规划解决那么这个问题肯定满足两个条件: **最优子结构** 和 **重叠子问题**  
**带记忆的自顶向下法**: 递归求解，递归的时候将子问题的解记在表格中以便求解重叠子问题时直接查表。Top-down approach，递归，备忘录/记忆化  
**自底向上法**: 先求子问题的解，然后利用这些子问题的解来直接解决问题。Bottom-up approach，迭代  
关键词: Bellman equation, Hamilton–Jacobi–Bellman equation, control theory, bioinformatics, economics, 状态转移方程, 无后效性  

#### 线性模型

- 钢条切割，长度为 i 英寸的钢条价格为 P[i]，给定长度为 n 的钢条如何切割使收益 r[n] **最大** ？  
r[n] = max{ P[i], r[n - i] }, i >= 0 && i <= n  
- 在一个夜黑风高的晚上，有n（n <= 50）个小朋友在桥的这边，现在他们需要过桥，但是由于桥很窄，每次只允许不大于两人通过，他们只有一个手电筒，所以每次过桥的两个人需要把手电筒带回来，i 号小朋友过桥的时间为T[i]，两个人过桥的总时间为二者中时间最长的那个。问所有小朋友过桥的总时间 **最短** 是多少？  
f[i] = min { f[i - 1] + T[1] + T[i], f[i - 2] + T[1] + T[i] + T[2] + T[2] }  

#### 区间模型

- 给定一个长度为 n（n <= 1000）的字符串 A，插入 **最少** 多少个字符使得它变成一个回文串？  
f[i][j] = min { f[i + 1][j], f[i][j - 1] } + 1  

#### 背包模型

- 有 N 种物品（每种物品 1 件）和一个容量为 V 的背包。放入第 i 种物品耗费的空间是 C[i]，得到的价值是 W[i]。将哪些物品装入背包可使价值总和 **最大** ？  
f[i][v] = max { f[i - 1][v], f[i - 1][v - C[i]] + W[i] }
- 每种物品 1 件叫 **0/1 背包** ，每种物品无限件叫 **完全背包** ，每种物品 M[i] 件叫 **多重背包**  

![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_17.png)

### 【内功】树

![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_18.png)  
关键词: 二叉树（binary tree），度（degree），层次（level），深度（depth），高度（height），有根树（rooted），满二叉树（full/proper/plane），完全二叉树（complete），完美二叉树（perfect），平衡二叉树（balanced），病态二叉树（degenerate/pathological），线索二叉树（threaded），霍夫曼树（Huffman tree），二叉查找树（search/ordered/sorted/BST），自平衡二叉查找树（self-balancing/height-balanced），AVL 树（Adelson-Velsky and Landis），红黑树（red–black）  
节点的度: 该节点孩子的数量，叶子节点的度为 0  
节点的层次: 1 + 从该节点到根节点的边数  
节点的深度: 从该节点到 **根** 节点的边数  
节点的高度: 从该节点到它的最高的 **叶子** 节点的边数  
树的高度: 根节点的高度  
树的分类:

- 无序树：树中任意节点的子节点之间没有顺序关系
- 有序树：树中任意节点的子节点之间有顺序关系
  - 二叉树: 每个节点最多两个孩子  
    - 完全二叉树: 除了最后一层每一层都填满，最后一层如果不满的话必须从左到右填充  
    - 满二叉树: 每个节点只能有 0 或 2 个孩子  
    - 完美二叉树: 所有内部结点都有 2 个孩子，所有叶子节点都在同一层。即 每一层都填满  
    - 平衡二叉树: 每个节点左右子树的高度差不超过 1。即 越胖越好  
    - 病态二叉树: 每个父节点有且仅有 1 个子节点。已经退化成线性结构了  
    - 霍夫曼树: 带权路径长度最短  
    - 二叉查找树: 每个节点存的都是 key（原则上带着 value 也可以），每个节点的 key 必须大于等于左子树的 key，小于等于右子树的 key，叶子节点没有 key
      - AVL 树: 经典的自平衡二叉查找树
      - 红黑树: （弱）自平衡二叉查找树

二叉树的属性:  

- n 个节点那么肯定有 n - 1 条边（除了根节点，每个节点都有父亲）  
- 二叉树的最小高度: logn  
- 平衡满二叉树: h = logl + 1 = log(n+1)  
- 完全二叉树内部结点个数: n/2  

二叉树的存储方式:  

- 节点和引用: 每个节点存储自己的数据和两个孩子的引用，由于很多节点的孩子都不到两个，导致空间的浪费，解决办法之一是将本应该为空的引用利用起来，将二叉树线索化，即 **线索二叉树**  
- 数组: 广度优先的隐式数据结构，如果是完全二叉树那么每个节点都会紧紧地挨在一起，没有任何的空间浪费。第 i 个节点的两个孩子肯定在 2*i + 1 和 2*i + 2 的位置，唯一父亲的位置肯定在 (i - 1) / 2 的位置  

> 霍夫曼编码: 霍夫曼在 MIT 读博的时候，为了完成 **找到最有效的二进制编码** 的  term paper，最终发现了 **基于有序频率构建二叉树** 进行编码的编码方式，并证明了它是最有效的编码方式  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_19.png)  
![image.png](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/nature_of_things_20.png)  
二叉查找树: 中序遍历的结果就是一个升序的数组，所以可以使用二分查，但是最坏的情况是构造的树是个病态的绝对左倾或绝对右倾树，直接退化成线性结构了，所以时间复杂度平均 O(log n)，最坏 O(n)，在 n 很大时 O(log n) 和 O(n) 的差别会特别大，所以为了避免二叉树退化，可以在插入节点时根据情况旋转一下来达到自平衡  
自平衡查找树的实现还有 B 树，伸展树，2-3 树等的实现
