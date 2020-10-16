# RxJava 常用操作符

所有的操作都是操作符，除了 `Subscribe` 每个操作符都会生成新的 `Observable`  
每个操作符的左边就是上游（upstream），右边就是下游（downstream），对于像 `Map` 这样的操作符就是对上游发射的数据进行变换，所以会继承 `AbstractObservableWithUpstream` 以持有上游的 `Observable`，并封装自己的 `Observer` 以对数据进行操作后交给下游 `Observer`  

## Map

对上游的数据项进行简单的变换（映射），返回新的 `ObservableMap`，实现就是把下游的 `Observer` 包装成 `MapObserver` 订阅给上游  

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
如果想要保证按顺序 merge 可以使用 `concatMap` 操作符

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

也就是说 `observeOn(Schedulers.computation())` 让下游 `MapObserver#onNext()` 在计算线程上执行了，当然 `observeOn()` 可以多次调用来改变下游 `onNext()` 在哪个线程上执行，如  

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

> 不要多次使用 `subscribeOn()`，因为这没有意义，还会造成逻辑混乱，如果多次调用了，那么后来的 `subscribeOn()` 不会影响发射数据所在的线程，也不会影响下游操作符被执行的线程，只会影响中间订阅被执行的线程（而这对用户来说不会产生任何影响）。所以不要多次调用 `subscribeOn()`  

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
