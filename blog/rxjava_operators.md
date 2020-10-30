# RxJava 常用操作符

所有的操作都是操作符，除了 `Subscribe` 每个操作符都会生成新的 `Observable`  
每个操作符的左边就是上游（upstream），右边就是下游（downstream），对于像 `Map` 这样的操作符就是对上游发射的数据进行变换，所以会继承 `AbstractObservableWithUpstream` 以持有上游的 `Observable`，并封装自己的 `Observer` 以对数据进行操作后交给下游 `Observer`  

## Map

Map 对上游的数据项进行简单的变换（映射），返回新的 `ObservableMap`。实现就是把下游的 `Observer` 包装成 `MapObserver` 订阅给上游  

```java
Observable
    .range(1, 5)
    .map(v -> v * 10)
    .subscribe(System.out::println);
```

```java
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    ...
    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }
    ...
}
```

## FlatMap

见名思意，就是把上游的数据项 Map 成 `Observable`，然后把这些 `Observable` flatten 拍平，这个拍平指的就是不保证顺序的 merge  

```java
Observable
    .range(1, 5)
    .flatMap(integer -> Observable.range(1, 5))
    .subscribe(System.out::println);
```

```java
public final class ObservableFlatMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    ...
    @Override
    public void subscribeActual(Observer<? super U> t) {
        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
            return;
        }
        source.subscribe(new MergeObserver<>(t, mapper, delayErrors, maxConcurrency, bufferSize));
    }
    ...
    static final class MergeObserver<T, U> extends AtomicInteger implements Disposable, Observer<T> {
        @Override
        public void onNext(T t) {
            ...
            ObservableSource<? extends U> p;
            try {
                p = Objects.requireNonNull(mapper.apply(t), "The mapper returned a null ObservableSource");
            } catch (Throwable e) {
                ...
            }
            ...
            subscribeInner(p);
        }
        void subscribeInner(ObservableSource<? extends U> p) {
            for (;;) {
                if (p instanceof Supplier) {
                    ...
                } else {
                    InnerObserver<T, U> inner = new InnerObserver<>(this, uniqueId++);
                    if (addInner(inner)) {
                        p.subscribe(inner);
                    }
                    break;
                }
            }
        }
    }
}
```

简单点说就是把上游的数据项都变换成 `Observable`，把这些 `Observable` 都订阅给下游，但是对于只发射一个值的 `Observable` 做一下特殊处理，对于并发操作也要进行处理，所以要复杂一点  

> 如果想要保证按顺序 merge 可以使用 `concatMap` 操作符

----

## SubscribeOn 和 ObserveOn

RxJava 不会直接使用线程或线程池，而是使用 `Scheduler` 调度器和 `SubscribeOn`，`ObserveOn` 这两个操作符来完成异步操作  
常用的线程调度器有：适合执行计算密集型任务的 `Schedulers.computation()`，适合执行 I/O 密集型任务的 `Schedulers.io()`，适合串行任务的 `Schedulers.single()`  
如果不指定调度器的话，那么在哪个线程订阅（`subscribe()`）的，就在哪个线程上执行操作符的逻辑，就在哪个线程通知观察者，如：  

```java
Observable
    .<String>create(emitter -> {
        for (int i = 0; i < 3; i++) {
            String numStr = String.valueOf(i);
            System.out.println(String.format(Locale.US, "emitting \"%s\"   in Thread: [%s]", numStr, Thread.currentThread().getName()));
            emitter.onNext(numStr);
        }
        emitter.onComplete();
    })
    .map(s -> {
        Integer number = Integer.decode(s);
        System.out.println(String.format(Locale.US, "map (\"%s\" -> %d) in Thread: [%s]", s, number, Thread.currentThread().getName()));
        return number;
    })
    .filter(v -> {
        System.out.println(String.format(Locale.US, "filter %d       in Thread: [%s]", v, Thread.currentThread().getName()));
        return v % 2 == 0;
    })
    .subscribe(r -> {
        System.out.println(String.format(Locale.US, "observe %d      in Thread: [%s]", r, Thread.currentThread().getName()));
    });
```

```text
emitting "0"   in Thread: [main]
map ("0" -> 0) in Thread: [main]
filter 0       in Thread: [main]
observe 0      in Thread: [main]
emitting "1"   in Thread: [main]
map ("1" -> 1) in Thread: [main]
filter 1       in Thread: [main]
emitting "2"   in Thread: [main]
map ("2" -> 2) in Thread: [main]
filter 2       in Thread: [main]
observe 2      in Thread: [main]
```

调用 `subscribe()` 订阅，就是调用最后一个操作符也就是 `ObservableFilter` 的 `subscribeActual()` 方法，而该方法就是调用上游 `ObservableMap` 的 `subscribeActual()` 方法，而它也是调用上游 `ObservableCreate` 的 `subscribeActual()` 方法，而它就是调用 `create()` 中的逻辑，`emitter.onNext()` 其实就是调用 `observer.onNext(t)`  
而我们定义的观察者在 `ObservableFilter` 的 `subscribeActual()` 中被封装成 `FilterObserver`，在 `ObservableMap` 中又被封装成 `MapObserver`，所以在 `ObservableCreate` 中操作的 observer 就是 `MapObserver`  
而 `MapObserver` 的 `onNext()` 就是把值 map 后交给下游 `FilterObserver` 的 `onNext()`，它在 filter 后交给下游我们定义的观察者的 `onNext()`  

> Observable [Create, Map, Filter, Subscribe] Observer  
> 在主线程中执行 `ObservableFilter` subscribe 我们的观察者  
在主线程中执行 `ObservableFilter#subscribeActual()`  
在主线程中执行上游 `ObservableMap` subscribe `FilterObserver`  
在主线程中执行 `ObservableMap#subscribeActual()`  
在主线程中执行上游 `ObservableCreate` subscribe `MapObserver`  
在主线程中执行 `ObservableCreate#subscribeActual()`  
在主线程中执行 `emitter.onNext()`  
在主线程中执行 `MapObserver#onNext()`  
在主线程中执行 `FilterObserver#onNext()`  
在主线程中执行我们定义的 `onNext()`  

整个调用序列都是在 main 线程一个线程上调用执行的  
`observeOn()` 可以指定一个调度器，返回一个新的 Observable（`ObservableObserveOn`），让下游的 `onNext()` 在给定的调度器（线程）上执行，如  

```java
Observable
    .create()
    .observeOn(Schedulers.computation())
    .map()
    .filter()
    .subscribe();
```

```text
emitting "0"   in Thread: [main]
emitting "1"   in Thread: [main]
emitting "2"   in Thread: [main]
map ("0" -> 0) in Thread: [RxComputationThreadPool-1]
filter 0       in Thread: [RxComputationThreadPool-1]
observe 0      in Thread: [RxComputationThreadPool-1]
map ("1" -> 1) in Thread: [RxComputationThreadPool-1]
filter 1       in Thread: [RxComputationThreadPool-1]
map ("2" -> 2) in Thread: [RxComputationThreadPool-1]
filter 2       in Thread: [RxComputationThreadPool-1]
observe 2      in Thread: [RxComputationThreadPool-1]
```

也就是说 `observeOn(Schedulers.computation())` 让下游 `MapObserver#onNext()` 在计算线程上执行了。当然 `observeOn()` 可以多次调用来改变下游 `onNext()` 在哪个线程上执行，如  

```java
Observable
    .create()
    .observeOn(Schedulers.computation())
    .map()
    .observeOn(AndroidSchedulers.mainThread())
    .filter()
    .subscribe();
```

```text
emitting "0"   in Thread: [main]
emitting "1"   in Thread: [main]
emitting "2"   in Thread: [main]
map ("0" -> 0) in Thread: [RxComputationThreadPool-1]
map ("1" -> 1) in Thread: [RxComputationThreadPool-1]
map ("2" -> 2) in Thread: [RxComputationThreadPool-1]
filter 0       in Thread: [main]
observe 0      in Thread: [main]
filter 1       in Thread: [main]
filter 2       in Thread: [main]
observe 2      in Thread: [main]
```

可见在计算线程中执行 `MapObserver#onNext()` 时，调用下游的 `ObservableObserveOn#onNext()`，而它就是在主线程上执行下游的 `FilterObserver#onNext()`  

简单点说，`observeOn()` 决定了下游在哪个线程上执行  

但是怎么控制最上游，也就是发射数据的 `ObservableCreate` 在哪个线程上执行呢？发射数据肯定是在 `subscribe()` 订阅的时候就发射的啊，所以在哪个线程中订阅就在哪个线程上发射，所以改变订阅（`subscribe()` 被调用）的线程就能改变发射数据所在的线程  
`subscribeOn()` 可以指定一个调度器，返回一个新的 Observable（`ObservableSubscribeOn`），让上游和下游在给定的调度器（线程）上发生订阅，如  

```java
Observable
    .create()
    .subscribeOn(Schedulers.io())
    .observeOn(Schedulers.computation())
    .map()
    .observeOn(AndroidSchedulers.mainThread())
    .filter()
    .subscribe();
```

```text
emitting "0"   in Thread: [RxCachedThreadScheduler-1]
emitting "1"   in Thread: [RxCachedThreadScheduler-1]
emitting "2"   in Thread: [RxCachedThreadScheduler-1]
map ("0" -> 0) in Thread: [RxComputationThreadPool-1]
map ("1" -> 1) in Thread: [RxComputationThreadPool-1]
map ("2" -> 2) in Thread: [RxComputationThreadPool-1]
filter 0       in Thread: [main]
observe 0      in Thread: [main]
filter 1       in Thread: [main]
filter 2       in Thread: [main]
observe 2      in Thread: [main]
```

可见 `subscribeOn(Schedulers.io())` 让上游 `ObservableCreate` 和下游在 I/O 线程上发生的订阅，发射数据也就在这个线程上  

简单点说，`subscribeOn()` 决定了最开始在哪个线程发射数据  

> 不要多次使用 `subscribeOn()`，因为这没有任何意义，还会造成逻辑混乱。如果多次调用了，那么后来的 `subscribeOn()` 不会影响发射数据所在的线程，也不会影响下游操作符被执行的线程，只会影响中间订阅被执行的线程（而这对用户来说不会产生任何影响）。所以不要多次调用 `subscribeOn()`  

```java
// 错误的用例
Observable
    .create()
    .subscribeOn(Schedulers.io())
    .map()
    .subscribeOn(Schedulers.computation())
    .filter()
    .subscribe();
```

```text
emitting "0"   in Thread: [RxCachedThreadScheduler-1]
map ("0" -> 0) in Thread: [RxCachedThreadScheduler-1]
filter 0       in Thread: [RxCachedThreadScheduler-1]
observe 0      in Thread: [RxCachedThreadScheduler-1]
emitting "1"   in Thread: [RxCachedThreadScheduler-1]
map ("1" -> 1) in Thread: [RxCachedThreadScheduler-1]
filter 1       in Thread: [RxCachedThreadScheduler-1]
emitting "2"   in Thread: [RxCachedThreadScheduler-1]
map ("2" -> 2) in Thread: [RxCachedThreadScheduler-1]
filter 2       in Thread: [RxCachedThreadScheduler-1]
observe 2      in Thread: [RxCachedThreadScheduler-1]
```

说明只有第一次 `subscribeOn()` 决定了发射数据所在的线程  

> Observable [Create, SubscribeOn, Map, ~~SubscribeOn~~, Filter, Subscribe] Observer  
> 在主线程中执行 `ObservableFilter` subscribe 我们定义的观察者  
在主线程中执行 `ObservableFilter#subscribeActual()`  
在主线程中执行上游 ~~`ObservableSubscribeOn`~~ subscribe `FilterObserver`  
在主线程中执行 ~~`ObservableSubscribeOn#subscribeActual()`~~  
**在计算线程中执行上游 `ObservableMap` subscribe ~~`SubscribeOnObserver`~~**  
在计算线程中执行 `ObservableMap#subscribeActual()`  
在计算线程中执行上游 `ObservableSubscribeOn` subscribe `MapObserver`  
在计算线程中执行 `ObservableSubscribeOn#subscribeActual()`  
**在 I/O 线程中执行上游 `ObservableCreate` subscribe `SubscribeOnObserver`**  
在 I/O 线程中执行 `ObservableCreate#subscribeActual()`  
在 I/O 线程中执行 `emitter.onNext()`  
在 I/O 线程中执行 `SubscribeOnObserver#onNext()`  
在 I/O 线程中执行 `MapObserver#onNext()`  
在 I/O 线程中执行 ~~`SubscribeOnObserver#onNext()`~~  
在 I/O 线程中执行 `FilterObserver#onNext()`  
在 I/O 线程中执行我们定义的 `onNext()`  

----

## Merge

`merge()` 可以合并多个 `Observable`，其实就是把多个 `Observable` 变成集合后调用 `flatMap()`。所以 merge 是不保证顺序的  

> 如果想要按顺序合并多个 `Observable`，可以使用 `concat()`  
如果不想因为其中一个 `Observable` 的错误导致流中断，可以使用 `mergeDelayError()` 等到所有数据项都发射完再发射和处理错误  
一个 `Observable` 实例可以使用 `mergeWith()` 来 merge 另一个实例  

## Zip

`zip()` 可以按顺序压缩所有 `Observable` 的第 i 个数据项。最少的那个 `Observable` 发射完了就算完成了，其他 `Observable` 可能会被马上 dispose 并且接收不到 `complete` 的回调（`doOnComplete()`）  

> 一个 `Observable` 实例可以使用 `zipWith()` 来 zip 其他的实例  

----

## StartWith

在发射数据项前发射一个（`startWithItem()`）或多个（`startWithArray`, `startWithIterable()`）数据项，当然，`startWith()` 一个 `Observable` 也可以  

## Concat

`concat()` 可以将多个 `Observable` 按顺序拼接起来，一个 `Observable` 发射完了再发射下一个 `Observable` 的  

> 一个 `Observable` 实例可以使用 `concatWith()` 来 concat 其他的实例

----

## Timer

`timer()` 可以在给定的时间延迟后发射一个 `0L`，默认在计算线程上发射，当然可以指定调度器  

## Interval

`interval()` 可以按照给定的时间间隔发射从 `0L` 开始递增的数字，默认在计算线程上发射，当然可以指定调度器  
可以指定第一次的延迟: `interval(2, 1, TimeUnit.SECONDS)`，2s 后发射第一个数字 `0L`，然后每隔 1s 发射 `1L`, `2L`, ...  
可以指定从哪个数字开始，一共发射几个数字: `intervalRange(5, 10, 2, 1, TimeUnit.SECONDS)`，从 `5L` 开始发射 10 个数字，2s 后发射第一个，然后每隔 1s 发射剩下的  

## Delay

`delay()` 让新的 `Observable` 在发射每个数据前都延迟给定时间后再发射，error 不会延迟发射  
`delaySubscription()` 可以延迟订阅当前 `Observable`  

----

## SkipUntil 和 SkipWhile

`skipUntil()` 让源 `Observable` 放弃发射数据，直到给定的 `Observable` 发射了数据它才可以正常发射数据  
`skipWhile()` 让源 `Observable` 放弃发射数据，直到给定条件变成 `false`  

## TakeUntil 和 TakeWhile

`takeUntil()` 让源 `Observable` 在给定的 `Observable` 发射了数据后马上 complete  
`takeWhile()` 会镜像发射源 `Observable` 的数据，直到给定条件变成 `false` 时马上 complete  

----

## Catch

Catch 操作符可以拦截 `onError` 通知，并且可以把它替换成数据项或数据项序列  
`onErrorReturn()` 在 `Observable` 遇到 error 时不调用 `Observer` 的 `onError()` 方法，而是发射一个给定的数据项，然后 complete  
`onErrorResumeNext()` 在 `Observable` 遇到 error 时不调用 `Observer` 的 `onError()` 方法，而是将控制权交给给定的 `Observable`  
`onErrorComplete()` 在 `Observable` 遇到 error 时不调用 `Observer` 的 `onError()` 方法，错误通知被丢弃并直接 complete  

## Retry

Retry 操作符可以在 `onError` 时重新订阅（也就是重试），重新发射数据  
`retry()` 可以指定最大重试次数，也可以指定重试条件（断言函数）  
`retryUntil()` 可以在给定的函数返回 `true` 时停止重试  
`retryWhen()` 在源 `Observable` 遇到 error 时传递到给定的函数中，如果给定函数返回的 `Observable` 发射了 `onComplete` 或 `onError` 那么继续向下传递这个信号（不是原来的 error 信号），否则重新订阅源 `Observable`  

```java
Observable
    .<String>create(emitter -> {
        System.out.println(String.format(Locale.US, "emitting \"1\"     in Thread: [%s]", Thread.currentThread().getName()));
        emitter.onNext("1");
        emitter.onComplete();
    })
    .flatMap(s -> Observable.error(new RuntimeException("always fails")))
    .retryWhen(attempts -> {
        AtomicInteger counter = new AtomicInteger();
        return attempts.takeWhile(e -> counter.getAndIncrement() != 3)
                .flatMap(e -> {
                    System.out.println(String.format(Locale.US, "retry delay %ds   in Thread: [%s]", counter.get(), Thread.currentThread().getName()));
                    return Observable.timer(counter.get(), TimeUnit.SECONDS);
                });
    })
    .subscribe(s -> {
        System.out.println(String.format(Locale.US, "observe \"%s\"     in Thread: [%s]", s, Thread.currentThread().getName()));
    }, throwable -> {
        System.out.println(String.format(Locale.US, "observe error    in Thread: [%s]", Thread.currentThread().getName()));
    }, () -> {
        System.out.println(String.format(Locale.US, "observe complete in Thread: [%s]", Thread.currentThread().getName()));
    });
```

```text
18:33 emitting "1"     in Thread: [main]
18:33 retry delay 1s   in Thread: [main]
18:34 emitting "1"     in Thread: [RxComputationThreadPool-4]
18:34 retry delay 2s   in Thread: [RxComputationThreadPool-4]
18:36 emitting "1"     in Thread: [RxComputationThreadPool-1]
18:36 retry delay 3s   in Thread: [RxComputationThreadPool-1]
18:39 emitting "1"     in Thread: [RxComputationThreadPool-2]
18:39 observe complete in Thread: [RxComputationThreadPool-2]
```

可见，重试 3 次后最后接收到的是 `onComplete` 信号，因为 `takeWhile()` 发射了 `onComplete`，直接就向下传递给了我们定义的观察者。如果想最终接收 `onError`，那么不应该使用 `takeWhile()`，而是在不满足重试条件时直接返回如 `Observable.error(throwable)` 这样的 `Observable`  

## 更多操作符

如果想自定义操作符或者将几个操作符组合成一个自定义的操作符，可以使用 `lift()` 自定义 `ObservableOperator`，或者使用 `compose()` 给上游 `Observable` 应用给定的 `ObservableTransformer`  

## 更多 Observable

有几个比较特殊的 `Observable`:  

- `Flowable` 支持背压  
- `Single` 发射 1 个成功值或 error  
- `Completable` 发射 complete 或 error  
- `Maybe` 发射 1 个成功值 或 complete 或 error  

当然，这些都提供了函数可以相互转换  
如果想要一个对象既是 `Observable` 又是 `Observer`，如果想要把一个异步操作转换成 `Observable`，如果想要创建一个热 `Observable`，就需要更灵活的 `Subject` 了  

`Subject` 继承了 `Observable`，实现了 `Observer`，它有几个实现类  

- `AsyncSubject` 只有在调用 `onComplete()` 时才会发射最近缓存的 1 个值和 complete，`onError()` 时就不发射最近的值了  
- `BehaviorSubject` 一旦有 `Observer` 订阅了它就开始发射最近的值，`onError()` 时就不发射最近的值了  
- `PublishSubject` 可以在创建之后立即发射数据，它不会缓存任何数据  
- `ReplaySubject` 不管 `Observer` 什么时候订阅都把之前发射过的所有数据再发一遍  
