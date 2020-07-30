# Android 相关源码分析

## Glide 4.11.0

```java
Glide.with(activity).load(url).into(imageView);
```

### with

图片加载库必须尊重 `Activity/Fragment/Context` 的生命周期，得在它们活跃的时候加载图片，在它们不活跃的时候暂停加载，在它们销毁的时候清理所占用的内存。也就是说 Glide 必须是 **生命周期敏感** 的，而实现生命周期敏感通常有两种方式，一种是向 Activity 或 Fragment 中插入一个不可见的 Fragment，然后在这个不可见的 Fragment 的声明周期回调中进行相应的处理。还有一种是继承 `LifecycleObserver`，直接作为观察者观察 Activity 或 Fragment 的生命周期。Glide 用的前者，也就是不可见 Fragment 的方式  
> 这种方式并不优雅，它更像是一种 hack，而这样的 Fragment 通常被称为 HeadlessFragment  

如果 with 的 `Context`，那么表示图片加载跟应用生命周期一样，直接返回一个单例 `RequestManager` 即可  
如果 with 的 `Activity`，那么利用它的 FragmentManager 注入一个不可见 Fragment，返回这个 Fragment 持有的 `RequestManager`  
如果 with 的 `Fragment`，那么利用它的 ChildFragmentManager 注入一个不可见 Fragment，返回这个 Fragment 持有的 `RequestManager`  

> 如果 with 在后台线程中调用，那么不管 with 的什么，都当 Context 处理，即返回与应用的生命周期一致的单例 `RequestManager`  

### load

图片加载库应该强大到可以加载各种类型的图片资源，不管是静态的还是动态的，不管是本地的还是远程的，不管是 bitmap、resourceId、file 还是其他自定义形式的  
Glide 把这些图片资源的类型抽象为 model，由对应的 ModelLoader 负责加载  
load 会返回 `RequestBuilder`，它的泛型 `<TranscodeType>` 表示交给目标的编码类型，如 Drawable。每个 `RequestBuilder` 都会持有当前 `RequestManager` 以便进行资源请求  

### into

图片加载库所加载图片的目标一般是 ImageView，但也可能是其他自定义 View，甚至没有明确目标。Glide 把目标都抽象成 `Target`，所以即使是 ImageView 也要被封装成 `ViewTarget`  
into 不但负责设置 Target，还负责构建并开始 Target 的图片请求、清理 Target 的资源并重置请求、继续 Target 重复设置的图片请求  
图片请求（SingleRequest）借助 负责开始加载资源和管理缓存资源的 Engine 工作，而 Engine 执行的 DecodeJob 会通过对应的 DataFetcher 获取资源，对于 url 来说就是 HttpUrlFetcher，在 loadData() 方法中利用 HttpURLConnection 或其他网络库请求远程资源  
最终，加载完的资源会通过 callback（SingleRequest 本身）传递到 Target 的 callback（onResourceReady）中，对于 Target 是 ImageView 的情况，一般调用它的 `setImageDrawable()` 方法就行了  

### Glide 分析

Glide 的优点很多，缺点也很多。为了避免频繁创建和回收 Bitmap 引发的 GC，Glide 弄了个 `BitmapPool` 来复用 Bitmap，这确实一定程度上避免了 GC，但是也因为复用带来了一系列问题，如 RGB565 模式下有些手机上会出现图片加载错乱问题。支持 cross fades 动画，但是动画可能产生图片变形问题。  
Glide 追求解耦，让职责很清晰，`RequestOptions` 负责图片请求配置，`Transformation` 负责图片变换，`ModelLoader` 负责资源获取，但是这也导致了每个类的构造参数和方法参数的膨胀，一个构造器或者方法十几个参数在 Glide 中都很常见  
Glide 缓存不但分为内存缓存和磁盘缓存，内存缓存还分了两级：活跃的（其它 View 正在使用的）和非活跃的（内存中已经有的）。活跃的用弱引用持有，不活跃的用 LRU 算法管理。磁盘缓存同样使用 LRU 算法管理  
Glide 为了保持流式调用的风格，引入 APT 技术，利用注解和注解处理器自动生成一些创建 options 的模板代码，以保证流式调用。我觉得没有必要引入或使用这个技术，尤其是项目要和具体图片库解耦时  

## Retrofit 2.9.0

```java
public interface UserService {
    @GET("/users/{username}")
    Call<User> getASingleUser(@Path("username") String username);
}
```

```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(GITHUB_END_POINT)
        .addConverterFactory(GsonConverterFactory.create())
        .build();
UserService userService = retrofit.create(UserService.class);
Call<User> user = userService.getASingleUser("google");
```

### WebService

客户端描述一个 RESTful API 有很多种方式。最传统的方式就是声明 API 的 url 常量字符串，但是这个 API 所属的 httpMethod 和 所需要的 field 等参数都没法指定，只能通过代码注释说明，在每次使用的时候按照注释单独实现，这种方式既繁琐又不直观又容易出错。还有一种方式是声明一个接口，通过接口的抽象方法来描述 API，抽象方法的参数可以用来描述 query 和 field 等参数，而 httpMethod 可以通过方法注解来描述。Retrofit 选择了后者，也就是接口的方式，这种方式虽然不一定是最好的，但是比传统的方式（前者）要好很多  
此时 Retrofit 就要面临一个问题， **怎么根据一个接口，生成自己的网络请求实现呢**  

```java
public interface UserService {
    @GET("/users/{username}")
    Call<User> getASingleUser(@Path("username") String username);
}
```

比如根据上面的接口生成一个这样的实现类：  

```java
public class OkHttpUserService implements UserService {
    @Override
    Call<User> getASingleUser(String username) {
        String url = buildUrl(annotations, username);
        okhttp3.Call okHttpCall = new OkHttpClient().newCall(new Request.Builder().url(url).get().build());
        return new Call<User>(okHttpCall);
    }
}
```

也就是说生成一个实现类，实现每个 API 方法，每个方法的内部实现 okhttp3.Call 的组装和调用，完成请求结果的解析和封装  
实现这样的代码生成有两种方式，一种是利用 [编译时注解和 APT 技术](https://github.com/shangmingchao/shangmingchao.github.io/blob/master/blog/android_annotation_exercises.md) 在编译时自动生成这样的代码，一种是利用 [运行时注解和动态代理技术](https://github.com/shangmingchao/shangmingchao.github.io/blob/master/blog/design_pattern_proxy.md) 在运行时动态生成代理对象。Retrofit 选择了后者，即在运行时通过反射解析注解并生成代理对象  

> 对于开发者，尤其是 Android 开发者，对性能的要求甚至到了苛刻的地步，对于枚举和反射等技术的使用更是敏感。传统方式虽然也能完成枚举和反射的工作，也会比枚举和反射更快一点，但是复杂度要更高，直观性不好。而枚举反射虽然性能差了一些但是要更加的简单直观，我觉得这是一个权衡问题（Trade off）。很多时候总是要做取舍的，就像有人花费了大量的时间、精力和资源把性能提升了一二十，可能在我看来不值得也没有必要。我用简单的方式实现了一个功能，可能在他人看来不够完美精致。所以我觉得以包容的态度看待技术还是很重要的  

### create

Retrofit 的 `create` 就是用来创建代理对象的，这个代理对象封装并实现了具体的网络请求逻辑。通过 《设计模式：代理模式》一文我们已经知道，动态代理会把代理对象的 `hashCode()`, `equals()`, `toString()`, `getASingleUser()` 方法的调用都转发给 `InvocationHandler` 的 `invoke` 处理，所以只需要在 `invoke` 完成对 API 的解析和请求逻辑的封装即可，而 `hashCode()`, `equals()`, `toString()` 这些方法不需要特别处理：  

```java
if (method.getDeclaringClass() == Object.class) {
  return method.invoke(this, args);
}
```

而对于 `getASingleUser()` 方法，只需要把这个方法封装为 `ServiceMethod` 并调用它的 `invoke` 实现

```java
loadServiceMethod(method).invoke(args);
```

`HttpServiceMethod` 或它的子类 `CallAdapted` 会利用 `RequestFactory` 完成对方法注解的解析以及请求 URL、Header 等的拼装，最终和 `okhttp3.Call.Factory` 一起完成对 okhttp3.Call 的组装  

```java
RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
...
for (Annotation annotation : method.getAnnotations()) {
  parseMethodAnnotation(annotation);
}
...
private void parseMethodAnnotation(Annotation annotation) {
  if (annotation instanceof DELETE) {
    parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
  } else if (annotation instanceof GET) {
    parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
  }
  ...
}
...
RequestBuilder requestBuilder =
    new RequestBuilder(
        httpMethod,
        baseUrl,
        relativeUrl,
        headers,
        contentType,
        hasBody,
        isFormEncoded,
        isMultipart);
...
return requestBuilder.get().tag(Invocation.class, new Invocation(method, argumentList)).build();
```

至此 Retrofit 已经组装好 okhttp3.Call 了，已经可以用了。而 Retrofit 灵活的一点是可以对这个 okhttp3.Call 进行封装，如封装成 `retrofit2.Call`、`io.reactivex.Observable` 等  
Retrofit 利用 `CallAdapter.Factory` 完成对 okhttp3.Call 的封装，利用 `Converter.Factory` 完成对请求响应的解析  

Retrofit 默认有两个 `CallAdapter.Factory`，一个是可以把 okhttp3.Call 封装成 `CompletableFuture`，一个可以把 okhttp3.Call 封装成 `retrofit2.Call`  

```java
List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
    @Nullable Executor callbackExecutor) {
  DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
  return hasJava8Types
      ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
      : singletonList(executorFactory);
}
```

Retrofit 默认有两个转换器，一个可以把响应转成 `ResponseBody`，一个可以把响应转成 `Optional`  

```java
converterFactories.add(new BuiltInConverters());
converterFactories.addAll(this.converterFactories);
converterFactories.addAll(platform.defaultConverterFactories());
...
List<? extends Converter.Factory> defaultConverterFactories() {
  return hasJava8Types ? singletonList(OptionalConverterFactory.INSTANCE) : emptyList();
}
```

### Retrofit 分析

Retrofit 最大个贡献是改变了描述 API 的方式，尤其是描述 RESTful API 的方式，让客户端对 API 的调用更加的简单、直观、安全  
Retrofit 这种 "接口 + 抽象方法 + 注解" 的方式虽然可以实现 API 的描述，但是不能可视化，不能文档化，不能 mock，不能自动化测试。所以我觉得换成 "配置文件 + 插件" 等其它方式可能要更好一点  

## ViewStub 29

只是一个占位，继承 `View`，不绘制（`setWillNotDraw(true)`），且宽高为 0（`setMeasuredDimension(0, 0)`），`inflate()` 就是从父容器中移除自己并 inflate 给定的 view 到自己的本来的位置（index 和 layoutParams），由于移除后就不知道自己的父容器了所以 `inflate()` 只能调用一次。ViewStub 的 `setVisibility()` 方法一般不用，如果用，那么在没 `inflate()` 的情况下会调用 `inflate()`  

## Handler 29

`Looper.prepare();` 可以给一个普通线程关联一个消息队列，`Looper.loop();` 开始循环处理消息队列中的消息，new 一个 `Handler` 可以发送和处理消息，创建 `Handler` 需要指定 `Looper`，如果不指定那么表明是针对当前线程的 `Looper` 的，主线程有个创建好的 `Looper.getMainLooper()` 单例可以直接用  

`Looper.loop();` 是个死循环，循环获取队列中的消息，转发给消息的 target，也就是发送它的 `Handler` 去处理  
而 `Handler` 处理消息的线程就是它关联的 `Looper` 所在的线程，也就是说创建 `Handler` 时传的 `Looper` 在哪个线程调用了 `Looper.loop();`，那么就在哪个线程回调 `handleMessage()`  

```java
for (;;) {
    Message msg = queue.next(); // might block
    ...
    msg.target.dispatchMessage(msg);
}
```

`queue.next()` 最终调用的是名为 `nativePollOnce()` 的 native 方法，而该方法使用的是 `epoll_wait` 系统调用，表示自己在等待 I/O 事件，线程可以让出 CPU，等到 I/O 事件来了才可以进入 CPU 执行  
而每次有新消息来的时候 `enqueueMessage()`，最终都会调用名为 `nativeWake()` 的 native 方法，该方法会产生 I/O 事件唤醒等待的线程  
所以 `nativePollOnce()` / `nativeWake()` 就像对象的 `wait()` / `notify()` 一样，死循环并不会一直占用 CPU，如果没有消息要处理，就让出 CPU 进入休眠，只有被唤醒的时候才会进入 CPU 处理工作  
`IdleHandler` 可以在消息队列中的消息处理完了，进入休眠之前做一些工作，所以可以利用 `Looper.myQueue().addIdleHandler()` 做一些延迟任务，如在主线程中延迟初始化一些大对象会耗时引擎  
`Handler` 的延迟发消息功能如 `sendMessageDelayed`，`postDelayed()` 也是通过延迟唤醒实现的，在消息入队的时候就确定好消息要唤醒的时间，即 `msg.when = SystemClock.uptimeMillis() + delayMillis`，插入自己在队列中应该出现的位置，在取消息时延迟 `nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE)` 时间去取即可  

## 参考

- [Glide](https://github.com/bumptech/glide)
- [Retrofit](https://github.com/square/retrofit)
