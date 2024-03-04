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

客户端描述一个 RESTful API 有很多种方式。最传统的方式就是声明 API 的 url 常量字符串，但是这个 API 所属的 httpMethod 和 所需要的 field 等参数都没法指定，只能通过代码注释说明，在每次使用的时候按照注释单独实现，这种方式既繁琐又不直观又容易出错。还有一种方式是声明一个接口，通过接口的抽象方法来描述 API，抽象方法的参数可以用来描述 query 和 field 等参数，而 url 和 httpMethod 等可以通过方法注解来描述。Retrofit 选择了后者，也就是接口的方式，这种方式虽然不一定是最好的，但是比传统的方式（前者）要好很多  
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

也就是说生成一个实现类，实现每个 API 方法，每个方法的内部实现 okhttp3.Call 的组装和调用，完成请求的执行以及结果的解析和封装  
实现这样的代码生成有两种方式，一种是利用 [编译时注解和 APT 技术](https://github.com/shangmingchao/shangmingchao.github.io/blob/master/blog/android_annotation_exercises.md) 在编译时自动生成这样的实现类然后手动创建对象，一种是利用 [运行时注解和动态代理技术](https://github.com/shangmingchao/shangmingchao.github.io/blob/master/blog/design_pattern_proxy.md) 在运行时动态生成代理对象。Retrofit 选择了后者，即在运行时通过反射解析接口并生成代理对象  

### create

Retrofit 的 `create` 就是用来创建代理对象的，这个代理对象封装并实现了具体的网络请求逻辑。通过 [设计模式：代理模式](https://github.com/shangmingchao/shangmingchao.github.io/blob/master/blog/design_pattern_proxy.md) 一文我们已经知道，动态代理会把代理对象的 `hashCode()`, `equals()`, `toString()`, `getASingleUser()` 方法的调用都转发给 `InvocationHandler` 的 `invoke` 处理，所以只需要在 `invoke` 完成对 API 的解析和请求逻辑的封装即可，而 `hashCode()`, `equals()`, `toString()` 这些方法不需要特别处理：  

```java
if (method.getDeclaringClass() == Object.class) {
  return method.invoke(this, args);
}
```

对于 `getASingleUser()` 方法，只需要把这个方法封装为 `ServiceMethod` 并调用它的 `invoke` 实现

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
  ...
  if (annotation instanceof GET) {
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
...
if (callFactory == null) {
  callFactory = new OkHttpClient();
}
...
okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
```

至此 Retrofit 已经组装好 okhttp3.Call 了，已经可以用了  
而 Retrofit 灵活的一点是可以对这个 okhttp3.Call 进行封装，如封装成 `retrofit2.Call`、`CompletableFuture`、`io.reactivex.Observable` 等，这样你就可以随意选择自己喜欢的异步技术进行网络请求了  
Retrofit 利用 `CallAdapter.Factory` 完成对 okhttp3.Call 的封装，利用 `Converter.Factory` 完成对请求响应的解析  

Retrofit 默认有两个 `CallAdapter.Factory`，一个可以把 okhttp3.Call 封装成 `CompletableFuture`，一个可以把 okhttp3.Call 封装成 `retrofit2.Call`  

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
Retrofit 这种 **“ 接口 + 抽象方法 + 注解 ”** 的方式虽然可以实现 API 的描述，但是不能可视化，不能结构化，不能文档化，不能 直接mock，不能自动化测试，不能指定公共参数。所以我觉得换成 **“ 配置文件 + 插件 ”** 等其它方式可能要更好一点  

## ViewStub 29

只是一个占位，继承 `View`，不绘制（`setWillNotDraw(true)`），且宽高为 0（`setMeasuredDimension(0, 0)`），`inflate()` 就是从父容器中移除自己并 inflate 给定的 view 到自己的本来的位置（index 和 layoutParams），由于移除后就不知道自己的父容器了所以 `inflate()` 只能调用一次。ViewStub 的 `setVisibility()` 方法一般不建议使用，如果用，那么在没 `inflate()` 的情况下会自动调用 `inflate()`  

## Handler 29

`Looper.prepare();` 可以给一个普通线程关联一个消息队列，`Looper.loop();` 开始循环处理消息队列中的消息，new 一个 `Handler` 可以发送和处理消息，创建 `Handler` 需要指定 `Looper`，如果不指定那么表明是针对当前线程的 `Looper` 的，主线程有个创建好的 `Looper.getMainLooper()` 单例可以直接用  

`Looper.loop();` 是个死循环，循环获取队列中的消息，转发给消息的 target 去处理，也就是当初发送它的 `Handler` 去处理  
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
`IdleHandler` 可以在消息队列中的消息都处理完了，进入休眠之前做一些工作，所以可以利用 `Looper.myQueue().addIdleHandler()` 做一些延迟任务，如在主线程中延迟初始化一些大对象或做一些可能耗时的操作  
`Handler` 的延迟发消息功能如 `sendMessageDelayed()`，`postDelayed()` 是通过延迟唤醒实现的，在消息入队的时候就确定好消息要唤醒的时间，即 `msg.when = SystemClock.uptimeMillis() + delayMillis`，插入自己在队列中应该出现的位置，在取下一个消息时延迟 `nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE)` 时间去取即可  

> 线程池技术保持线程活跃也是通过 epoll 机制实现的，死循环中阻塞地从队列中取消息，利用 `LockSupport.park()` 或 `Condition` 的 `await()` 就能让线程保持活跃的同时让出 CPU 等待事件  

## RecyclerView 1.1.0

### 术语

目的: 在有限的窗口中显示大量的数据  
`Adapter`: 为数据集中的数据项提供对应的 View  
`Position`: 数据项在数据集中的位置  
`Index`: View 在容器中的位置  
`Binding`: 为 position 位置的数据准备对应 View 的过程  
`Recycle (view)`: 之前使用过的 View 可能会被放到缓存中，之后在显示相同类型数据时可以直接拿出来重用  
`Scrap (view)`: 进入临时 detached 状态的 View，不用完全 detached 就能重用  
`Dirty (view)`: 在被显示前必须重新绑定的 View  
LayoutManager 维护的 `LayoutPosition` 理论上和 Adapter 维护的 `AdapterPosition` 是一样的，但是对数据集 position 的修改是马上就起效的，而修改布局的 position 需要一定时间。所以最好在写 LayoutManager 的时候用 `LayoutPosition`，写 Adapter 的时候用 `AdapterPosition`  
如果数据集发生了变化，而你想通过 Diff 算法提升性能（只刷新必要的 View），那么可以直接使用 `ListAdapter` 这个 Adapter，它会在后台线程中比较新 List 和 旧 List 从而自动完成局部更新。或者在自己的 Adapter 中直接使用 `AsyncListDiffer` 实现。再或者直接使用 `DiffUtil` 工具比较列表也可以，更灵活，但也更繁琐  
如果列表是有序的，那么使用 `SortedList` 可能比使用 List 要更好一些  

### onMeasure

作为容器，RecyclerView 要干两件事，在 `onMeasure()` 中确定自己和孩子的尺寸，在 `onLayout()` 中布局孩子的位置  

如果没指定 `LayoutManager` 或者它的宽高都是 `MeasureSpec.EXACTLY` 的，那么就跟普通 View 一样默认测量就行了  

```java
if (mLayout == null) {
    defaultOnMeasure(widthSpec, heightSpec);
    return;
}
if (mLayout.isAutoMeasureEnabled()) {
    final int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    final boolean measureSpecModeIsExactly =
            widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
    if (measureSpecModeIsExactly || mAdapter == null) {
        return;
    }
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
    }
    mLayout.setMeasureSpecs(widthSpec, heightSpec);
    mState.mIsMeasuring = true;
    dispatchLayoutStep2();
    mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    if (mLayout.shouldMeasureTwice()) {
        mLayout.setMeasureSpecs(
                MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    }
} else {
    if (mHasFixedSize) {
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        return;
    }
    ...
}
```

大部分 `LayoutManager` 的 `isAutoMeasureEnabled()` 都是 `true`，表示使用 RecyclerView 的 **自动测量机制** 进行测量，此时，`LayoutManager#onMeasure(Recycler, State, int, int)` 内部只是调用了 `defaultOnMeasure()`，不要重写这个方法。`false` 的话就得重写  

**第一步** `dispatchLayoutStep1()` 主要处理 Adapter 的更新和动画运行 `processAdapterUpdatesAndSetAnimationFlags();`，保存动画过程中的 View 状态 `mViewInfoStore.addToPreLayout(holder, animationInfo);`，必要的话还会预布局并保存信息 `recordPreLayoutInformation()`  
**第二步** `dispatchLayoutStep2()` 对孩子进行真正的测量和布局 `mLayout.onLayoutChildren(mRecycler, mState);`  

> 你会发现 `dispatchLayoutStep2()` 可能被调用多次，你也会发现 `dispatchLayoutStep1()` 和 `dispatchLayoutStep2()` 既有测量的功能也有布局的功能，虽然在 measure 里布局有点奇怪，但是在真正 layout 的时候能省这两步时间  

## onLayout

`onLayout()` 中只是调用 `dispatchLayout()` 方法真正开始对子 View 进行布局

```java
void dispatchLayout() {
    if (mAdapter == null) {
        Log.e(TAG, "No adapter attached; skipping layout");
        return;
    }
    if (mLayout == null) {
        Log.e(TAG, "No layout manager attached; skipping layout");
        return;
    }
    mState.mIsMeasuring = false;
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
            || mLayout.getHeight() != getHeight()) {
        // First 2 steps are done in onMeasure but looks like we have to run again due to
        // changed size.
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```

**最后一步** `dispatchLayoutStep3();` 记录并开始 View 动画 `mViewInfoStore.process(mViewInfoProcessCallback);`，然后做一些必要的清理工作  

### onDraw

`onDraw()` 中只需要绘制 `ItemDecoration` 即可：  

```java
mItemDecorations.get(i).onDraw(c, this, mState);
```

但是对于边界装饰的绘制或者 `ItemDecoration#onDrawOver()` 的实现就要重写 `draw()` 方法了  

```java
mItemDecorations.get(i).onDrawOver(c, this, mState);
```

### 缓存

在第二步布局时 `dispatchLayoutStep2()` 会调用 `mLayout.onLayoutChildren(mRecycler, mState);`，所以在 `onLayoutChildren()` 中完成 View 的获取  
以 `LinearLayoutManager` 的 `onLayoutChildren()` 为例，它的布局算法就是先检查孩子和其它变量，寻找锚点坐标，然后从尾到头的方向填充以及从头到尾的方向填充（`fill()`），然后滚动以满足需要  
`fill()` 是一个神奇的方法，它可以在指定方向上填充满子 View  

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
         RecyclerView.State state, boolean stopOnFocusable) {
    ...
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        ...
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        ...
    }
    ...
}
```

`layoutChunk()` 中就是取 View 的过程 `View view = layoutState.next(recycler);`  

```java
View next(RecyclerView.Recycler recycler) {
    if (mScrapList != null) {
        return nextViewFromScrapList();
    }
    final View view = recycler.getViewForPosition(mCurrentPosition);
    mCurrentPosition += mItemDirection;
    return view;
}
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                                                 boolean dryRun, long deadlineNs) {
    ...
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
    }
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }
    if (holder == null) {
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
        }
        if (holder == null && mViewCacheExtension != null) {
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
            }
        }
        if (holder == null) {
            holder = getRecycledViewPool().getRecycledView(type);
        }
        if (holder == null) {
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
        }
    }
    ...
    return holder;
}
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    final int scrapCount = mAttachedScrap.size();
    for (int i = 0; i < scrapCount; i++) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
            return holder;
        }
    }
    if (!dryRun) {
        View view = mChildHelper.findHiddenNonRemovedView(position);
        if (view != null) {
            final ViewHolder vh = getChildViewHolderInt(view);
            mChildHelper.unhide(view);
            mChildHelper.detachViewFromParent(layoutIndex);
            scrapView(view);
            return vh;
        }
    }
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        if (!holder.isInvalid() && holder.getLayoutPosition() == position
                && !holder.isAttachedToTransitionOverlay()) {
            if (!dryRun) {
                mCachedViews.remove(i);
            }
            return holder;
        }
    }
    return null;
}
```

所以取 ViewHolder(View) 的过程大体是这样的  

- `getChangedScrapViewForPosition()`  *isPreLayout*  
- **`getScrapOrHiddenOrCachedHolderForPosition()`**  
  - `mAttachedScrap`  
  - `findHiddenNonRemovedView()`
  - `mCachedViews`
- `getScrapOrCachedViewForId()` *hasStableIds*  
- `getChildViewHolder()`  *mViewCacheExtension*  
- **`getRecycledViewPool().getRecycledView()`**  
- `mAdapter.createViewHolder()`  

每次 layout 或者 scroll 的时候都会取 ViewHolder(View) 来更新 RecyclerView 的渲染（`dispatchLayoutStep2()` 最终会调用 `fill()`，`scrollBy()` 最终也会调用 `fill()`，`fill()` 就是不断地取 ViewHolder(View) 来填充满容器）  
所以对于屏幕中已经有的 View 直接用即可，这就是 `mAttachedScrap`，`ArrayList<ViewHolder>` 类型的  
对于动画过程中隐藏的 View 也可以直接用，这就是 `mHiddenViews`，`List<View>` 类型的  
对于刚刚划出屏幕的 View 是可以马上拿过来直接用或复用，这就是 `mCachedViews`，`ArrayList<ViewHolder>` 类型的  
对于想要多个 RecyclerView 共享的 View，可以使用 RecycledViewPool（如果不显式指定的话每个 RecyclerView 都会创建自己的 RecycledViewPool），这个 RecycledViewPool 里会维护一个 `SparseArray<ScrapData>`，key 就是 viewType，值是 `ArrayList<ViewHolder>` 类型的  

所以，真正的缓存（真正回收复用的缓存）有两个，一个是 `mCachedViews`，默认容量为 2，超过了就从头删（FIFO）:  

```java
if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
    recycleCachedViewAt(0);
    cachedViewSize--;
}
```

一个是 RecycledViewPool 中的 `ScrapData`，默认容量为 5，超过了直接丢弃:  

```java
if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
    return;
}
```

RecyclerView 缓存灵活的一点是可以通过 `setViewCacheExtension` 自定义 RecycledViewPool 前一级缓存  

### RecyclerView 分析

RecyclerView 统一了传统滚动列表（ListView / GridView），并且做了完善，更灵活也更强大，尤其是对 View 的回收复用极大程度上避免了不必要的视图绑定过程。但是过分追求灵活，过分追求性能也暴露出了缺陷，比如说 RecyclerView 类自己本身的代码量就膨胀到几万行，代码注释中的 ugly, TODO, consider 等描述也是让人哭笑不得，设计的越复杂越容易出 Bug。而 RecyclerView 使用起来也不够简洁，绝大部分情况下列表都是简单的竖向的传统列表，`LayoutManager` 却不能缺省，忘写了也不会报错，分割线不能缺省也不能静态指定，Adapter 的模板代码也非常多  

## RxJava 3.0.0

RxJava 是 ReactiveX 的 Java 实现，往高了说它提供了响应式和函数式编程范式，优雅地解决了异步和基于事件的编程问题，而实际上只是弥补了传统 Java 异步困难的设计缺陷。在传统 Java 中同步获取单个值是这样的 `T getData()`，同步获取多个值是这样的 `Iterable<T> getData()`。但是异步获取单个值却要开线程返回个包装类 `Future<T> getData()`，然后利用 `Future` 的 `get()` 方法才能拿到真正的值，而异步获取多个值好像只能这样了 `Iterable<Future<T>> getData()`。而问题是 `Future` 的 `get()` 方法是阻塞的，会阻塞之后代码的执行，这在逻辑稍微复杂（一个 Future 依赖另一个 Future）时阻塞会变成性能噩梦，而过多的回调和回调嵌套会变成回调地狱  
所以，传统 Java 的 Future 在面对复杂异步操作时变得越来越无能为力，而 RxJava 的异步流正好弥补了这个缺陷，让所有的操作都是基于数据流的，异步对于流来说也仅仅是个操作符，逻辑一下子清晰了，代码一下子简洁了  

### Observable

`Observable` 就是数据源，既可以同步也可以异步地发射数据，既可以发射单个值也可以发射多个值，它是个抽象类，所以需要通过它的 `Create`、`Just`、`From` 等操作符来创建具体的 `Observable`:  

```java
Observable
    .create(emitter -> {
        emitter.onNext("Hello World");
        emitter.onComplete();
    })
    .subscribe(System.out::println);
```

`Create` 操作符创建的是 `ObservableCreate`，`Just` 创建的是 `ObservableJust`，它们根据自己的需要实现 `subscribeActual()` 方法，如 `Create` 的:  

```java
public final class ObservableCreate<T> extends Observable<T> {
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        ...
        source.subscribe(parent);
        ...
    }
}
```

`source` 就是我们传的用来发射数据的 `ObservableOnSubscribe`，`parent` 就是封装了我们定义的观察者的 `CreateEmitter`。而 `Just` 的实现更简单，直接调用 `observer.onNext(value);` 和 `observer.onComplete();`  

### Observer

声明观察者很简单，`onNext()`、`onError()`、`onComplete()` 中处理到来的数据或事件，`onSubscribe()` 中进行取消订阅的处理  

### subscribe

`Observable` 是一个异步 push 的数据流，所以每次订阅一个观察者时都调用 `subscribeActual()` 发射数据即可:  

```java
@Override
public final void subscribe(@NonNull Observer<? super T> observer) {
    Objects.requireNonNull(observer, "observer is null");
    try {
        ...
        subscribeActual(observer);
    }
    ...
}
```

即使你订阅（`subscribe()`）的时候不传 `Observer`，如传一个只处理数据的 Lambda，最终也会被封装成 `Observer`  

### 常用操作符

比较常用的包括 `Map`，`FlatMap`，`Filter`，`SubscribeOn`，`ObserveOn` 等操作符详见 [RxJava 常用操作符](https://github.com/shangmingchao/shangmingchao.github.io/blob/master/blog/rxjava_operators.md)  

### RxJava 分析

RxJava 真正强大的地方不是 `Observable` 和 `Observer`，不是观察者模式，而是丰富且强大的操作符，这些操作符让异步变得清晰简单，又避免了复杂的错误处理过程。但是操作符又多又强大既是它的优点也是它的缺点，使用者需要花费大量时间和精力去学习各种操作符，而且很容易被滥用。所以我觉得应该在复杂业务逻辑时使用它，简单的不需要异步处理的逻辑应该尽量避免直接使用它  

## LiveData 2.2.0

Android 迫切需要一个能自动跟页面生命周期绑定的 `Observable`。但是 `Activity`，`Fragment`，`View` 三者并没有统一实现页面（生命周期）相关的逻辑，导致其他组件尊重这些 UI 组件生命周期的过程变得尤为困难，最简单暴力的方式就是在它们的生命周期方法回调中进行额外的处理，但是弊端很明显，特别繁琐，不容易统一控制，而且万一忘记了处理也不会马上发现  
Android 团队终于意识到了这一点并从设计上设计了一个跟生命周期绑定的数据容器 `LiveData`，它严格意义上来说并不是自动跟生命周期绑定，只能说是能自动解绑，因为它的核心方法 `observe(LifecycleOwner, Observer<? super T>)` 除了第二个参数观察者还是需要主动提供一个 UI 组件上下文（`LifecycleOwner`）的  
`LiveData` 的实现很简单，就是存储 `LifecycleOwner` 和对应的 `Observer`，通过观察 `LifecycleOwner` 的状态来决定何时移除 `Observer`，来决定在什么状态下才可以通知 `Observer`  

```java
private SafeIterableMap<Observer<? super T>, ObserverWrapper> mObservers = new SafeIterableMap<>();
...
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    ...
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    ...
    owner.getLifecycle().addObserver(wrapper);
}

class LifecycleBoundObserver extends ObserverWrapper implements LifecycleEventObserver {
    ...
    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        Lifecycle.State currentState = mOwner.getLifecycle().getCurrentState();
        if (currentState == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        Lifecycle.State prevState = null;
        while (prevState != currentState) {
            prevState = currentState;
            activeStateChanged(shouldBeActive());
            currentState = mOwner.getLifecycle().getCurrentState();
        }
    }
    ...
}
```

可以看到:  
由于 `LifecycleOwner` 和 `Observer` 是一对多的关系，所以利用 Map 存储 `LifecycleOwner` 和对应的 `Observer`s，key 是 `Observer`，value 是 `LifecycleBoundObserver`（LifecycleOwner, Observer）  
只有 `STARTED` 和 `RESUMED` 状态下的 `LifecycleOwner` 才算是活跃的  
只有活跃状态下的 `Observer` 才能接收通知  
当 `LifecycleOwner` 状态变成活跃时，通知 `Observer`（这点至关重要，这也是 LiveData 最亮眼的一点，很多时候离开了某个页面，但是当对应的数据发生变化时，而我们又不希望马上更新一个看不见的页面，而是回到页面时才对页面进行更新，这种情况下 LiveData 就能很好地完成任务）  
当 `LifecycleOwner` 状态变成 `DESTROYED` 时，移除 `Observer`  
移除 `Observer` 时移除对应的 `LifecycleOwner` 的观察者  
当 `Observer` 的数量由 0 到 1 时调用 `LiveData` 的 `onActive()`，从 1 到 0 时调用 `onInactive()`  

### LiveData 分析

`LiveData` 把数据对象和页面生命周期关联了起来，而且是以一种简单的方式实现，它可以满足大部分数据驱动视图的需求。但是 `LiveData` 只能只是简单持有一个数据对象，缺乏流式处理的灵活性。而且由于封装数据，导致与其他同步或异步模块交互时需要额外的封装和解封装过程  

## 参考

- [Glide](https://github.com/bumptech/glide)
- [Retrofit](https://github.com/square/retrofit)
- [RxJava](https://github.com/ReactiveX/RxJava)
