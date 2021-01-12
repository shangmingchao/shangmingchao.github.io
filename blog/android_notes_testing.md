# Android 学习笔记测试篇

## JUnit 单元测试

JUnit 可以方便地写测试，最新版本是 JUnit 5，但是我们更喜欢更熟悉的是 JUnit 4  

### RunWith

使用 JUnit 就不用再为每个测试专门书写包含 `main()` 函数的测试类了（这种类被称为 Runner），使用 `@RunWith` 注解就可以了  

```kotlin
@RunWith(AndroidJUnit4::class)
class MainFragmentTest {
}
```

比如在这里我们为了测试 Android 中的类，使用的是 `AndroidJUnit4` Runner，而不是 JUnit 默认的 Runner（`BlockJUnit4ClassRunner`） 或其他 Runner（`Suite`，`Parameterized`）  

### Test

在这个 Runner 中我们可以使用 `@Test` 注解标记测试函数，这个注解中可以指定这个函数期望抛出的异常，也可以指定这个函数超时时间

```kotlin
@Test(expected = IndexOutOfBoundsException::class)
fun testException() {
}
```

### Before 和 After

如果想要在所有测试方法执行之前做一些如初始化对象等工作，可以使用 `@Before` 注解这个初始化方法。同样 `@After` 方法可以做一些清理工作  

```kotlin
@Before
fun startTheKoin() {
}
@Test
fun testHttpClientModule() {
}
@After
fun stopTheKoin() {
}
```

### Rule

`@Before` 和 `@After` 大部分情况下可以满足需求，但是如果像 `Activity` 创建和销毁的逻辑虽然可以利用这两个注解的方法实现，但是每次都要写这样的逻辑既麻烦又不优雅。所以可以利用 `@Rule` 注解指定一个功能自给的 Rule 就可以自动完成了  

```kotlin
@get:Rule
var mainActivityRule = activityScenarioRule<MainActivity>()
```

这里我们使用了 `ActivityScenarioRule`，它可以自动在测试开始前启动 `Activity`，在测试结束后关闭 `Activity`  

## Android 单元测试

Android 的单元测试一般都是直接使用 JUnit，测试文件放到 `src/test` 目录下  
要自动生成测试类可以右键菜单选择 `Generate` 然后选择 `Test...`。或者直接 `command N` 快捷键，就可以在对应目录创建对应的测试文件了  
关于断言，推荐使用 Google 的 `Truth`，它比其他的断言库更加简单易读，流式调用就像写句子一样  
关于 mock，如果想 mock 出一个普通对象，推荐使用 `Mockito`。如果想 mock 复杂的 Android 对象，推荐使用 `Robolectric`  
运行测试用例的时候直接右键 `Run 'XXX'` 就可以了  

> 不建议在真机或者模拟器上运行单元测试（这种单元测试通常被称为 Instrumented 仪表化单元测试），如果有充分的的理由一定要这么做，需要把测试文件放在 `src/androidTest/` 目录下，然后可能还需要引入 Hamcrest，Espresso，UI Automator 这些库  

## Android UI 测试

如果 UI 测试只在当前 App 内，推荐使用 Espresso。如果跨 App 的话推荐使用 UI Automator  

### Espresso

Espresso 虽然在单独的线程上运行测试代码，但是它可以保证操作的时序性，也就是说操作是同步的。这里有一点要注意，窗口动画或者其他 UI 动画可能会影响这种时序性，可能会造成测试不通过，所以要要在开发者选项中关闭这些动画的执行  

- Window animation scale（窗口动画缩放） `adb shell settings put global window_animation_scale 0`
- Transition animation scale（过渡动画缩放） `adb shell settings put global transition_animation_scale 0`
- Animator duration scale（Animator 时长缩放） `adb shell settings put global animator_duration_scale 0`

Espresso 需要把默认的 Runner 设置为 AndroidJUnitRunner，所以要在脚本中的 `android defaultConfig` 中加入  

```groovy
testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
```

然后测试类使用 `@RunWith(AndroidJUnit4::class)` 注解就行了  

UI 测试的流程一般也很简单，就是:  
在这个 View 上（ViewMatcher） 执行这个操作（ViewAction） 期望的结果是这样的（ViewAssertion）  

```kotlin
onView(withId(R.id.my_view))
        .perform(click())
        .check(matches(isDisplayed()))
```

ViewMatcher 有几个方法可以方便地获取 View，如 `withId()` 可以根据 id 获取 View，`withText()` 可以获取 text 是给定值的 TextView，`withHint()` 可以获取 hint 是给定值的 TextView  
如果想要组合多个或者想要更复杂的匹配逻辑，可以使用 Hamcrest 的 `allOf()` 或 `ont()`，如 `onView(allOf(withId(R.id.button_signin), not(withText("Sign-out"))))`  
ViewAction 比较常用的有 `click()`，`longClick()`，`pressBack()`，`swipeLeft()`，点击并输入文本 `typeText()`，点击某个键 `pressKey()`，清除文本 `clearText()`，如 `perform(click())`  
ViewAssertion 比较常用的有 `doesNotExist()`，`matches()`，可以用来判断期望的 View 状态，如 `check(matches(withText(STRING_TO_BE_TYPED)))`  

### UI Automator

UI Automator 测试框架可以让你关注设备上可见的 UI 元素，它支持跨应用的 UI 测试。但是它有一些需要注意的地方，如:  
UI 元素要有可见的 text 标签，或者 `android:contentDescription` 值  
默认的 View 都实现了可访问性（Accessibility），为了让 UI Automator 更好地工作，要保证所有自定义 View 都支持可访问性  
uiautomatorviewer 工具现在放在 `<android-sdk>/tools/bin/` 目录下了，需要先 cd 到这个目录然后执行 `./uiautomatorviewer` 命令启动，而且只支持 Java8，如果当前使用的 JDK 版本高于 8 会启动失败  
UI Automator 使用 `UiDevice` 对象控制设备，使用 `UiObject` 对象控制 UI 元素，而每次获取或者控制 `UiObject` 都是当前屏幕上显示的页面的  

```kotlin
val okButton: UiObject = device.findObject(
        UiSelector().text("OK").className("android.widget.Button")
)
```

获取 View，对 View 执行操作，断言期望的状态都跟 Espresso 差不多，只不过使用的是 `UiObject`，`UiSelector`  

### 关于 Activity Fragment 等的测试

Activity 的测试可以借助 `ActivityScenario` 完成，推荐使用 `ActivityScenarioRule`  

```kotlin
@get:Rule
var mainActivityRule = activityScenarioRule<MainActivity>()
@Test
fun testEvent() {
    val scenario = mainActivityRule.scenario
    scenario.moveToState(Lifecycle.State.RESUMED)
}
```

或者直接使用 `val scenario = launch(MainActivity::class.java)`  
有了 `ActivityScenario`，就可以把 Activity 驱动到给定的状态 `scenario.moveToState(Lifecycle.State.RESUMED)`，可以重建 `scenario.recreate()`，可以在 Activity 主线程中做一些操作 `scenario.onActivity {  }`  
Fragment 和 Activity 差不多，只不过 `FragmentScenario` 有两个方式获取，一个是 `launchFragmentInContainer()` 可以在一个空 Activity 的根视图中加载这个 Fragment，一个是 `launchFragment()` 可以在一个空 Activity 中 attach 这个 Fragment（即这个 Fragment 没有视图容器）。当然 `FragmentScenario` 是支持 `DialogFragment` 的  
对于 `Service` 的测试可以使用 `ServiceTestRule`  
对于 `ContentProvider` 的测试可以使用 `ProviderTestCase2` 进行沙箱测试  

## 测试实践

```groovy
android {
    buildTypes {
        debug {
            testCoverageEnabled true
        }
    }
    testOptions {
        unitTests.returnDefaultValues = true
        unitTests.includeAndroidResources = true
    }
}
dependencies {
    testImplementation deps.junit
    testImplementation deps.mockito
    testImplementation deps.google_truth

    androidTestImplementation deps.androidx_test.core
    androidTestImplementation deps.androidx_test.runner
    androidTestImplementation deps.androidx_test.rules
    androidTestImplementation deps.androidx_test.junit
    androidTestImplementation deps.androidx_test.junit_ktx
    androidTestImplementation deps.androidx_test.espresso_core
    androidTestImplementation deps.androidx_test.espresso_contrib
    androidTestImplementation deps.androidx_test.espresso_intents
    androidTestImplementation deps.google_truth

    debugImplementation deps.androidx_test.core
    debugImplementation deps.androidx_test.fragment
}
```

```kotlin
@RunWith(AndroidJUnit4::class)
class UserFragmentTest {
    @Test
    fun testEvent() {
        val scenario = launchFragmentInContainer<UserFragment>(bundleOf("username" to "google"))
        scenario.moveToState(Lifecycle.State.RESUMED)
        onView(withId(R.id.userTextView)).check(matches(isDisplayed()))
    }
}
```

有些开源的自动化测试框架也可以使用，如 Appium，SoloPi，Airtest  

## 参考

- [Junit 4](https://junit.org/junit4/)
- [Google Truth](https://truth.dev/)
- [Mockito](https://site.mockito.org/)
- [Appium](https://github.com/appium/appium)
- [SoloPi](https://github.com/appium/appium)
- [Airtest](https://github.com/AirtestProject/Airtest)
