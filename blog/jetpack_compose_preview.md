# Jetpack Compose 新的尝试

## 概述

Android 中的布局文件是借助 XML 实现的，描述的很直观，也很容易复用，但是 XML 毕竟只是简单的标记语言，只能用来描述页面结构，而数据和页面元素的关系以及其他复杂的业务逻辑还需要通过其他程序代码主动处理。在 Activity 中，通过显式编程的方式解析 XML 文件找到你的控件，然后通过同步或者异步的方式获取控件相关的数据，最后将数据显示到控件上。这是一个很传统、很简单、很有效的流程，但是随着需求的不断变化，越来越多的弊端暴露了出来：  

- **文件数剧增**：由于布局文件是作为资源而非源码存在的，所以只能存放在 `res/layout` 这个唯一目录下，随着产品的不断迭代，该目录下的文件急剧增加，命名、查找、维护的难度变得越来越大  
- **关系不直观**：一个页面（Activity）中可能包含几个甚至十几个布局文件，没有办法直观精确地查看和定位一个页面涉及到的布局文件，在源码和这些布局文件之间来回切换阅读和修改是异常艰难的，而对于 `res/layout` 目录下的布局文件也没有办法直观地了解哪些地方正在使用这些布局文件  
- **繁琐**：在布局文件中我们给每个控件起一个名字（ID），而在程序中我们需要根据这个 ID 查找我们需要的具体控件，如 `TextView nameTextView = (TextView) rootView.findViewById(R.id.tv_name);`，页面中的每个控件都需要这样的操作获取，还要重新再起一个符合源码规范的新名字，对于几十甚至几百个控件的页面来说，完成这样的操作是相当繁琐相当痛苦的  
- **耦合严重**：随着应用迭代，XML 布局文件和源码都要同步地进行更改，也就是说每加一个控件，XML 文件和源码都要相应地进行更改，如果 XML 中移除了某个控件而源码没有更改就会出现运行时异常，所以它们实际上是强耦合的  
- **方式不统一**：一个页面布局既可以通过 XML 文件的方式静态指定，也可以通过编写源码的方式动态创建，这两种截然不同的方式虽然都可以实现页面布局，但毕竟是不同的语言，不同的系统，很难统一管理和维护  
- **架构缺陷**：数据驱动视图思想的实现需要视图可以方便的与数据进行单向或者双向的绑定，只有 Data Binding 技术可以实现 XML 里插入代码，完成和数据的绑定，但是这样的操作就像 JSP 在 HTML 文件中插入 Java 代码一样，虽然简单直接，但是代码逻辑的连贯性、一致性以及可维护性会面临前所未有的挑战  

那什么样的 UI 构建方式才能避免上述的问题呢？什么样的构建方式才是简单有效的呢？我相信大多数人的回答都是 “声明式（declarative）”，在源码中声明式地构建 UI 既直观又不会损失源码的能力。但是这个愿景在实现上却又困难重重，怎么让 Java 或 Kotlin 拥有声明式语法的能力，怎么让排版布局更加的简洁直观，怎么避免 UI 逻辑和业务逻辑的耦合等等都是需要重点解决的问题，而我觉得 [Jetpack Compose](https://developer.android.com/jetpack/compose) 是个很不错的尝试  

## 框架思想

### 关注点分离

关注点分离（separation of concerns）是最常见最出名的软件设计原则，也是每个开发者都应该了解并遵循的，其实关注点分离最初是对另外两个词的概括：耦合（coupling）和内聚（cohesion）。理论上，当我们写代码时，我们会把应用看成多个模块，而且还可能把每个模块看成多个单元，这些模块或单元之间的依赖关系就是耦合，也就是说，如果我在某处对一些代码进行了更改，那么我还必须对其他文件进行多少更改？所以我们一般的想法就是尽可能的减少耦合。有时耦合是隐式的，那些我们依赖的依赖或者其他我们依赖的东西实际上是不确定的，但是还是会因为我们的更改而被破坏。另一方面，内聚指的是模块中的单元如何相互归属，它们彼此相关，高内聚通常被视为一件好事。因此关注点分离就是将尽可能多的相关代码组织在一起，以便我们的代码可以随着时间推移而更好地维护，随着应用的成长而真正地扩展  
在 Android 中一般的做法是用 XML 布局显示东西，用 ViewModel 给这个布局提供数据，事实上这里隐含了很多依赖，ViewModel 和布局之间存在很多耦合，如果 XML 中新增了控件，ViewModel 中也要新增对应的数据，这个关系是隐式的，但又是真实存在的。如果我们用相同的语言如 Kotlin 构建 UI，那么这个关系就可能会变成显式的了，甚至我们接下来开始重构一些代码，将一些东西移到它们所属的地方，实际上减少了某些耦合，增加了一些内聚。你可能会问了，这不是把业务逻辑和 UI 混在一起了吗？好吧，我们换个角度看一下，一些业务逻辑难道不是 UI 的一部分吗？其实任何框架都不能完美地帮你分离你的关注点，也不能阻止你将逻辑和 UI 混在一起，但是 Jetpack Compose 提供了工具可以让你很容易进行分离，这个工具就是组合式函数（composable functions），一个加了 `@Composable` 注解的函数，所以你之前写函数时重构，写可靠、可维护性、整洁代码的技巧同样适用于组合式函数  

### 声明式 vs 命令式

声明式编程（declarative）和命令式编程（imperative）是不同的编程思想，比如有个需求是这样的，未读消息数是 0 的时候显示一个空信封的图标，有几个消息的时候在信封图标上加个信件图标和消息数 badge，消息数超过 100 时再加个火苗并且 badge 不再是具体数字而是 99+。如果是命令式编程，我们肯定要写一个根据数量进行更新的函数：  

```kotlin
fun updateCount(count: Int) {
    if (count > 0 && !hasBadge()) {
        addBadge()
    } else if (count == 0 && hasBadge()) {
        removeBadge()
    }
    if (count > 99 && !hasFire()) {
        addFire()
        setBadgeText("99+")
    } else if (count <= 99 && hasFire()) {
        removeFire()
    }
    if (count > 0 && !hasPaper()) {
        addPaper()
    } else if (count == 0 && hasPaper()) {
        removePaper()
    }
    if (count <= 99) {
        setBadgeText("$count")
    }
}
```

我们弄清楚如何调整 UI 以使其呈现正确的状态，实际上可能还有很多极端情况，这个逻辑并不简单，但是这已经算是相对简单的例子了。而如果你用声明式的方式写这段逻辑那么会是这样的：  

```kotlin
@Composable
fun BadgeEnvelope(count: Int) {
    Envelope(fire = count > 99, paper = count > 0) {
        if (count > 0) {
            Badge(text = if (count > 99) "99+" else "$count")
        }
    }
}
```

你会发现至少在 UI 操作上来说声明式编程要更加直观，更加简洁  
而 UI 开发者最关心的是什么呢？对于给定的数据 UI 该怎么显示？怎么响应事件让 UI 进行交互？~~UI 随着时间应该怎样变化？~~，有了声明式编程，有了 Jetpack Compose，我们不再需要考虑 UI 随时间的变化，注意，这是最重要，最关键的点，因为在我们拿到数据后我们就定义了它在各个状态下应该怎么展示，之后框架会控制如何从一个状态进入另一个状态，即 “根据提供的参数来描述 UI”。组合式函数，是个函数定义，但是它在一个地方描述了 UI 所有可能的状态，而且是本地定义的，这就是组合（composition），因此有了 Compose 和 `@Composable` 这两个名字  

### 组合 vs 继承

组合（composition）和继承（Inheritance）是面向对象编程中最常见的关联关系，继承是扩展类功能最简单直接的方式，但是多继承弊端太大导致除了 C++ 的大部分语言都是只允许单继承的，如果我们把 View 系统通过继承实现，那么就会出现类似这样的问题，如果我想要个 Input，那么我继承 View，如果我想要个 ValidatedInput 那么我继承 Input，如果我想要个 DateInput 那么我继承 ValidatedInput，如果我想要个 DateRangeInput 怎么办呢？我不能继承 DateInput 因为我有两个 Date，但我又想拥有 DateInput 的能力，所以，我们最终还是遇到了单继承的限制。而在 Jetpack Compose 中这个问题就很简单了，我们无非多组合一个 DateInput 而已  

### 封装

Jetpack Compose 另一个做得比较好的地方就是封装，一个 composable 就是 **给定参数**，一个 composable 可以 **管理状态**，这是你开放你的 API 时唯一需要考虑的。另一方面，composable 可以管理和创建状态，然后它可以将状态以及接收到的数据作为参数传递给其他 composable，子 composable 也可以通过回调的方式通知你状态的更改  

### 重组

重组（Recomposition）最基本的就是任何组合式函数都有 **随时被再次调用** 的能力，这也就意味着，如果你有一个很大的层级结构，当一部分层级改变后，你不需要重建整个层级。你可以利用这个特性做一些大事，比如对于之前这样的操作：  

```kotlin
fun bind(liveMsgs: LiveData<MessageData>) {
    liveMsgs.observe(this) { msgs ->
        updateBody(msgs)
    }
}
```

我们观察这个 LiveData，每次 LiveData 更新的时候都会调用我们传入的 lambda，然后更新 UI。但是这毕竟是异步回调的形式，不符合我们的习惯，而在 Jetpack Compose 中我们就可以把这个关系转换过来：  

```kotlin
@Composable
fun Messages(liveMsgs: LiveData<MessageData>) {
    val msgs = +observe(liveMsgs)
    for (msg in msgs) {
        Message(msg)
    }
}
```

在这里我们调用了 `observe()` 函数，它做了两件事，首先是解封装 LiveData 来返回它的当前值，这也就意味着你可以在函数体中直接使用这个值。其次，它还隐式地将 LiveData 订阅到这个它会被解封装的组合式函数作用域中。这也就意味着，我们不再需要传递 lambda 表达式了，我们只需要知道这个组合式函数每次在 LiveData 变化时都会重组就行了。让我们再次比较上面两段代码，虽然在代码量上没有什么差异，但是在思想上后者要更加符合我们的思维习惯，更加直观  

### 数据驱动视图

数据驱动视图的思想既能简化 UI 操作又能保证数据展示的一致性，而 Data Binding 对于数据驱动视图的尝试虽然有效，但是并不优雅，一个 Model 可以插入到 XML 中，可以进行一些简单的处理，而如果让视图跟随 Model 变化还需要将 Model 转化成 Observable，这个转化是需要手动完成的。而 Jetpack Compose 对于数据驱动视图的尝试要更优雅一些，如这里的一个计数器功能：  

```kotlin
@Composable
fun Counter() {
    val count = +state { 0 }
    Button(
        text = "Count: ${count.value}",
        onClick = { count.value += 1 }
    )
}
```

`state()` 函数可以直接返回包裹了给定值的 State 状态类实例，State 类用了 `@Model` 注解，而 `@Model` 注解就意味着这个类的所有属性的读写操作都是 observable 的，Jetpack Compose 做得就是当你执行你的组合式函数时，如果你读取了一些 Model 实例，那么 Jetpack Compose 将自动订阅所在的作用域以便进行 Model 的读写。因此这个例子中的 Counter 是独立自给的，每次 Model 的值发生更改时 Counter 都会重组

## 使用

### 组合式函数

Jetpack Compose 是建立在组合式函数（composable functions）的基础上的，这些函数可以让你以编程的方式定义 UI（通过描述它的形状和数据依赖），而不是关注 UI 的构建过程  
一个组合式函数只能被另一个组合式函数调用，所以组合式函数需要添加 `@Composable` 注解  

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Greeting("Android")
        }
    }
}

@Composable
fun Greeting(name: String) {
    Text("Hello $name!")
}
```

没有参数的组合式函数是可以直接预览的，只需要添加 `@Preview` 注解即可  
![preview](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/jetpack_compose_preview_1.png)  
Jetpack Compose 团队认为 “创建未被应用调用的单独的预览函数是最佳的做法，专门的预览函数不但可以提高性能，还可以方便地提供多个预览”，不过我觉得有点鸡肋，单个组合式函数应该可以自动或者手动预览，不应该书写额外的预览函数，而对于多个预览函数，应该属于 UI 测试的范畴，不需要也不应该出现在源码中  
![multiple previews](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/jetpack_compose_preview_2.png)  

### 布局

使用 `Column()` 函数可以竖向堆叠元素，可以通过它的 `crossAxisSize` 参数指定列的大小，通过它的 `modifier` 参数指定修饰样式  

```kotlin
Column(
    crossAxisSize = LayoutSize.Expand,
    modifier = Spacing(16.dp)
) {
    Text("A day in Shark Fin Cove")
    Text("Davenport, California")
    Text("December 2018")
}
```

`Container()` 可以作为通用容器包裹和限制里面的元素。`HeightSpacer()` 可以用作留白。`Clip()` 可以裁剪，参数是用来裁剪的 `Shape`，`Shape` 是不可见的。`MaterialTheme()` 可以给组件应用主题，然后就可以给文本应用样式了，如 `Text("A day in Shark Fin Cove", style = +themeTextStyle { h6 })`  

```kotlin
@Composable
private fun TopicItem(topicKey: String, itemTitle: String) {
    val image = +imageResource(R.drawable.placeholder_1_1)
    Padding(left = 16.dp, right = 16.dp) {
        FlexRow(
            crossAxisAlignment = CrossAxisAlignment.Center
        ) {
            inflexible {
                Container(width = 56.dp, height = 56.dp) {
                    Clip(RoundedCornerShape(4.dp)) {
                        DrawImage(image)
                    }
                }
            }
            expanded(1f) {
                Text(
                    text = itemTitle,
                    modifier = Spacing(16.dp),
                    style = +themeTextStyle { subtitle1 })
            }
            inflexible {
                val selected = isTopicSelected(topicKey)
                SelectTopicButton(
                    onSelected = {
                        selectTopic(topicKey, !selected)
                    },
                    selected = selected
                )
            }
        }
    }
}
```

这是一个包含三个元素的列表项，看起来还算直观，但是还是感觉哪里有点别扭  
由于借助了 Kotlin 的 trailing lambda 表达式的语法，代码最终看起来还是能够表达出某种层次或结构的

## 思考

Jetpack Compose 是个很有趣的尝试，让我看到了 Android 新的构建 UI 方式的可能，从语法上来看还是有一些 HTML 和 Flutter 的影子的，对于页面复杂嵌套层级过深情况下的处理应该还有很长一段路要走，Flex 布局能否解决这个问题，Flex 布局是否真的适合 Android，Jetpack Compose 的性能如何保证，调试是否方便，Gap Buffer 的算法是否比 Diff 算法有优势等等都是需要面对，需要思考，需要时间去解决的问题  
总之，Jetpack Compose 目前只是一个尝试，还缺少足够的控件支持，还缺少足够的工具支持，还缺少足够的稳定性，不过我很乐意看到这种新的尝试出现，刀耕火种的时代总是要过去的  

## 参考

- [Jetpack Compose Basics](https://developer.android.com/jetpack/compose/tutorial)
- [Understanding Compose (Android Dev Summit '19)](https://www.youtube.com/watch?v=Q9MtlmmN4Q0)
