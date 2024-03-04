# HarmonyOS 学习笔记

## 基础知识

### 应用程序框架

* `UIAbility` 是一种包含用户界面的应用组件，主要用于和用户进行交互。是系统调度的单元，为应用提供窗口，以及在其中绘制页面
* 每一个 `UIAbility` 实例，都对应于一个最近任务列表中的任务
* 一个 `UIAbility` 可以对应于多个页面，建议将一个独立的功能模块放到一个 `UIAbility` 中
* 调用 `router.back()` 方法，不会新建页面，返回的是原来的页面，在原来页面中 `@State` 声明的变量不会重复声明，也不会触发页面的 `aboutToAppear()` 回调，因此无法直接在变量声明以及页面的 `aboutToAppear()` 中获取参数，可以在用到的地方或 `onPageShow()` 中获取
* `UIAbility` 的生命周期包括 Create、Foreground、Background、Destroy 四个状态，WindowStageCreate 和 WindowStageDestroy 为窗口管理器（`WindowStage`）在 `UIAbility` 中管理 UI 界面功能的两个生命周期回调，从而实现 UIAbility 与窗口之间的弱耦合
* `UIAbility` 实例创建完成之后，在进入 Foreground 之前，系统会创建一个 `WindowStage`。每一个 `UIAbility` 实例都对应持有一个 `WindowStage` 实例
* 可以在 `onWindowStageCreate()` 中加载页面，订阅获焦/失焦、可见/不可见事件
* 如果 `UIAbility` 的 `launchType` 是 `singleton`（默认类型） 并且实例已经存在时，会直接进入 `onNewWant()` 回调。如果是 `specified` 并且实例存在的话会先进入 `onAcceptWant()` 回调，然后才进入 `onNewWant()`
* `router.pushUrl()` mode 参数如果是 Single，如果目标 url 页面已存在，栈数不变，否则按 Standard 跳转，栈数加一
* `router.replaceUrl()` mode 参数如果是 Single，如果目标 url 页面已存在，当前页面会被替换，栈数减一，否则按 Standard 跳转，栈数不变
* `app.json5` 主要用来配置应用全局配置（包名、版本号、应用名称和图标等）和特定设备类型的配置信息
* `module.json5` 主要用来配置 module 信息（名称、类型、支持的设备）、应用组件信息（UIAbility 和 ExtensionAbility）、应用权限信息
* HAP 是应用安装的基本单位，一个 App Pack 可以包含多个 .hap 文件
* HAR 跟随使用方编译，多个使用方会有多个拷贝。HSP 独立编译，运行时一个进程只有一份

### UI 基础

* 自定义组件的生命周期函数：`aboutToAppear()` 在创建实例后 `build()` 前被调用，可以改变状态变量，如果是 `@Entry` 类型的组件，每次显示（路由/回到前台）都会触发 `onPageShow()`。不要在 `aboutToDisappear()` 中更改状态变量，尤其是 `@Link` 的，也不要使用 `async/await`
* 退出应用时会依次调用 onPageHide --> Parent aboutToDisappear --> Child aboutToDisappear
* Web 组件默认允许缩放（`zoomAccess()`），文本缩放比例使用 `.textZoomAtio()` 配置。默认允许 JS 执行（`javaScriptAccess()`），`onConfirm()` 返回 true 可以自定义弹窗，原生调 JS 可以使用 `controller.runJavaScript()`，JS 调原生可以直接在初始化时调用 `javaScriptProxy()` 注册，或者先 `controller.registerJavaScriptProxy()` 注入对象，然后调用 `controller.refresh()` 才能生效。使用 `controller.createWebMessagePorts()` 创建通信端口，再使用 `controller.postMessage()` 把一个端口告诉前端页面就可以实现双向通信了。使用 `deleteJavaScriptRegister()` 可以删除注册。使用 `onConsole()` 可以获取控制台日志
* 首选项（ohos.data.preferences）存键值对时，key 字符长度不超过 80，value 不超过 8192，不建议超过 1 万条
* 配置组件的事件方法，如果使用匿名函数表达式或组件成员函数，必须 bind(this)，以确保函数体中的 this 指向当前组件
* 自定义构建函数 `@Builder` 的参数默认是值传递，如果想要引用传递，使用 `($$: { param1: string })` `$$.param1` 这种范式

