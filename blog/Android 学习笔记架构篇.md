# Android 学习笔记架构篇

## 架构原则

### 关注点分离

* 一个组件应该只关注一个简单的问题，只负责完成一项简单的任务，应该尽少依赖其它组件
* 就算依赖另一个组件，也不能同时依赖它下下一级的组件，要像网络协议分层一样简单明确
* `Activity` 和 `Fragment` 作为操作系统和应用之间的粘合类，不应该将所有代码写在它们里面，它们甚至可以看成是有生命周期的普通 View，大部分情况下就是 **被** 用来简单 **显示数据的**  

### 模型驱动视图

* 为了保证数据 model 和它对应显示的 UI 始终是一致的，应该用 model 驱动 UI，而且最好是是持久化 model。model 是负责处理应用数据的组件，只关心数据

### 单一数据源

* 为了保证数据的一致性，必须实现相同的数据来自同一个数据源。如: 好友列表页显示了好友的备注名，数据来源于服务器的 `api/friends` 响应，好友详情页也显示了好友的备注名，数据来源于服务器的 `api/user` 响应，此时在好友详情页更改了对这个好友的备注名，那么好友列表并不知情，它的数据模型并没有发生变化，所以还是显示原来的备注名，这就产生了数据不一致的问题
* 要实现单一数据源（Single source of truth），最简单的方式就是将本地数据库作为单一数据源，主键和外键的存在保证了数据对应实体的一致性

## 推荐架构

![arch](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/android_notes_arch_1.png.png)  
Android Jetpack 组件库中有一个叫 Architecture Components 的组件集，里面包含了 Data Binding，Lifecycles，LiveData，Navigation，Paging，Room，ViewModel，WorkManager 等组件的实现

* `ViewModel` 用来为指定的 UI 组件提供数据，它只负责根据业务逻辑获取合适的数据，他不知道 View 的存在，所以它不受系统销毁重建的影响，一般它的生命周期比 View 更长久  
![viewmodel](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/android_notes_arch_2.png.png)  
* `LiveData` 是一个数据持有者，它持有的数据可以是任何 Object 对象。它类似于传统观察者模式中的 Observable，当它持有的数据发生变化时会通知它所有的 Observer。同时它还可以感知 Activity，Fragment 和 Service 的生命周期，只通知它们中 active 的，在生命周期结束时自动取消订阅
* `Activity/Fragment` 持有 `ViewModel` 进行数据的渲染，`ViewModel` 持有 `LiveData` 形式的数据以便尊重应用组件的生命周期，但是获取 `LiveData` 的具体实现应该由 **Repository** 完成
* **Repository** 是数据的抽象，它提供简洁一致的操作数据的 API，内部封装好对持久化数据、缓存数据、后台服务器数据等数据源数据的操作。所以 `ViewModel` 不关心数据具体是怎么获得的，甚至可以不关心数据到底是从哪拿到的

## 实践

### 基础设施建设

创建项目时要勾选 【Use AndroidX artifacts】 复选框以便自动使用 AndroidX 支持库，否则需要手动在 `gradle.properties` 文件中添加

```groovy
android.useAndroidX=true
android.enableJetifier=true
```

然后在项目根目录创建 `versions.gradle` 文件，以便统一管理依赖和版本号

```groovy
ext.deps = [:]

def build_versions = [:]
build_versions.min_sdk = 14
build_versions.target_sdk = 28
ext.build_versions = build_versions

def versions = [:]
versions.android_gradle_plugin = "3.3.0"
versions.support = "1.1.0-alpha01"
versions.constraint_layout = "1.1.3"
versions.lifecycle = "2.0.0"
versions.room = "2.1.0-alpha04"
versions.retrofit = "2.5.0"
versions.okhttp = "3.12.1"
versions.junit = "4.12"
versions.espresso = "3.1.0-alpha4"
versions.atsl_runner = "1.1.0-alpha4"
versions.atsl_rules = "1.1.0-alpha4"

def deps = [:]

deps.android_gradle_plugin = "com.android.tools.build:gradle:$versions.android_gradle_plugin"

def support = [:]
support.app_compat = "androidx.appcompat:appcompat:$versions.support"
support.v4 = "androidx.legacy:legacy-support-v4:$versions.support"
support.constraint_layout = "androidx.constraintlayout:constraintlayout:$versions.constraint_layout"
support.recyclerview = "androidx.recyclerview:recyclerview:$versions.support"
support.cardview = "androidx.cardview:cardview:$versions.support"
support.design = "com.google.android.material:material:$versions.support"
deps.support = support

def lifecycle = [:]
lifecycle.runtime = "androidx.lifecycle:lifecycle-runtime:$versions.lifecycle"
lifecycle.extensions = "androidx.lifecycle:lifecycle-extensions:$versions.lifecycle"
lifecycle.java8 = "androidx.lifecycle:lifecycle-common-java8:$versions.lifecycle"
lifecycle.compiler = "androidx.lifecycle:lifecycle-compiler:$versions.lifecycle"
deps.lifecycle = lifecycle

def room = [:]
room.runtime = "androidx.room:room-runtime:$versions.room"
room.compiler = "androidx.room:room-compiler:$versions.room"
deps.room = room

def retrofit = [:]
retrofit.runtime = "com.squareup.retrofit2:retrofit:$versions.retrofit"
retrofit.gson = "com.squareup.retrofit2:converter-gson:$versions.retrofit"
deps.retrofit = retrofit

deps.okhttp_logging_interceptor = "com.squareup.okhttp3:logging-interceptor:${versions.okhttp}"

deps.junit = "junit:junit:$versions.junit"

def espresso = [:]
espresso.core = "androidx.test.espresso:espresso-core:$versions.espresso"
deps.espresso = espresso

def atsl = [:]
atsl.runner = "androidx.test:runner:$versions.atsl_runner"
deps.atsl = atsl

ext.deps = deps
```

以显示 **谷歌的开源仓库列表**（[https://api.github.com/users/google/repos](https://api.github.com/users/google/repos)）为例，先依赖好 `ViewModel`、`LiveData` 和 `Retrofit`:

```groovy
apply plugin: 'com.android.application'

android {
    compileSdkVersion build_versions.target_sdk
    defaultConfig {
        applicationId "cn.frank.sample"
        minSdkVersion build_versions.min_sdk
        targetSdkVersion build_versions.target_sdk
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    lintOptions {
        abortOnError false
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation deps.support.app_compat
    implementation deps.support.constraint_layout

    implementation deps.lifecycle.runtime
    implementation deps.lifecycle.extensions
    annotationProcessor deps.lifecycle.compiler
    implementation deps.room.runtime
    annotationProcessor deps.room.compiler
    implementation deps.retrofit.runtime
    implementation deps.retrofit.gson
    implementation deps.okhttp_logging_interceptor

    testImplementation deps.junit
    androidTestImplementation deps.atsl.runner
    androidTestImplementation deps.espresso.core
}
```

然后根据习惯合理地设计源码的目录结构，如  
![dic](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/android_notes_arch_3.png.png)

```java
public class RepoRepository {

    private static RepoRepository sInstance;

    public RepoRepository() {
    }

    public static RepoRepository getInstance() {
        if (sInstance == null) {
            synchronized (RepoRepository.class) {
                if (sInstance == null) {
                    sInstance = new RepoRepository();
                }
            }
        }
        return sInstance;
    }

    public LiveData<List<Repo>> getRepo(String userId) {
        final MutableLiveData<List<Repo>> data = new MutableLiveData<>();
        ServiceGenerator.createService(GithubService.class)
                .listRepos(userId)
                .enqueue(new Callback<List<Repo>>() {
                    @Override
                    public void onResponse(Call<List<Repo>> call, Response<List<Repo>> response) {
                        data.setValue(response.body());
                    }

                    @Override
                    public void onFailure(Call<List<Repo>> call, Throwable t) {

                    }
                });
        return data;
    }

}
```

```java
public class RepoViewModel extends AndroidViewModel {

    private LiveData<List<Repo>> repo;
    private RepoRepository repoRepository;

    public RepoViewModel(@NonNull Application application) {
        super(application);
        this.repoRepository = ((SampleApp) application).getRepoRepository();
    }

    public void init(String userId) {
        if (this.repo != null) {
            return;
        }
        this.repo = repoRepository.getRepo(userId);
    }

    public LiveData<List<Repo>> getRepo() {
        return repo;
    }
}
```

```java
public class RepoFragment extends Fragment {

    private static final String ARG_USER_ID = "user_id";

    private RepoViewModel viewModel;
    private TextView repoTextView;

    public RepoFragment() {

    }

    public static RepoFragment newInstance(String userId) {
        RepoFragment fragment = new RepoFragment();
        Bundle args = new Bundle();
        args.putString(ARG_USER_ID, userId);
        fragment.setArguments(args);
        return fragment;
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View rootView =  inflater.inflate(R.layout.fragment_repo, container, false);
        repoTextView = (TextView) rootView.findViewById(R.id.repo);
        return rootView;
    }

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        Bundle args = getArguments();
        if (args != null) {
            String userId = args.getString(ARG_USER_ID);
            viewModel = ViewModelProviders.of(this).get(RepoViewModel.class);
            viewModel.init(userId);
            viewModel.getRepo().observe(this, new Observer<List<Repo>>() {
                @Override
                public void onChanged(List<Repo> repos) {
                    StringBuilder builder = new StringBuilder();
                    if (repos != null) {
                        for (Repo repo : repos) {
                            builder.append(repo.getFull_name()).append("\n");
                        }
                    }
                    repoTextView.setText(builder);
                }
            });
        }
    }

}
```

这是最简单直接的实现，但还是存下很多模板代码，还有很多地方可以优化

1. 既然 View 是和 ViewModel 绑定在一起的，那为什么每次都要先 `findViewById()` 再 `setText()` 呢？在声明或者创建 View 的时候就给它指定好对应的 ViewModel 不是更简单直接么
2. `ViewModel` 的实现真的优雅吗？`init()` 方法和 `getRepo()` 方法耦合的严重么？`ViewModel` 应该在什么时刻开始加载数据？
3. 网络请求的结果最好都缓存到内存和数据库中，既保证了单一数据源原则又能提升用户体验

#### Data Binding

对于第一个问题，Data Binding 组件是一个还算不错的实现，可以在布局文件中使用 **表达式语言** 直接给 View 绑定数据，绑定可以是单向的也可以是双向的。Data Binding 这样绑定可以避免内存泄漏，因为它会自动取消绑定。可以避免空指针，因为它会宽容评估表达式。可以避免同步问题，可以在后台线程更改非集合数据模型，因为它会在评估时本地化数据  
为了使用 Data Binding，需要在 app module 的 `build.gradle` 文件中添加

```groovy
dataBinding {
    enabled = true
}
```

利用 `@{}` 语法可以给 View 的属性绑定数据变量，但是该表达式语法应该尽可能简单直接，复杂的逻辑应该借助于自定义 `BindingAdapter`  

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"/>
   </LinearLayout>
</layout>
```

不需要重新编译代码，构建工具就会为每个这样的布局文件自动生成一个对应的绑定类，继承自 `ViewDataBinding`，路径为 `app/build/generated/data_binding_base_class_source_out/debug/dataBindingGenBaseClassesDebug/out/cn/frank/sample/databinding/FragmentRepoBinding.java`，默认的类名是布局文件名的大驼峰命名加上 Binding 后缀，如 `fragment_repo.xml` 对应 `FragmentRepoBinding`，可以通过 `<data class=".ContactItem">` 自定义类名和所在包名。可以通过 `DataBindingUtil` 的 `inflate()` 等静态方法或自动生成的绑定类的 `inflate()` 等静态方法获取绑定类的实例，然后就可以操作这个实例了  

##### 操作符和关键字

这个表达式语言的 **操作符和关键字** 包括: 数学运算 `+ - / * %`，字符串拼接 `+`，逻辑 `&& ||`，二进制运算 `& | ^`，一元操作符 `+ - ! ~`，移位 `>> >>> <<`，比较 `== > < >= <=`，判断实例 `instanceof`，分组 `()`，字符/字符串/数字/`null` 的字面量，强制转化，方法调用，字段访问，数组访问 `[]`，三目运算符 `?:`，二目空缺省运算符 `??`  

```text
android:text="@{String.valueOf(index + 1)}"
android:visibility="@{age > 13 ? View.GONE : View.VISIBLE}"
android:transitionName='@{"image_" + id}'
android:text="@{user.displayName ?? user.lastName}"
android:text="@{user.lastName}"
android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
android:text="@{@string/nameFormat(firstName, lastName)}"
android:text="@{@plurals/banana(bananaCount)}"
```

小于比较符 `<` 需要转义为 `&lt;`，为了避免字符串转义单引号和双引号可以随便切换使用  
`<import>` 的类冲突时可以取别名加以区分

```xml
<import type="android.view.View"/>
<import type="com.example.real.estate.View"
        alias="Vista"/>
```

`<include>` 布局中可以传递变量

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:bind="http://schemas.android.com/apk/res-auto">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <include layout="@layout/name"
           bind:user="@{user}"/>
       <include layout="@layout/contact"
           bind:user="@{user}"/>
   </LinearLayout>
</layout>
```

不支持 `<merge>` 结合 `<include>` 的使用

##### 事件处理

View 事件的分发处理有两种机制，一种是 **Method references**，在表达式中直接通过监听器方法的签名来引用，Data Binding 会在编译时评估这个表达式，如果方法不存在或者签名错误那么编译就会报错，如果表达式评估的结果是 `null` 那么 Data Binding 就不会创建监听器而是直接设置 `null` 监听器，Data Binding 在 **绑定数据的时候** 就会创建监听器的实例: `android:onClick="@{handlers::onClickFriend}"`。一种是 **Listener bindings**，Data Binding 在 **事件发生的时候** 才会创建监听器的实例并设置给 view然后评估 lambda 表达式，`android:onClick="@{(theView) -> presenter.onSaveClick(theView, task)}"`  

##### 绑定 Observable 数据

虽然 View 可以绑定任何 PO 对象，但是所绑定对象的更改并不能自动引起 View 的更新，所以 Data Binding 内置了 `Observable` 接口和它的 `BaseObservable`，`ObservableBoolean` 等子类可以方便地将对象、字段和集合变成 observable  

```java
private static class User {
    public final ObservableField<String> firstName = new ObservableField<>();
    public final ObservableInt age = new ObservableInt();
}
```

```java
private static class User extends BaseObservable {
    private String firstName;
    private String lastName;

    @Bindable
    public String getFirstName() {
        return this.firstName;
    }

    @Bindable
    public String getLastName() {
        return this.lastName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
        notifyPropertyChanged(BR.firstName);
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
        notifyPropertyChanged(BR.lastName);
    }
}
```

##### 执行绑定

有时候绑定需要立即执行，如在 `onBindViewHolder()` 方法中:

```java
public void onBindViewHolder(BindingHolder holder, int position) {
    final T item = mItems.get(position);
    holder.getBinding().setVariable(BR.item, item);
    holder.getBinding().executePendingBindings();
}
```

Data Binding 在为 View 设置表达式的值的时候会自动选择对应 View 属性的 setter 方法，如 `android:text="@{user.name}"` 会选择 `setText()` 方法，但是像 `android:tint` 属性没有 setter 方法，可以使用 `BindingMethods` 注解自定义方法名

```java
@BindingMethods({
       @BindingMethod(type = "android.widget.ImageView",
                      attribute = "android:tint",
                      method = "setImageTintList"),
})
```

如果要自定义 setter 方法的绑定逻辑，可以使用 `BindingAdapter` 注解

```java
@BindingAdapter("android:paddingLeft")
public static void setPaddingLeft(View view, int padding) {
  view.setPadding(padding,
                  view.getPaddingTop(),
                  view.getPaddingRight(),
                  view.getPaddingBottom());
}
```

```xml
<ImageView app:imageUrl="@{venue.imageUrl}" app:error="@{@drawable/venueError}" />
```

```java
@BindingAdapter({"imageUrl", "error"})
public static void loadImage(ImageView view, String url, Drawable error) {
  Picasso.get().load(url).error(error).into(view);
}
```

如果要自定义表达式值的自动类型转换，可以使用 `BindingConversion` 注解

```xml
<View
   android:background="@{isError ? @color/red : @color/white}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```

```java
@BindingConversion
public static ColorDrawable convertColorToDrawable(int color) {
    return new ColorDrawable(color);
}
```

`ViewModel` 可以实现 `Observable` 接口并结合 `PropertyChangeRegistry` 可以更方便地控制数据更改后的行为

##### 双向绑定

使用 `@={}` 符号可以实现 View 和数据的双向绑定  

```xml
<CheckBox
    android:id="@+id/rememberMeCheckBox"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:checked="@={viewmodel.rememberMe}" />
```

```java
public class LoginViewModel extends BaseObservable {
    // private Model data = ...

    @Bindable
    public Boolean getRememberMe() {
        return data.rememberMe;
    }

    public void setRememberMe(Boolean value) {
        // 为了防止无限循环，必须要先检查再更新
        if (data.rememberMe != value) {
            data.rememberMe = value;
            saveData();
            notifyPropertyChanged(BR.remember_me);
        }
    }
}
```

自定义属性的双向绑定还需要借助 `@InverseBindingAdapter` 和 `@InverseBindingMethod`

```java
@BindingAdapter("time")
public static void setTime(MyView view, Time newValue) {
    // Important to break potential infinite loops.
    if (view.time != newValue) {
        view.time = newValue;
    }
}

@InverseBindingAdapter("time")
public static Time getTime(MyView view) {
    return view.getTime();
}
```

监听属性的更改，事件属性以 `AttrChanged` 作为后缀

```java
@BindingAdapter("app:timeAttrChanged")
public static void setListeners(
        MyView view, final InverseBindingListener attrChange) {
    // Set a listener for click, focus, touch, etc.
}
```

可以借助转换器类定制 View 的显示规则

```xml
<EditText
    android:id="@+id/birth_date"
    android:text="@={Converter.dateToString(viewmodel.birthDate)}" />
```

```java
public class Converter {
    @InverseMethod("stringToDate")
    public static String dateToString(EditText view, long oldValue,
            long value) {
        // Converts long to String.
    }

    public static long stringToDate(EditText view, String oldValue,
            String value) {
        // Converts String to long.
    }
}
```

Data Binding 内置了 `android:text`，`android:checked` 等的双向绑定

#### 生命周期敏感组件

在 Activity 或 Fragment 的生命周期方法中进行其它组件的配置并不总是合理的，如在 `onStart()` 方法中注册广播接收器 A、开启定位服务 A、启用组件 A 的监听、启用组件 B 的监听等等，在 `onStop()` 方法中注销广播接收器 A、关闭定位服务 A、停用组件 A 的监听、停用组件 B 的监听等等，随着业务逻辑的增加这些生命周期方法变得越来越臃肿、越来越乱、越来越难以维护，如果这些组件在多个 Activity 或 Fragment 上使用那么还得重复相同的逻辑，就更难以维护了。 而且如果涉及到异步甚至 **没办法保证** `onStart()` 方法中的代码 **一定** 在 `onStop()` 方法执行前执行  
**关注点分离**，这些组件的行为受生命周期的影响，所以它们自己应该意识到自己是生命周期敏感的组件，当生命周期变化时它们应该 **自己决定** 自己的行为，而不是交给生命周期的拥有者去处理  
生命周期有两个要素: 事件和状态，生命周期事件的发生一般会导致生命周期状态的改变  
生命周期敏感组件应该实现 `LifecycleObserver` 以观察 `LifecycleOwner` 的生命周期，支持库中的 Activity 和 Fragment 都实现了 `LifecycleOwner`，可以直接通过它的 `getLifecycle()` 方法获取 `Lifecycle` 实例  

```java
MainActivity.this.getLifecycle().addObserver(new MyLocationListener());
```

```java
class MyLocationListener implements LifecycleObserver {
    private boolean enabled = false;

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void start() {
        if (enabled) {
           // connect
        }
    }

    public void enable() {
        enabled = true;
        if (lifecycle.getCurrentState().isAtLeast(STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void stop() {
        // disconnect if connected
    }
}
```

`GenericLifecycleObserver` 接口继承了 `LifecycleObserver`，有一个接口方法 `onStateChanged(LifecycleOwner, Lifecycle.Event)` 表明它可以接收所有的生命周期过渡事件

#### LiveData

它的 `observe()` 方法源码

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```

说明 `LiveData` 只能在主线程中订阅，订阅的观察者被包装成生命周期组件的观察者 `LifecycleBoundObserver`

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserve
    @NonNull
    final LifecycleOwner mOwner;
    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer)
        super(observer);
        mOwner = owner;
    }
    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }
    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }
    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

当观察到生命周期状态变化时会调用 `onStateChanged()` 方法，所以当状态为 `DESTROYED` 的时候会移除数据观察者和生命周期观察者，`shouldBeActive()` 方法的返回值表明只有生命周期状态是 `STARTED` 和 `RESUMED` 的 `LifecycleOwner` 对应的数据观察者才是 active 的，只有 active 的数据观察者才会被通知到，当数据观察者 **第一次** 从 inactive 变成 active 时，**也会** 收到通知  
`observeForever()` 方法也可以订阅，但是 `LiveData` 不会自动移除数据观察者，需要主动调用 `removeObserver()` 方法移除  
`LiveData` 的 `MutableLiveData` 子类提供了 `setValue()` 方法可以在主线程中更改所持有的数据，还提供了 `postValue()` 方法可以在后台线程中更改所持有的数据  
可以继承 `LiveData` 实现自己的 observable 数据，`onActive()` 方法表明有 active 的观察者了，可以进行数据更新通知了，`onInactive()` 方法表明没有任何 active 的观察者了，可以清理资源了  
单例的 `LiveData` 可以实现多个 Activity 或 Fragment 的数据共享  
可以对 `LiveData` 持有的数据进行变换，需要借助 `Transformations` 工具类

```java
private final PostalCodeRepository repository;
private final MutableLiveData<String> addressInput = new MutableLiveData();
public final LiveData<String> postalCode =
    Transformations.switchMap(addressInput, (address) -> {
        return repository.getPostCode(address);
    });
```

```java
private LiveData<User> getUser(String id) {
  ...;
}
LiveData<String> userId = ...;
LiveData<User> user = Transformations.switchMap(userId, id -> getUser(id) );
```

`LiveData` 的 `MediatorLiveData` 子类可以 merge 多个 LiveData 源，可以像 ReactiveX 的操作符一样进行各种变换  

### 几点感悟

* 避免手动写任何模板代码，尤其是写业务代码时
* 不要用回调的思想去写 Rx 的代码，用 Stream 思想去写，思想要彻底转变过来
* 把异步的思想调整回同步:  

```text
void businessLogic() {
    showLoadingView();
    request(uri, params, new Callbacks() {

        @Override
        void onSuccess(Result result) {
            showDataView(result);
        }

        @Override
        void onFailure(Error error) {
            showErrorView(error);
        }

    });
}
```

```text
void businessLogic() async {
    showLoadingView()
    Result result = await request(uri, params)
    result.ok() ? showDataView(result) : showErrorView(result)
}
```

* **约定优于配置** ，适当的约定可以减少相当可观的劳动量
* 能自动的就不要手动，比如取消网络请求等操作要自动进行，不要出现手动操作的代码
* 避免源码中模板代码的重复，包括用工具自动生成的
* 借助编译器在编译时生成的代码就像一把双刃剑，它独立于源码库，又与源码紧密联系，要谨慎使用

### Sample

* [googlesamples/android-architecture-components](https://github.com/googlesamples/android-architecture-components)  
* [shangmingchao/Sample](https://github.com/shangmingchao/Sample)
