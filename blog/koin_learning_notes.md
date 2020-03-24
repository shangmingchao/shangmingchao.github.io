# Koin 依赖注入学习笔记

```groovy
koin.core = "org.koin:koin-core:$versions.koin"
koin.ext = "org.koin:koin-core-ext:$versions.koin"
koin.test = "org.koin:koin-test:$versions.koin"
koin.android = "org.koin:koin-android:$versions.koin"
koin.androidx_scope = "org.koin:koin-androidx-scope:$versions.koin"
koin.androidx_viewmodel = "org.koin:koin-androidx-viewmodel:$versions.koin"
koin.androidx_fragment = "org.koin:koin-androidx-fragment:$versions.koin"
koin.androidx_ext = "org.koin:koin-androidx-ext:$versions.koin"
```

> Koin 就是个 Kotlin DSL，一个轻量的容器，一个实用的 API

Koin 相对于其他依赖注入容器最大的优点是简单纯净：它不使用代理，不使用反射，不生成额外的代码  
Koin 就是用来提供和存放对象的容器，你需要什么对象就跟 Koin 说，它会给你想要的对象  
在这之前你需要跟 Koin 声明一下你想要 Koin 帮你管理的对象，Koin 把这个声明叫做 Definition，这些被管理的对象有时也被称为组件（Component）  
Koin 有两个 DSL，一个是 Application DSL，用来对容器进行各种配置，一个是 Module DSL，用来描述将来要被注入到其它地方的各种组件

## 容器配置

要创建一个 Koin 容器配置（`KoinApplication`）有两个方法，一个是直接用 `koinApplication { }` 创建实例，一个是用 `startKoin { }` 创建实例并绑定到 `GlobalContext`  
使用 `logger()` 可以配置容器日志系统，`modules()` 指定容器要加载的 module，`properties()` 用参数配置，`fileProperties()` 用文件配置，`environmentProperties()` 用环境变量配置

## module 配置

一个 module 就是一系列 Definition 的集合：

```kotlin
module { }
```

其中

- `factory { //definition }` 提供一个工厂 bean definition
- `single { //definition }` - 提供一个单例 bean definition (也被称为 bean)
- `get()` - 解决一个组件依赖 (可以用 name，scope 或 parameters)
- `bind()` - 对给定的 bean definition 添加类型绑定
- `binds()` - 对给定的 bean definition 添加类型数组绑定
- `scope { // scope group }` - 为 scoped definition 定义一个逻辑组
- `scoped { //definition }`- 提供一个只在这个 scope 中的 bean definition

可以用 `named()` 函数以字符串或类型限定符对 definition 进行命名

## Definition

`single`，`factory` 和 `scoped` 都可以通过 lambda 的方式声明组件，通常我们是通过构造器进行实例化的，当然也可以用其他方式  
`single` 方式定义的组件表示 Koin 容器会创建并保存这个组件的唯一实例  
`single` 方式定义的组件表示每次你跟 Koin 要这个组件实例时 Koin 容器都给你创建一个新的实例，不会保存它，因为没有别的地方会使用这个实例  
使用 `get()` 函数可以解决并注入依赖：

```kotlin
class Controller(val service : BusinessService)
class BusinessService()
...
val myModule = module {
  single { Controller(get()) }
  single { BusinessService() }
}
```

默认是通过类型进行组件的声明的，也就是说，上面的 `get()` 表示跟 Koin 容器要一个 `BusinessService` 类型的实例，而第二行声明了 `BusinessService` 类型的实例该怎么创建，所以是完全没问题的  
而如果 `BusinessService` 实现了 `Service` 接口，我们跟 Koin 容器要一个 `Service` 实例怎么办？可以使用 `as` 进行转化，也可以使用类型推导：

```kotlin
// Will match type BusinessService only
single { BusinessService() }
// Will match type Service only
single { BusinessService() as Service }
```

```kotlin
// Will match type BusinessService only
single { BusinessService() }
// Will match type Service only
single<Service> { BusinessService() }
```

我们只想要一个 `Service` 或 `BusinessService` 类型的实例，却写了两行几乎一样的 definition，那能不能一个 definition 中就匹配多个类型呢？可以使用 `bind` 操作符

```kotlin
// Will match types BusinessService & Service
single { BusinessService() } bind Service::class
```

如果 `WebService` 也实现了 `Service`，怎么区分我要的 `Service` 实例是 `BusinessService` 还是 `WebService` 呢？通过类型已经不足以说明了，可以通过名字来说明，`named()` 函数可以给 definition 起个名字：

```kotlin
single<Service>(named("business")) { BusinessService() }
single<Service>(named("web")) { WebService() }
...
val service : Service by inject(name = named("business"))
```

注入的时候可以提供参数：

```kotlin
class Presenter(val view : View)
...
val myModule = module {
    single{ (view : View) -> Presenter(view) }
}
...
val presenter : Presenter by inject { parametersOf(view) }
```

可以通过把 `createOnStart` flag 设置为 true 来让容器启动时就创建实例：

```kotlin
single<Service>(createAtStart=true) { BusinessService() }
```

definition 不会考虑泛型参数，所以有时你可能不得不这样使用：

```kotlin
single(named("Ints")) { ArrayList<Int>() }
single(named("Strings")) { ArrayList<String>() }
```

## module

一个 module 用来组织一些相关的 definition，也就是说一个组件可以属于不同的 module：

```kotlin
class Repository(val datasource : Datasource)
interface Datasource
class LocalDatasource() : Datasource
class RemoteDatasource() : Datasource
...
val repositoryModule = module {
    single { Repository(get()) }
}
val localDatasourceModule = module {
    single<Datasource> { LocalDatasource() }
}
val remoteDatasourceModule = module {
    single<Datasource> { RemoteDatasource() }
}
...
// Load Repository + Local Datasource definitions
startKoin {
    modules(repositoryModule,localDatasourceModule)
}
```

## Component

为了让一个类拥有使用 Koin 的能力，可以让它实现 `KoinComponent` 接口：

```kotlin
class MyComponent : KoinComponent {
    // lazy inject Koin instance
    val myService : MyService by inject()
    // or eager inject Koin instance
    val myService : MyService = get()
}
```

去 Koin 容器中拿实例有两种办法，一种是 `val t : T by inject()`，可以延迟获取实例，一种是 `val t : T = get()` 马上获取实例

## Scope

Scope 就是作用域，就是生命周期，默认有 3 种 scope：  

- `single` 创建的对象会保持在容器整个生命周期而不被丢弃  
- `factory` 每次都会创建新的对象，不会共享，不会被容器保持  
- `scoped` 创建的对象会持续到对应 scope 的整个生命周期：

```kotlin
module {
    scope(named("A_SCOPE_NAME")){
        scoped { Presenter() }
    }
}
...
// create scope instance "myScope" (the scope Id) for scope "A_SCOPE_NAME" (the qualifier)
val scope = koin.createScope("myScope","A_SCOPE_NAME")
// resolve presenter instance
val presenter = scope.get<Presenter>()
```

## Koin Android

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidLogger()
            androidContext(this@App)
            modules(
                serializationModule, httpClientModule, webServiceModule, databaseModule,
                repositoryModule, viewModelModule
            )
        }
    }
}
```

容器的配置文件是存放在 `assets` 目录下的 `assets/koin.properties` 文件，可以通过 `androidFileProperties()` 读取
`androidContext()` 或 `androidApplication()` 函数可以用来获取上下文对象  
Activity，Service 等系统级组件是系统负责创建和维护的，Koin 无法进行干预，而且 Koin 必须尊重它们的生命周期
像 WebService，Dao 这些生命周期长到跟应用一样长的组件，用 `single` 就可以了  
但像 ViewModel 等生命周期很短的组件怎么办呢？它们一般会跟随页面（Activity/Fragment）的创建和销毁，所以我们可以利用 scope 进行规范：

```kotlin
val androidModule = module {
    scope(named<MyActivity>()) {
        scoped { Presenter() }
    }
}
...
class MyActivity : AppCompatActivity() {

    // inject Presenter instance from current scope
    val presenter : Presenter by currentScope.inject()
}
```

需要共享一个实例时可以这样用：

```kotlin
module {
    // Shared user session data
    scope(named("session")) {
        scoped { UserSession() }
    }
    // Inject UserSession instance from "session" Scope
    factory { (scopeId : ScopeID) -> Presenter(getScope(scopeId).get())}
}
...
val ourSession = getKoin().createScope("ourSession",named("session"))
...
class MyActivity1 : AppCompatActivity() {
    val userSession : UserSession by ourSession.inject()
}
class MyActivity2 : AppCompatActivity() {
    val userSession : UserSession by ourSession.inject()
}
```

用完 scope 记得释放：

```kotlin
val ourSession = getKoin().getScope("ourSession")
ourSession.close()
```

`viewModel` 关键字是 `single` 和 `factory` 的补充，用来声明 ViewModel 并把它绑定到系统组件的生命周期：

```kotlin
val appModule = module {
    viewModel { DetailViewModel(get(), get()) }
}
```

`by viewModel()` 可以延迟注入，`getViewModel()` 可以马上注入  
共享的 ViewModel 可以通过 `by viewModel<T>()` 或 `getSharedViewModel<T>()` 注入，需要限定符作为 Tag
ViewModel 除了可以使用普通的注入参数，还可以使用 `Bundle`  
使用 `fragmentFactory()` 可以让容器安装一个 `KoinFragmentFactory` 实例，然后就可以构造器注入了：

```kotlin
val appModule = module {
    single { MyService() }
    fragment { MyFragment(get()) }
}
```

Activity 可以使用 `setupKoinFragmentFactory()` 安装 Fragment 工厂，当然也可以指定 Scope：

```kotlin
val appModule = module {
    scope<MyActivity> {
        scoped { MyFragment(get()) }
    }
}
...
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        setupKoinFragmentFactory(currentScope)
        super.onCreate(savedInstanceState)
    }
}
```

Android MVVM 中最直观的应用是这样的：Fragment 中注入 `ViewModel`：

```kotlin
class MyFragment : Fragment() {
    private val repoViewModel: RepoViewModel by viewModel { parametersOf(Bundle(), "google") }
    ...
}
```

`ViewModel` 中注入 Repository 和其他参数：

```kotlin
class MyViewModel(
    private val handle: SavedStateHandle,
    private val myParam: String,
    private val repository: Repository
) : ViewModel() {
    ...
}
```

Repository 中注入 WebService 和 Dao：

```kotlin
class Repository(
    private val webService: WebService,
    private val dao: Dao
) {
    ...
}
```

你会发现，除了 `by viewModel` 其它的没有依赖注入的痕迹，很整洁很直观

## 测试

```kotlin
class DITest : KoinTest {
    @Before
    fun startTheKoin() {
        startKoin {
            modules(
                serializationModule, httpClientModule, webServiceModule, databaseModule,
                repositoryModule, viewModelModule
            )
        }
    }
    @Test
    fun testHttpClientModule() {
        val retrofit1 = get<Retrofit>(named(RETROFIT_GITHUB))
        val retrofit2 = get<Retrofit>(named(RETROFIT_GITHUB))
        assertEquals(retrofit1, retrofit2)
    }
    @After
    fun stopTheKoin() {
        stopKoin()
    }
}
```

## 参考

- [Koin Documentation References](https://doc.insert-koin.io/#/)
