# 基础知识
## 底层原理
* Android 操作系统是一个多用户 Linux 操作系统，每个应用都是一个用户
* 操作系统一般会给每个应用分配一个唯一的 Linux 用户 ID，这个 ID 对应用是不可见的。但有些情况下两个应用可以共享同一个 Linux 用户 ID，此时他们可以访问彼此的文件，甚至还可以运行在同一个 Linux 进程中，共享同一个虚拟机。但两个应用的签名必须是一样的
* 每个进程都有自己的虚拟机，一般每个应用都运行在自己的 Linux 进程中
## 应用组件
* 应用没有唯一的入口，没有 `main()` 函数，因为应用是由多个组件拼凑在一起的，每个组件都是系统或者用户进入应用的入口，组件之间既可以是相互独立的，也可以是相互依赖的。系统和其它应用在被允许的情况下可以启动/激活一个应用的任意一个组件
* 组件有四种类型: `Activity`，`Service`，`BroadcastReceiver` 和 `ContentProvider`
### Activity
* `Activity` 表示一个新的用户界面，只能由系统进行创建和销毁，应用只能监听到一些生命周期回调，这些回调通常也被叫作生命周期方法
* `Activity` 的名字一旦确定好就不要再更改了，否则可能会引发一系列问题
### Service
* `Service` 表示一个后台服务，`Service` 可以是独立的，可以在应用退出后继续运行。也可以绑定到其他进程或 `Activity`，表示其他进程想使用这个 `Service`，像输入法、动态壁纸、屏保等系统功能都是以 `Service` 的形式存在的，在需要运行的时候进行绑定
* 大部分情况下，更建议使用 `JobScheduler`，因为 `JobScheduler` 和 `Doze` API 配合下一般会比简单使用 `Service` 更省电
### BroadcastReceiver
* `BroadcastReceiver` 是一个事件传递的组件，通过它应用可以响应系统范围的广播通知。系统的包管理器会在安装应用时将应用中的静态广播接收器注册好，所以即使应用没在运行，系统也能把事件传递到该组件。
* 通过 `BroadcastReceiver` 可以实现进程间通信
### ContentProvider
* `ContentProvider` 是在多个应用间共享数据的组件，如果应用的一些数据想要被其它应用使用，必须通过 `ContentPrivider` 进行管理，不过应用的私有数据也可以通过 `ContentProvider` 进行管理，主要还是因为 `ContentProvider` 提供了共享数据的抽象，使用者不需要知道数据究竟是以文件形式还是数据库等其他形式存储的，只需要通过 `ContentProvider` 提供的 **统一的 API** 进行数据的增删改查即可。同时 `ContentProvider` 还提供了 **安全** 环境，可以根据需要方便地控制数据的访问权限，不需要手动控制文件权限或数据库权限
* 为了安全，也为了方便，一般需要通过 `ContentResolver` 操作 `ContentProvider`
* 通过 `ContentProvider` 可以实现进程间通信
## 激活组件
* 应用不能也不应该直接激活其它应用的任意一个组件，但是系统可以，所以要想激活一个组件，需要给系统发一个消息详细说明你的意图（ Intent ），之后系统就会为你激活这个组件
* `Activity`，`Service`，`BroadcastReceiver` 都需要通过被称为 `Intent` 的异步消息激活
* 被激活组件返回的结果也是 `Intent` 形式的
* `ContentProvider` 只有在收到 `ContentResolver` 的请求时才会被激活
* 只有 `BroadcastReceiver` 可以不在 manifest 文件中注册，因为有些 `BroadcastReceiver` 需要在程序运行时动态地注册和注销。而其它组件必须在 manifest 文件中注册，否则无法被系统记录，也就无法被激活
* 如果 `Intent` 通过组件类名显式指明了唯一的目标组件，那么这个 `Intent` 就是显式的，否则就是隐式的。隐式 `Intent` 一般只描述要执行动作的类型，必要时可以携带数据，系统会根据这个隐式 `Intent` 的描述决定激活哪个组件，如果有多个组件符合激活条件，系统一般会弹出选择框让用户选择到底激活哪个组件
* `Service` 必须使用显式 `Intent` 激活，不能声明 `IntentFilter`
* 启动指定的 `Activity` 使用显式 `Intent`，启动随便一个能完成指定工作的 `Activity` 使用隐式 `Intent`。能完成指定工作的那些想要被隐式 `Intent` 激活的 `Activity` 需要事先声明好 `IntentFilter` 表示自己有能力处理什么工作，`IntentFilter` 一般通过 能完成的动作 、意图类型 和 额外数据 来描述
* 要想被隐式 `Intent` 激活，意图类型至少要包含 `android.intent.category.DEFAULT` 的意图类型
* 在使用隐式 `Intent` 激活 `Activity` 之前一定要检查一下有没有 `Activity` 能处理这个 `Intent` :
```java
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(sendIntent);
}
```
```java
PackageManager packageManager = getPackageManager();
List<ResolveInfo> activities = packageManager.queryIntentActivities(intent,
        PackageManager.MATCH_DEFAULT_ONLY);
boolean isIntentSafe = activities.size() > 0;
```
* 使用隐式 `Intent` 时每次都强制用户选择一个组件激活:
```java
Intent intent = new Intent(Intent.ACTION_SEND);
String title = getResources().getString(R.string.chooser_title);
Intent chooser = Intent.createChooser(intent, title);
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
```
* 如果想要你的 `Activity` 能被隐式 `Intent` 激活，如果想要某个 **链接** 能直接跳转到你的 `Activity`，必须配置好 `IntentFilter`。这种链接分为两种: **Deep links** 和 **Android App Links**
* **Deep links** 对链接的 scheme 没有要求，对系统版本也没有要求，也不会验证链接的安全性，不过需要一个 `android.intent.action.VIEW` 的 action 以便 Google Search 能直接打开，需要 `android.intent.category.DEFAULT` 的 category 才能响应隐式 Intent，需要 `android.intent.category.BROWSABLE` 的 category 浏览器打开链接时才能跳转到应用，所以经典用例如下。一个 intent filter 最好只声明一个 data 描述，否则你得考虑和测试所有变体的情况。系统处理这个链接的流程为: 如果用户之前指定了打开这个链接的默认应用就直接打开这个应用 → 如果只有一个应用可以处理这个链接就直接打开这个应用 → 弹窗让用户选择用哪个应用打开
```xml
<activity
    android:name="com.example.android.GizmosActivity"
    android:label="@string/title_gizmos" >
    <intent-filter android:label="@string/filter_view_http_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "http://www.example.com/gizmos” -->
        <data android:scheme="http"
              android:host="www.example.com"
              android:pathPrefix="/gizmos" />
        <!-- note that the leading "/" is required for pathPrefix-->
    </intent-filter>
    <intent-filter android:label="@string/filter_view_example_gizmos">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <!-- Accepts URIs that begin with "example://gizmos” -->
        <data android:scheme="example"
              android:host="gizmos" />
    </intent-filter>
</activity>
```
* **Android App Links** 是一种特殊的 Deep links，要求链接必须是你自己网站的 HTTP URL 链接，系统版本至少是 Android 6.0 (API level 23)，优点是安全且具体，其他应用不能使用你的链接，不过你得先 **验证你的链接**，由于链接和网站链接一致所以可以无缝地在应用和网站间切换，可以支持 Instant App，可以通过浏览器、谷歌搜索 APP、系统屏幕搜索、甚至 Google Assistant 的链接直接跳转到应用。验证链接的流程为: 将 `<intent-filter>` 标签的 `android:autoVerify` 设置为 `true` 以告诉系统自动验证你的应用属于这个 HTTP URL 域名 → 填写好网站域名和应用 ID 并使用签名文件生成 Digital Asset Links JSON 文件 → 将文件上传到服务器，访问路径为 `https://domain.name/.well-known/assetlinks.json` ，响应格式为 `application/json`，子域名也需要存在对应的文件，一个域名可以关联多个应用，一个应用也可以关联多个域名，且可以使用相同的签名 → 利用编辑器插件完成关联并验证
* 使用 Intent Scheme URL 需要做过滤。如果浏览器支持 Intent Scheme Uri 语法，如果过滤不当，那么恶意用户可能通过浏览器 js 代码进行一些恶意行为，比如盗取 cookie 等。所以如果使用了 `Intent#parseUri()` 方法，获取的 intent 必须严格过滤，intent 至少包含 `addCategory(“android.intent.category.BROWSABLE”)`，`setComponent(null)`，`setSelector(null)` 3 个策略
* 开放的 Activity/Service/BroadcastReceiver 等需要对传入的 intent 做合法性校验  
## 应用资源
* 添加资源限定符的顺序为: SIM 卡所属的国家代码和移动网代码 → **语言区域代码** → 布局方向 → 最小宽度 → 可用宽度 → 可用高度 → 屏幕大不大 → 屏幕长不长 → 屏幕圆不圆 → 屏幕色域宽不宽 → 屏幕支持的动态范围高不高 → **屏幕方向** → 设备的 UI 模式 → 夜间模式 → **屏幕像素密度** → 触摸屏类型 → 键盘类型 → 主要的文字输入方式  → 导航键是否可用 → 主要的非触摸导航方式 → **支持的 API level**
* 一个资源目录的每种资源限定符最多只能出现一次
* 必须提供缺省的资源文件
* 资源目录名是大小写不敏感的
* drawable 资源取别名:
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <drawable name="icon">@drawable/icon_ca</drawable>
</resources>
```
* 布局文件取别名:
```xml
<?xml version="1.0" encoding="utf-8"?>
<merge>
    <include layout="@layout/main_ltr"/>
</merge>
```
* 只有动画、菜单、raw 资源 以及 xml/ 目录中的资源不能使用别名
* 寻找使用最优资源的流程:
![res](https://user-gold-cdn.xitu.io/2019/1/29/16898e52fcd83292?w=442&h=520&f=png&s=164181)
* 在应用程序运行时，设备的配置可能会发生变化（如屏幕方向变化、切换到多窗口模式，切换了系统语言），默认情况下系统会销毁重建正在运行的 `Activity` ，所以应用程序必须保证销毁重建的过程中用户的数据和页面状态完好无损地恢复。如果不想系统销毁重建你的 `Activity` 只需要在 manifest 文件的 `<activity>` 标签的 `android:configChanges` 属性中添加你想自己处理的配置更改，多个配置使用 "`|`" 隔开，此时系统就不会在这些配置更改后销毁重建你的这个 `Activity` 而是直接调用它的 `onConfigurationChanged()` 回调方法，你需要在这个回调中自己处理配置更改后的行为。
* `Activity` 的销毁重建不但发生在设备配置更改后，只要用户离开了某个 `Activity`，那么那个 `Activity` 就随时可能被系统销毁。所以销毁重建是无法避免的，也不应该逃避，而是应该想办法保存和恢复状态
* 由于各种各样的硬件都能安装 Android 操作系统，Android 操作系统之间也可能千差万别，而应用程序的一些功能是与这些软硬件息息相关的，如拍照应用需要设备必须有摄像头才能正常工作。应用可以通过 `<uses-feature>` 标签声明只有满足这些软硬件要求的设备才能安装，通过它的 `android:required` 属性设置该要求是不是必须的，程序中可以通过 `PackageManager.hasSystemFeature()` 方法判断
# 核心知识
## Activity 相关
### 生命周期方法
* 当 `Activity` 变得对用户可见时，将会回调 `onStart()`, 当 `Activity` 变得可以和用户交互时，将会回调 `onResume()`
* `onPause()` 被调用时 `Activity` 可能依然对用户全部可见，如多窗口模式下没有获得焦点时，所以在 `onResume()` 中申请资源在 `onPause()` 中释放资源的想法并不总是合理的
* `onStop()` 被调用时表示 `Activity` 已经完全不可见了，此时应该尽量停止包含动画在内的 UI 更新，尽量释放暂时不用的资源。对于 stopped 的 `Activity`，系统随时可能杀掉包含这个 `Activity` 的进程，如果没有合适的机会可以在 `onStop()` 中保存一些数据
* 如果系统在未经用户允许的情况下销毁了 `Activity`（杀掉了该 `Activity` 实例所在的进程），那么系统肯定记得这个实例存在过，在用户重新回到这个 `Activity` 时会重新创建一个新的实例，并将之前保存好的实例状态传递给这个新的实例。这个系统之前保存好的用来恢复 `Activity` 状态的数据被称为**实例状态**（Instance state），实例状态是以键值对的形式存储在 Bundle 对象中的，默认系统只能自动存储和恢复有 ID 的 View 的简单状态（如输入框的文本，滚动控件的滚动位置），但由于在主线程中序列化或反序列化 `Bundle` 对象既消耗时间又消耗系统进程内存，所以最好只用它保存简单、轻量的数据
* `onSaveInstanceState()` 被调用的时机: 对于 `Build.VERSION_CODES.P` 及之后的系统该方法会在 `onStop()` 之后随时可能被调用，对于之前的系统该方法会在 `onStop()` 之前随时被调用
* `onRestoreInstanceState()` 被调用的时机: 如果有实例状态要恢复那么一定会在 `onStart()` 之后被调用
* `onActivityResult()` 被调用时机: `onResume()` 之前。目标 `Activity` 没有显式返回任何结果或者崩溃那么 resultCode 就会是 `RESULT_CANCELED`
* 在保存实例状态之后恢复实例状态之前的一些操作（如 Fragment 的事务提交）是不允许的，Android 系统会不惜一切代价避免状态丢失。`Activity#onCreate()` 方法中提交事务是没问题的，因为你可以在里面根据保存的状态重建，但是在其他生命周期回调中提交事务就可能会出现问题了。`FragmentActivity#onPostResume()` 方法中调用了 `FragmentActivity#onResumeFragments()` 方法完成其关联的所有的 Fragment 的 resume 事件的分发，执行完这两个方法 Activity 和它关联的所有 Fragment 才算真正的 resumed，才算恢复了状态，才可以提交事务，所以如果非要在 `Activity#onCreate()` 之外的回调中提交事务那么 `FragmentActivity#onPostResume()` 和 `FragmentActivity#onResumeFragments()` 是最好的选择。避免在异步的回调中提交事务: 因为在这些回调执行的时候很难确定当前 Activity 正处于什么生命周期状态，而且突然地提交事务更改大量 UI 会产生糟糕的用户体验，所以如果遇到这样的场景可以考虑换一种实现思路，不要随便使用 `commitAllowingStateLoss()` 方法  
* 如非必须，避免使用多层嵌套的 Fragment，否则容易出现 Bug
### 任务和返回栈
* `Activity` 可以在 manifest 文件中定义自己应该如何与当前任务相关联，`Activity` 也可以在启动其它 `Activity` 时通过 `Intent` 的 flag 要求其它 `Activity` 应该如何与当前任务相关联，如果两者同时出现，那么 `Intent` 的 flag 要求获胜
* `launchMode` 属性默认是 `standard`，每次启动这样的 `Activity` 都会新建一个新的实例放入启动它的任务中。**一个新的 Intent 总会创建一个新的实例。一个任务可以有多个该 Activity 的实例，每个该 Activity 的实例可以属于不同的任务**
* `launchMode` 属性是 `singleTop` 的 `Activity` : 如果当前任务顶部已经是这个 `Activity` 的实例那么就直接将 `Intent` 传递给这个实例的 `onNewIntent()` 方法。**一个任务可以有多个该 Activity 的实例，每个该 Activity 的实例可以属于不同的任务**
* `launchMode` 属性是 `singleTask` 的 `Activity` : 如果这个 `Activity` 的实例已经在某个任务中存在了那么就直接将 `Intent` 传递给这个实例的 `onNewIntent()` 方法，并将其所在的任务移到前台即当前任务顶部，否则会新建一个任务并实例化一个这个 `Activity` 的实例放在栈底
* `launchMode` 属性是 `singleInstance` 的 `Activity` : 和 `singleTask` 类似，不过它会保证新的任务中有且仅有一个这个 `Activity` 的实例
* `FLAG_ACTIVITY_NEW_TASK` : 行为和 `singleTask` 一样，不过在新建任务之前会先寻找是否已经存在和这个 `Activity` 有相同 affinity 的任务，如果已经存在就不新建任务了，而是直接在那个任务中启动
* `FLAG_ACTIVITY_SINGLE_TOP` : 行为和 `singleTop` 一样
* `FLAG_ACTIVITY_CLEAR_TOP` : 如果当前任务中已经有要启动的 `Activity` 的实例了，那么就销毁它上面所有的 `Activity`(**甚至包括它自己**)，由于 `launchMode` 属性是 `standard` 的 `Activity` 一个新的 Intent 总会创建一个新的实例，所以如果要启动的 `Activity` 的 `launchMode` 属性是 `standard` 的并且没有 `FLAG_ACTIVITY_SINGLE_TOP` 的 flag，那么这个 flag 会**销毁它自己**然后创建一个**新的实例**
* `FLAG_ACTIVITY_CLEAR_TOP` 和 `FLAG_ACTIVITY_NEW_TASK` 结合使用可以直接定位指定的 `Activity` 到前台
* 不管要启动的 `Activity` 是在当前任务中启动还是在新任务中启动，点击返回键都可以直接或间接回到之前的 `Activity`，间接的情况像 `singleTask` 是将整个任务而不是只有一个 `Activity` 移到前台，任务中的所有的 `Activity` 在点击返回键的时候都要依次弹出
* 如果离开了任务，系统可能会清除任务中除了最底层 `Activity` 外的的所有 `Activity`。将最底层 `Activity` 的 `<activity>` 标签的 `alwaysRetainTaskState` 属性设置为 `true` 可以保留任务中所有的 `Activity`。将最底层 `Activity` 的 `<activity>` 标签的 `clearTaskOnLaunch` 属性设置为 `true` 可以在无论何时进入或离开这个任务都清除任务中除了最底层 `Activity` 外的的所有 `Activity`。包含最底层 `Activity` 在内的任何 `Activity` 只要 `finishOnTaskLaunch` 属性设置为 `true` 那么离开任务再回来都不会出现了
* 将 `Activity` 作为新文档添加到最近任务中需要设置 `newDocumentIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_DOCUMENT);` 且 `launchMode` 必须是 `standard` 的，如果此时又设置了 `newDocumentIntent.addFlags(Intent.FLAG_ACTIVITY_MULTIPLE_TASK)` 那么系统每次都会创建新的任务并将目标 `Activity` 作为根 `Activity`，如果没有设置 `FLAG_ACTIVITY_MULTIPLE_TASK`，那么 `Activity` 实例会被重用到新的任务中（如果已经存在这样的任务就不会重建，而是直接将任务移到前台并调用 `onNewIntent()`）
* `<activity>` 标签的 `android:documentLaunchMode` 属性默认是 `none` : 不会为新文档创建新的任务。`intoExisting` 与设置了 `FLAG_ACTIVITY_NEW_DOCUMENT` 但没设置 `FLAG_ACTIVITY_MULTIPLE_TASK` 一样。`always` 与设置了 `FLAG_ACTIVITY_NEW_DOCUMENT` 同时设置了 `FLAG_ACTIVITY_MULTIPLE_TASK` 一样。`never` 和 `none` 一样不过会覆盖 `FLAG_ACTIVITY_NEW_DOCUMENT` 和 `FLAG_ACTIVITY_MULTIPLE_TASK`
* 使用 `Intent.FLAG_ACTIVITY_NEW_DOCUMENT|android.content.Intent.FLAG_ACTIVITY_RETAIN_IN_RECENTS;` 同时 `<activity>` 标签的 `android:autoRemoveFromRecents` 属性设置为 `false` 可以让文档 `Activity` 即使结束了也可以保留在最近任务中
* 使用 `finishAndRemoveTask()` 方法可以移除当前任务
### 动态申请权限
* Android 6.0 (API level 23) 开始 targetSdkVersion >= 23 的应用必须在运行时动态申请权限
* 权限请求对话框是操作系统进行管理的，应用无法也不应该干预。
* 系统对话框描述的是权限组而不是某个具体权限
* 如果用户授予了权限组中的一个权限，那么再申请该权限组的其它权限时系统会自动授予，不需要用户再授权。但这并不意味着该权限组中的其它权限就不用申请了，因为权限处于哪个权限组将来有可能会发生变化
* 调用 `requestPermissions()` 并不意味着系统一定会弹出权限请求对话框，也就是说不能假设调用该方法后就发生了用户交互，因为如果用户之前勾选了 “禁止后不再询问” 或者系统策略禁止应用获取权限，那么系统会直接拒绝此次权限请求，没有任何交互
* 如果某个权限跟应用的主要功能无关，如应用中广告可能需要位置权限，用户可能很费解，此时在申请权限之前弹出对话框向用户解释为什么需要这个权限是个不错的选择。但不要在所有申请权限之前都弹出对话框解释，因为频繁地打断用户的操作或让用户进行选择容易让用户不耐烦
* `Fragment` 中的 `onRequestPermissionsResult()` 方法只有在使用 `Fragment#requestPermissions()` 方法申请权限时才可能接收到回调，建议将权限放在所属 `Activity` 中申请和处理
* 应用应该尽量少地申请权限，像让用户拍一张照片或者选择一张图片完全不需要相机权限和外存权限，可以通过隐式 Intent 拉起系统相机或其他应用完成，应用只需要在 `onActivityResult()` 回调中接收数据就行了。但是有一点一定要注意，如果你在 `AndroidManifest.xml` 文件中声明了相机权限，你就必须得动态申请并获得相机权限才能拉起系统相机  

```java
// 请求通讯录权限的模板代码如下
private void showContactsWithPermissionsCheck() {
    if (ContextCompat.checkSelfPermission(MainActivity.this,
            Manifest.permission.READ_CONTACTS)
            != PackageManager.PERMISSION_GRANTED) {
        if (ActivityCompat.shouldShowRequestPermissionRationale(MainActivity.this,
                Manifest.permission.READ_CONTACTS)) {
            // TODO: 弹框解释为什么需要这个权限. 【下一步】 -> 再次请求权限
        } else {
            ActivityCompat.requestPermissions(MainActivity.this,
                    new String[]{Manifest.permission.READ_CONTACTS},
                    RC_CONTACTS);
        }
    } else {
        showContacts();
    }
}
private void showContacts() {
    startActivity(ContactsActivity.getIntent(MainActivity.this));
}
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions,
                                       @NonNull int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    switch (requestCode) {
        case RC_CONTACTS:
            if (grantResults.length > 0
                    && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                showContacts();
            } else {
                if (!ActivityCompat.shouldShowRequestPermissionRationale(MainActivity.this,
                        Manifest.permission.READ_CONTACTS)) {
                    // TODO: 弹框引导用户去设置页主动授予该权限. 【去设置】 -> 应用信息页
                } else {
                    // TODO: 弹框解释为什么需要这个权限. 【下一步】 -> 再次请求权限
                }
            }
            break;
        default:
            break;
    }
}
@Override
protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode == RC_SETTINGS) {
        // TODO: 在用户主动授予权限后重新检查权限，但不要在这里进行事务提交等生命周期敏感操作
    }
}
```
### Shortcut
* 类似于 iOS 的 3D Touch，长按启动图标弹出几个快捷入口，入口最好不要超过 4 个，像搜索、扫描二维码、发帖等应用程序最常用功能的入口被称为静态 shortcut，不会随着用户不同或随着用户使用而改变。还有一种像从某个存档点继续游戏、任务进度等与用户相关的上下文敏感入口被称为动态 shortcut，会因用户不同或随着用户使用不断变化。还有一种在 Android 8.0 (API level 26) 及以上系统版本上像固定网页标签等用户主动固定到桌面的快捷方式被称为固定 shortcut
* 静态 shortcut 系统可以自动备份和恢复，动态 shortcut 需要应用自己备份和恢复，固定 shortcut 的图标系统无法备份和恢复因此需要应用自己完成
* `android:shortcutId` 和 `android:shortcutShortLabel` 属性是必须的，`android:shortcutShortLabel` 不能超过 10 个字符，`android:shortcutLongLabel` 不能超过 25 个字符，`android:icon` 不能包含 tint
* 获取 `ShortcutManager` 的方式有两个: `getSystemService(ShortcutManager.class)` 和 `getSystemService(Context.SHORTCUT_SERVICE)`
* 创建固定 shortcut:
```java
ShortcutManager mShortcutManager =
        context.getSystemService(ShortcutManager.class);
if (mShortcutManager.isRequestPinShortcutSupported()) {
    ShortcutInfo pinShortcutInfo =
            new ShortcutInfo.Builder(context, "my-shortcut").build();
    Intent pinnedShortcutCallbackIntent =
            mShortcutManager.createShortcutResultIntent(pinShortcutInfo);
    PendingIntent successCallback = PendingIntent.getBroadcast(context, 0,
            pinnedShortcutCallbackIntent, 0);
    mShortcutManager.requestPinShortcut(pinShortcutInfo,
            successCallback.getIntentSender());
}
```
### 其它
* `Parcelable` 对象用来在进程间、`Activity` 间传递数据，保存实例状态也是用它，`Bundle` 是它的一个实现，最好只用它存储和传递少量数据，别超过 50k，否则既可能影响性能又可能导致崩溃
* Android 9 (API level 28) 开始废弃了 `Loader` API，包括 `LoaderManager` 和 `CursorLoader` 等类的使用。推荐使用 `ViewModel` 和 `LiveData` 在 `Activity` 或 `Fragment` 生命周期中加载数据
* `Activity` 可以通过 `getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)` 保持屏幕常亮，这是最推荐、最简单、最安全的保持屏幕常亮的方法，给 view 添加 `android:keepScreenOn="true"` 也是一样的。这个只在这个 `Activity` 生命周期内有效，所以大可放心，如果想提前解除常亮，只需要清除这个 flag 即可
* `WAKE_LOCK` 可以阻止系统睡眠，保持 CPU 一直运行，需要 `android.permission.WAKE_LOCK` 权限，通过 `powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "MyApp::MyWakelockTag")` 创建实例，通过 `wakeLock.acquire()` 方法请求锁，通过 `wakelock.release()` 释放锁
* `WakefulBroadcastReceiver` 结合 `IntentService` 也可以阻止系统睡眠
## UI 相关
### 系统栏适配
* Android 4.1 (API level 16) 开始可以通过 `setSystemUiVisibility()` 方法在各个 view 层次中（一般是在 DecorView 中）配置 UI flag 实现系统栏（状态栏、导航栏统称）配置，最终汇总体现到 window 级
* `View.SYSTEM_UI_FLAG_FULLSCREEN` 可以隐藏状态栏，`View.SYSTEM_UI_FLAG_HIDE_NAVIGATION` 可以隐藏导航栏。**但是:** 用户的任何交互包括触摸屏幕都会导致 flag 被清除进而系统栏**保持可见**，一旦离开当前 `Activity` flag 就会被清除，所以如果在 `onCreate()` 方法中设置了这个 flag 那么按 HOME 键再回来状态栏又保持可见了，非要这样设置的话一般要放在 `onResume()`  或 `onWindowFocusChanged()` 方法中，而且这样设置只有在目标 View 可见时才会生效，状态栏/导航栏的显示隐藏会导致显示内容的大小尺寸跟着变化。
* `View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_LAYOUT_STABLE` 可以让内容显示在状态栏后面，`View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_STABLE` 可以让内容显示在导航栏后面，这样无论系统栏显示还是隐藏内容都不会跟着变化，但不要让可交互的内容出现在系统栏区域内，通过将 `android:fitsSystemWindows` 属性设置为 `true` 可以让父容器调整 padding 以便为系统栏留出空间，如果想自定义这个 padding 可以通过覆写 View 的 `fitSystemWindows(Rect insets)` 方法（API level 20 以上覆写 `onApplyWindowInsets(WindowInsets insets)` 方法）完成  
* lean back 全屏模式: `View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION`，隐藏状态栏和导航栏，任何交互都会清除 flag 使系统栏**保持可见**
* Immersive 全屏模式: `View.SYSTEM_UI_FLAG_IMMERSIVE | View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION`，隐藏状态栏和导航栏，从被隐藏的系统栏边缘向内滑动会使系统栏**保持可见**，应用无法响应这个手势
* sticky immersive 全屏模式: `View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY | View.SYSTEM_UI_FLAG_FULLSCREEN | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION`，隐藏状态栏和导航栏，从被隐藏的系统栏边缘向内滑动会使系统栏**暂时可见**，flag 不会被清除，且系统栏的背景是半透明的，会覆盖应用的内容，应用也可以响应这个手势，在用户没有任何交互或者没有系统栏交互几秒钟后系统栏会**自动隐藏**
* 真正的沉浸式全屏体验需要 6 个 flag: `View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY | View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_FULLSCREEN`
* 监听系统栏可见性（sticky immersive 全屏模式无法监听）:
```java
decorView.setOnSystemUiVisibilityChangeListener(new View.OnSystemUiVisibilityChangeListener() {
    @Override
    public void onSystemUiVisibilityChange(int visibility) {
        if ((visibility & View.SYSTEM_UI_FLAG_FULLSCREEN) == 0) {
            // TODO: The system bars are visible. Make any desired
        } else {
            // TODO: The system bars are NOT visible. Make any desired
        }
    }
});
```
* 全面屏适配只需要指定支持的最大宽高比即可: `<meta-data android:name="android.max_aspect" android:value="2.4"/>`
* Android 9 (API level 28) 开始支持刘海屏 cutout 的配置，window 的属性 `layoutInDisplayCutoutMode` 默认是 `LAYOUT_IN_DISPLAY_CUTOUT_MODE_DEFAULT`，竖屏时可以渲染到刘海区，横屏时不允许渲染到刘海区。`LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES` 横竖屏都可以渲染到刘海区。`LAYOUT_IN_DISPLAY_CUTOUT_MODE_NEVER` 横竖屏都不允许渲染到刘海区，可以在 `values-v28/styles.xml` 文件中通过 `android:windowLayoutInDisplayCutoutMode` 指定默认的刘海区渲染模式
* 华为手机通过 `<meta-data android:name="android.notch_support" android:value="true" />` 属性声明应用是否已经适配了刘海屏，如果没适配，那么在横屏或者竖屏不显示状态栏时会禁止渲染到刘海区，开发者文档: [《华为刘海屏手机安卓O版本适配指导》](https://developer.huawei.com/consumer/cn/devservice/doc/50114)
* 小米手机通过 `<meta-data android:name="notch.config" android:value="portrait|landscape" />` 设置默认的刘海区渲染模式，开发者文档: [《小米刘海屏 Android O 适配》](https://dev.mi.com/console/doc/detail?pId=1293)，[《小米刘海屏 Android P 适配》](https://dev.mi.com/console/doc/detail?pId=1341)
* 其他手机的开发者文档有: OPPO 手机的 [《OPPO凹形屏适配说明》](https://open.oppomobile.com/wiki/doc#id=10159)，VIVO 手机的 [《异形屏应用适配指南》](https://dev.vivo.com.cn/documentCenter/doc/103)，锤子手机的 [《Smartisan 开发者文档》](https://resource.smartisan.com/resource/61263ed9599961d1191cc4381943b47a.pdf)
* Android 5.0 (API level 21) 开始支持通过 window 的 `setStatusBarColor()` 方法设置状态栏背景色，要求 window 必须添加 `WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS` 的 flag 并且清除 `WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS` 的 flag
* Android 6.0 (API level 23) 开始可以通过 `setSystemUiVisibility()` 方法设置 `View.SYSTEM_UI_FLAG_LIGHT_STATUS_BAR` flag 兼容亮色背景的状态栏，同样要求 window 必须添加 `WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS` 的 flag 并且清除 `WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS` 的 flag
* 小米手机在 MIUI 开发版 7.7.13 之前需要通过反射兼容亮色背景的状态栏，开发者文档: [《MIUI 9 & 10“状态栏黑色字符”实现方法变更通知》](https://dev.mi.com/console/doc/detail?pId=1159)
* 魅族手机同样需要通过反射兼容亮色背景的状态栏，开发者文档: [《状态栏变色》](http://open-wiki.flyme.cn/doc-wiki/index#id?79)<br />
### 动画
* **view 动画系统**只能作用于 view 对象，只能改变 view 的部分样式，只是简单改变了 view 绘制，并没有改变 view 真正的位置和属性。核心类是 `android.view.animation.Animation` 和它的 `ScaleAnimation` 等子类，一般使用 `AnimationUtils.loadAnimation()` 方法加载。不建议使用，除非为了方便又能满足现在和将来的需求
* **属性动画系统**是一个健壮的、优雅的动画系统，可以对任意对象的属性做动画。核心类是 `android.animation.Animator` 的子类 `ValueAnimator`、`ObjectAnimator`、`AnimatorSet`
* 通过调用 `ValueAnimator` 的 `ofInt()`、`ofFloat()` 等工厂方法获取 `ValueAnimator` 对象，通过它的 `addUpdateListener()` 方法可以监听动画值并在里面进行自定义操作
* `ObjectAnimator` 作为 `ValueAnimator` 的子类可以自动地为目标对象的命名属性设置动画，但是对目标对象有严格的要求: 目标对象必须有对应属性的 setter 方法，如果在工厂方法中只提供了一个动画值那么它会作为终止值，起始值为目标对象的当前值，此时为了获取当前属性值目标对象必须有对应属性的 getter 方法。有些属性的更改不会导致 view 重新渲染，此时需要主动调用 `invalidate()` 方法强制触发重绘
* `AnimatorListenerAdapter` 提供了 `Animator.AnimatorListener` 接口的空实现
* 多数情况下可以直接使用系统提供的几个动画 duration，如 `getResources().getInteger(android.R.integer.config_shortAnimTime)`
* 可以调用任意 view 对象的 `animate()` 方法获取 `ViewPropertyAnimator` 对象，链式调用这个对象的 `scaleX()`、`alpha()` 等方法可以简单方便地同时对 view 的多个属性做动画
* 为了更好地重用和管理属性动画，最好使用 XML 文件来描述动画并放到 `res/animator/` 目录下，`ValueAnimator` 对应 `<animator>` ，`ObjectAnimator` 对应 `<objectAnimator>`，`AnimatorSet` 对应 `<set>`，使用 `AnimatorInflater.loadAnimator()` 可以加载这些动画
* 动态 Drawable 的实现有两种，最传统最简单的就是像电影关键帧一样依次指定关键帧和每一帧的停留时间，`AnimationDrawable` 对应于 XML 文件中的 `<animation-list>`，保存目录为 `res/drawable/`，`AnimationDrawable` 的 `start()` 方法可以在 `onStart()` 中调用。还有一种是 `AnimatedVectorDrawable`，需要 `res/drawable/` 中的 `<animated-vector>` 引用 `res/drawable/` 中的 `<vector>` 对其使用 `res/animator/` 中的 `<objectAnimator>` 动画
* 突然更改显示的内容会让视觉感受非常突兀不和谐，而且可能意识不到哪些内容突然变了，所以很多场景下需要使用动画过渡一下，而不是突然更改显示的内容
* 显示隐藏 view 的常用动画有三个: crossfade 动画，card flip 动画，circular reveal 动画
* crossfade 动画就是内容淡出另一个内容淡入交叉进行，也被称为**溶入**动画。实现方式为: 事先将淡入 view 的 visibility 设置为 `GONE` → 开始动画时将淡入 view 的 alpha 设置为 0，visibility 设置为 `VISIBLE` → 将淡入 view 的 alpha 动画到 1，将淡出 view 的 alpha 动画到 0 并在动画结束时将淡出 view 的 visibility 设置为 `GONE`
* card flip 动画就是卡片翻转动画，需要四个动画描述: `card_flip_right_in`、`card_flip_right_out`、`card_flip_left_in`、`card_flip_left_out`
* Android 5.0 (API level 21) 开始支持 circular reveal 圆形裁剪动画，实现方式为: 事先将 view 的 visibility 设置为 `INVISIBLE` → 利用 `ViewAnimationUtils.createCircularReveal()` 方法创建半径从 0 到 `Math.hypot(cx, cy)` 的圆形裁剪动画 → 将 view 的 visibility 设置为 `VISIBLE` 然后开启动画
* **直线**动画移动 view 只需要借助 `ObjectAnimator.ofFloat()` 方法动画设置 view 的 `translationX` 或 `translationY` 属性即可
* **曲线**动画移动 view 还需要借助 Android 5.0 (API level 21) 开始提供的 `PathInterpolator` 插值器（对应于 XML 文件中的 `<pathInterpolator>`），他需要个 `Path` 对象描述运动的贝塞尔曲线。可以使用 `ObjectAnimator.ofFloat(view, "translationX", 100f)` 同时设置 `PathInterpolator` 也可以直接设置 view 动画路径 `ObjectAnimator.ofFloat(view, View.X, View.Y, path)`。系统提供的 `fast_out_linear_in.xml`、`fast_out_slow_in.xml`、`linear_out_slow_in.xml` 三个基础的曲线插值器可以直接使用
* 基于物理的动画需要引用 `support-dynamic-animation` 支持库，最常见的就是 `FlingAnimation` 和 `SpringAnimation` 动画，物理动画主要是模拟现实生活中的物理世界，利用经典物理学的知识和原理实现动画过程，其中最关键的就是**力**的概念。`FlingAnimation` 就是用户通过手势给动画元素一个力，动画元素在这个力的作用下运动，之后由于摩擦力的存在慢慢减速直到结束，当然这个力也可以通过程序直接指定（指定固定的初始速度）。`SpringAnimation` 就是弹簧动画，动画元素的运动与弹簧有关
* `FlingAnimation` 通过 `setStartVelocity()` 方法设置初始速度，通过 `setMinValue()` 和 `setMaxValue()` 约束动画值的范围，通过 `setFriction()` 设置摩擦力（如果不设置默认为 1）。如果动画的属性不是以像素为单位的，那么需要通过 `setMinimumVisibleChange()` 方法设置用户可察觉到动画值的最小更改，如对于 `TRANSLATION_X`，`TRANSLATION_Y`，`TRANSLATION_Z`，`SCROLL_X`，`SCROLL_Y` 1 像素的更改就对用户可见了，而对于 `ROTATION`，`ROTATION_X`，`ROTATION_Y` 最小可见更改是 `MIN_VISIBLE_CHANGE_ROTATION_DEGREES` 即 1/10 像素，对于 `ALPHA` 最小可见更改是 `MIN_VISIBLE_CHANGE_ALPHA` 即 1/256 像素，对于 `SCALE_X` 和 `SCALE_Y` 最小可见更改是 `MIN_VISIBLE_CHANGE_SCALE` 即 1/500 像素，计算公式为: 自定义属性值的范围 / 动画的变化像素范围。
* `SpringAnimation` 需要先巩固一下弹簧的知识，弹簧有一个属性叫**阻尼比 ζ**（damping ratio），是实际的粘性阻尼系数 C 与临界阻尼系数 Cr 的比。ζ = 1 时为临界阻尼，这是最小的能阻止系统震荡的情况，系统可以最快回到平衡位置。0 < ζ < 1 时为欠阻尼，物体会作对数衰减振动。ζ > 1 时为过阻尼，物体会没有振动地缓慢回到平衡位置。ζ = 0 表示不考虑阻尼，震动会一直持续下去不会停止。默认是 `SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY` 即 0.5，可以通过 `getSpring().setDampingRatio()` 设置。弹簧另一个属性叫**刚度**（stiffness），刚度越大形变产生的力就越大，默认是 `SpringForce.STIFFNESS_MEDIUM` 即 1500.0，可以通过 `getSpring().setStiffness()` 设置
![damping ratio](https://user-gold-cdn.xitu.io/2019/1/29/16898e76930fd4d4?w=272&h=174&f=png&s=15104)
* `FlingAnimation` 和 `SpringAnimation` 动画通过 `setStartVelocity()` 设置固定的初始速度时最好用 dp/s 转成 px/s : `TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dpPerSecond, getResources().getDisplayMetrics())`，用户手势的初始速度可以通过 `GestureDetector.OnGestureListener` 或 `VelocityTracker` 计算
* `SpringAnimation` 动画使用 `start()` 方法开始动画时属性值不会马上变化，而是在每次动画脉冲即绘制之前更改。`animateToFinalPosition()` 方法会马上设置最终的属性值，如果动画没开始就开始动画，这在链式依赖的弹簧动画中非常有用。`cancel()` 方法可以结束动画在其当前位置，`skipToEnd()` 方法会跳转至终止值再结束动画，可以通过 `canSkipToEnd()` 方法判断是否是阻尼动画
* 放大预览动画只需要同时动画更改目标 view 的 `X`，`Y`，`SCALE_X`，`SCALE_Y` 属性即可，不过要先计算好两个 view 最终的位置和初始缩放比
* Android 提供了预加载的布局改变动画，可以通过 `android:animateLayoutChanges="true"` 属性告诉系统开启默认动画，或者通过 `LayoutTransition` API 设置
* Activity 内部的布局过渡动画: 过渡动画框架可以在开始 `Scene` 和结束 `Scene` 开始过渡动画，`Scene` 存储着 view hierarchy 状态，包括所有 view 和其属性值，开始 `Scene` 可以通过 `setExitAction()` 定义过渡动画开始前要执行的操作，结束 `Scene` 可以通过 `Scene.setEnterAction()` 定义过渡动画完成后要执行的操作。如果 view hierarchy 是静态不变的，可以通过布局文件描述和加载 `Scene.getSceneForLayout(mSceneRoot, R.layout.a_scene, this)`，否则可以手动创建 `new Scene(mSceneRoot, mViewHierarchy)`。`Transition` 的内置子类包括 `AutoTransition`、`Fade`、`ChangeBounds`，可以在 `res/transition/` 目录下定义内置的 `<fade xmlns:android="http://schemas.android.com/apk/res/android" />`，多个组合包裹在 `<transitionSet>` 标签中，然后使用 `TransitionInflater.from(this).inflateTransition(R.transition.fade_transition)` 加载。还可以手动创建 `new Fade()`。开始过渡动画时只需要执行 `TransitionManager.go(mEndingScene, mFadeTransition)` 即可。默认是对 `Scene` 中所有的 view 作动画，可以通过 `addTarget()` 或 `removeTarget()` 在开始过渡动画前进行调整。如果不想在两个 view hierarchy 间进行过渡，而是在同一个 view hierarchy 状态更改后执行过渡动画，那就不需要使用 `Scene` 了，先利用 `TransitionManager.beginDelayedTransition(mRootView, mFade)` 让系统记录 view 的更改，然后增删 view 来更改 view hierarchy 的状态，系统会在重绘 UI 时执行延迟过渡动画。由于 `SurfaceView` 由非 UI 线程更新，所以它的过渡可能有问题，`TextureView` 在一些过渡类型上可能有问题，`AdapterView` 与过渡动画框架不兼容，`TextView` 的大小过渡动画可能有问题
* Activity 之间的过渡动画: 需要 Android 5.0 (API level 21) ，内置的进入退出过渡动画包括: explode 从中央进入或退出，slide 从一边进入或退出，fade 透明度渐变进入或退出。内置的共享元素过渡动画包括: changeBounds 动态更改目标 view 的边界，changeClipBounds 动态裁剪目标 view 的边界，changeTransform 动态更改目标 view 的缩放和旋转，changeImageTransform 动态更改目标 view 的缩放和尺寸。过渡动画需要两个 Activity 都要开启 window 的内容过渡: `android:windowActivityTransitions` 属性设置为 `true` 或者代码中手动 `getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS)` 开启。`setExitTransition()` 和 `setSharedElementExitTransition()` 方法可以为起始 Activity 设置退出过渡动画，`setEnterTransition()` 和 `setSharedElementEnterTransition()` 方法可以为目标 Activity 设置进入过渡动画。激活目标 Activity 的时候需要携带 `ActivityOptions.makeSceneTransitionAnimation(this).toBundle()` 的 Bundle，返回的时候要使用 `finishAfterTransition()` 方法。共享元素需要使用 `android:transitionName` 属性或者 `View.setTransitionName()` 方法指定名字，多个共享元素使用 `Pair.create(view1, "agreedName1")` 传递信息
* 自定义过渡动画需要继承 `Transition`，实现 `captureStartValues()` 和 `captureEndValues()` 方法捕获过渡的 view 属性值并告诉过渡框架，具体实现为通过 `transitionValues.view` 检索当前 view，通过 `transitionValues.values.put(PROPNAME_BACKGROUND, view.getBackground())` 存储属性值，为了避免冲突 key 的格式必须为 `package_name:transition_name:property_name`。同时还要实现 `createAnimator(ViewGroup sceneRoot, TransitionValues startValues, TransitionValues endValues)` 方法，框架调用这个方法的次数取决于开始和结束 scene 需要更改的元素数
* 动画可能会影响性能，必要时可以启用 Profile GPU Rendering 进行调试
### 其它
* Android 8.0 (API level 26) 开始支持自适应启动图标，自适应启动图标必须由前景和背景两部分组成，尺寸必须都是 108 x 108 dp，其中内部的 72 x 72 dp 用来显示图标，靠近四个边缘的 18 dp 是保留区域，用来进行视觉交互
* 对于字体大小自适应的 `TextView` 宽和高都不能是 `wrap_content`，`autoSizeTextType` 默认是 `none`，设置为 `uniform` 开启自适应，默认最小 `12sp`，最大 `112sp`，粒度 `1px`。`autoSizePresetSizes` 属性可以设置预置的一些大小
* Android 8.0 (API level 26) 开始支持 XML 自定义字体，兼容库可以兼容到 Android 4.1 (API level 16)，字体文件路径为 `res/font/`，使用属性为 `fontFamily`，获取 `Typeface` 为 `getResources().getFont(R.font.myfont);`，兼容库使用 `ResourcesCompat.getFont(context, R.font.myfont)`
* Android 9 (API level 28) 支持控件放大镜功能，`Magnifier` 的 `show()` 方法的参数是相对于被放大 View 的左上角的坐标
* 工程中的 Drawable 资源只能有一个状态，你**不应该手动更改它的任何属性**，否则会影响到其它使用这个 Drawable 资源的地方
* Android 7.0 (API level 24) 开始支持在 XML 文件中使用自定义 Drawable，公共顶级类使用全限定名作为标签名即可 `<com.myapp.MyDrawable>`，公共静态内部类可以使用 class 属性 `class="com.myapp.MyTopLevelClass$MyDrawable"`
* Android 5.0 (API level 21) 开始支持为 Drawable 设置 tint
* Android 5.0 (API level 21) 开始支持矢量图，支持库可以支持到 Android 2.1 (API level 7+)，兼容低版本是需要 Gradle 插件版本大于 2.0+ 时添加 `vectorDrawables.useSupportLibrary = true` 并使用 `VectorDrawableCompat` 和 `AnimatedVectorDrawableCompat`
## BroadcastReceiver 相关
* Android 9 (API level 28) 开始 `NETWORK_STATE_CHANGED_ACTION` 广播不再包含 SSID，BSSID 等信息
* Android 8.0 (API level 26) 开始限制应用静态注册一些非当前应用专属的隐式广播的 `BroadcastReceiver`，免除这项限制的广播包括 `ACTION_LOCKED_BOOT_COMPLETED` [等](https://developer.android.com/guide/components/broadcast-exceptions)不太可能影响用户体验的广播
* Android 7.0 (API level 24) 开始不能发送和接收 `ACTION_NEW_PICTURE` 和 `ACTION_NEW_VIDEO` 系统广播，可以通过 `JobInfo` 和 `JobParameters` 完成。不能静态注册 `CONNECTIVITY_ACTION` 广播，如果想在网络变化时调度任务可以选择使用 WorkManager，如果只在应用运行期间监听网络变化使用 `ConnectivityManager` 比动态注册注销 `BroadcastReceiver` 更优雅
* 应该尽量在代码中动态注册注销 `BroadcastReceiver`
* `onReceive()` 方法中不能进行复杂工作否则会导致 ANR，`onReceive()` 方法一旦执行完，系统可能就认为这个广播接收器已经没用了，随时会杀掉包含这个广播接收器的进程，包括这个进程启动的线程。使用 `goAsync()` 方法可以在 `PendingResult#finish()` 方法执行前为广播接收器的存活争取更多的时间，但最好还是使用 `JobScheduler` 等方式进行长时间处理工作
* 使用 `sendBroadcast()` 方法发的广播属于常规广播，所有能接收这个广播的广播接收器接收到广播的顺序是不可控的
* 使用 `sendOrderedBroadcast()` 方法发的广播属于有序广播，根据广播接收器的优先级一个接一个地传递这条广播，相同优先级的顺序不可控，广播接收器可以选择继续传递给下一个，也可以选择直接丢掉
* 使用 `LocalBroadcastManager.getInstance(this).sendBroadcast()` 方法发的广播属于应用进程内的本地广播，这样的广播只有应用自己知道，比系统级的全局广播更安全更有效率
* 为了保证广播的 action 全局唯一，action 的名字最好使用应用的包名作为前缀，最好声明成静态字符串常量
## 数据存储与共享
### 存储方式
* 系统会在**安装应用**时在**内部存储器**的文件系统中为应用生成一个私有文件目录，一般是 `/data/data/your.application.package/` 或 `/data/user/0/your.application.package/`，当**卸载应用**时这个目录也会被删除。这个目录除了系统和应用自己谁都无法访问，除非拥有权限。这个路径下有个 `files/` 子目录用来存储应用的文件，可以通过 `getFilesDir()` 方法获取这个路径表示，可以通过 `openFileOutput(filename, Context.MODE_PRIVATE)` 写这个目录下的文件。还有一个 `cache/` 子目录用来存储临时缓存文件，系统可能会在存储空间不足时清理这个目录，可以通过 `getCacheDir()` 方法获取这个路径表示，可以通过 `File#createTempFile(fileName, null, context.getCacheDir())` 方法在这个目录下创建一个临时文件。还有一个 `shared_prefs/` 子目录用来以 XML 文件的形式存储简单的键值对数据，需要使用 `SharedPreferences` API 进行管理
* 读写外存（外存是指可以被移除的**外部存储器**）文件需要先动态申请 `READ_EXTERNAL_STORAGE` 或 `WRITE_EXTERNAL_STORAGE` 权限，然后检查外存是否可用: `Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())` 表示可写，`Environment.MEDIA_MOUNTED.equals(state) || Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)` 表示可读。外存根目录可以使用 `Environment.getExternalStorageDirectory()` 方法获取，一般是 `/storage/emulated/0/`，使用 `new File(Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES), albumName)` 可以读写外存公有目录的文件。使用 `getExternalFilesDir(null)` 可以获取该应用的外存根目录，一般是 `/storage/emulated/0/Android/data/your.application.package/files`，使用 `new File(context.getExternalFilesDir(Environment.DIRECTORY_PICTURES), albumName)` 可以读写文件，应用的外存目录也会在**卸载应用**时被删除。使用 `getExternalCacheDir()` 可以获取应用的外存缓存目录。
* 使用 `myFile.delete()` 或 `myContext.deleteFile(fileName)` 删除文件
* 直接使用 SQLite API 进行数据库操作既麻烦又容易出错，建议使用 Room 等其它 ORM 库进行数据库操作
* 获取 `SharedPreferences` 的方式有三个: 通过 `PreferenceManager.getDefaultSharedPreferences()` 可以获取或创建名字为 `context.getPackageName() + "_preferences"` 模式为 `Context.MODE_PRIVATE` 的文件。通过 `MainActivity.this.getPreferences(Context.MODE_PRIVATE)` 可以获取或创建名字为当前 `Activity` 类名的文件。使用 `context.getSharedPreferences("file1", Context.MODE_PRIVATE)` 可以获取或创建名字是 file1 的文件。`MODE_WORLD_READABLE` 和 `MODE_WORLD_WRITEABLE` 从 Android 7.0 (API level 24) 开始被**禁止使用**了。`commit()` 方法会将数据同步写到磁盘所以可能会阻塞 UI，而 `apply()` 方法会异步写到磁盘。
### 分享文件
* 为了安全地共享文件，分享的文件必须通过 content URI 表示，必须授予这个 content URI 临时访问权限。`FileProvider` 作为 `ContentProvider` 的特殊子类，它的 `getUriForFile()` 静态方法可以为文件生成 content URI。
```xml
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="com.example.myapp.fileprovider"
    android:grantUriPermissions="true"
    android:exported="false">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/filepaths" />
</provider>
```
```xml
<paths>
    <files-path path="images/" name="myimages" />
</paths>
```
* `android:authorities` 属性一般是以当前应用包名为前缀的字符串，用来标志数据的所有者，多个的话用分号隔开  
* `<files-path/>` 代表 `getFilesDir()`
* `<cache-path/>` 代表 `getCacheDir()`
* `<external-path/>` 代表 `Environment.getExternalStorageDirectory()`
* `<external-files-path>` 代表 `getExternalFilesDir(null)`
* `<external-cache-path>` 代表 `getExternalCacheDir()`
* `<external-media-path>` 代表 `getExternalMediaDirs()`
```java
File imagePath = new File(getFilesDir(), "images");
File newFile = new File(imagePath, "default_image.jpg");
Uri contentUri = FileProvider.getUriForFile(getContext(), "com.example.myapp.fileprovider", newFile);
```
* 给 Intent 添加 `FLAG_GRANT_READ_URI_PERMISSION` 或 `FLAG_GRANT_WRITE_URI_PERMISSION` 的 flag 授予对这个 content URI 的临时访问权限，该权限会被目标 `Activity` 所在应用的其它组件继承，会在所在的任务结束时自动撤销授权
* 调用 `Context.grantUriPermission(package, Uri, mode_flags)` 方法也可以授予 `FLAG_GRANT_READ_URI_PERMISSION` 或 `FLAG_GRANT_WRITE_URI_PERMISSION` 权限，但只有在主动调用 `revokeUriPermission()` 方法后或者重启系统后才会撤销授权
```java
List<ResolveInfo> activities = getPackageManager().queryIntentActivities(intent,
        PackageManager.MATCH_DEFAULT_ONLY);
if (activities.size() > 0) {
    for (ResolveInfo resolveInfo : activities) {
        grantUriPermission(resolveInfo.activityInfo.packageName,
                outputUri, Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
    }
}
...
revokeUriPermission(outputUri, Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
```
### ContentProvider
* `ContentProvider` 的数据形式和关系型数据库的表格数据类似，因此 API 也像数据库一样包含增删改查（CRUD）操作，但为了更好地组织管理一个或多个 `ContentProvider`，最好通过 `ContentResolver` 操作 `ContentProvider`
* 对于 `ContentProvider` 的增删改查操作，**不能直接在 UI 线程上执行**
* `Uri` 和 `ContentUris` 类的静态方法可以方便地构造 content URI
```sql
SELECT _ID, word, locale FROM words WHERE word = <userinput> ORDER BY word ASC;
```
```java
mCursor = getContentResolver().query(
        UserDictionary.Words.CONTENT_URI,
        mProjection,
        mSelectionClause,
        mSelectionArgs,
        mSortOrder);
```
* 为了防止 SQL 注入，禁止拼接 SQL 语句，如 mSelectionClause 不能直接包含 selectionArgs 参数值
* `ContentProvider` 所在应用本身的组件可以随便访问它，不需要授权
* 如果 `ContentProvider` 的应用不指定任何权限，那么其它应用就无法访问这个 `ContentProvider` 的数据
* 使用者需要事先通过 `<uses-permission>` 标签获取访问权限
* 创建 `ContentProvider` 需要继承 `ContentProvider` 并实现增删改查等一系列方法: `onCreate()` 在系统创建 provider 后马上调用，可以在这里创建数据库，但不要在这里做耗时操作。`getType()` 返回 content URI 的 MIME 类型。`query()`、`insert()`、`update()`、`delete()` 进行增删改查。除了 `onCreate()` 方法其它方法必须要保证是线程安全的
### 其它
* Android 7.0 (API level 24) 开始禁止使用 file URI 进行文件共享
* Android 7.1.1 (API level 25) 开始安装 APK 时必须声明 `REQUEST_INSTALL_PACKAGES` 权限，数据必须通过 `FileProvider` 形式共享，数据类型是 `application/vnd.android.package-archive`，必须给 Intent 添加 `FLAG_GRANT_READ_URI_PERMISSION` flag 授予临时访问权限
```java
Intent installIntent = new Intent(Intent.ACTION_VIEW);
File apkPath = new File(Environment.getExternalStorageDirectory(), "apks");
File apkFile = new File(apkPath, "myapp.apk");
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    Uri contentUri = FileProvider.getUriForFile(MainActivity.this, "com.example.myapp.fileprovider", apkFile);
    installIntent.setDataAndType(contentUri, "application/vnd.android.package-archive");
    installIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
} else {
    installIntent.setDataAndType(Uri.fromFile(apkFile), "application/vnd.android.package-archive");
}
installIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
if (installIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(installIntent);
}
```
## Notification 相关
* Android 5.0 (API level 21) 开始通知可以出现在锁屏页面
* Android 7.0 (API level 24) 开始可以在通知中直接输入文本或执行一些自定义操作，如直接回复按钮
* Android 8.0 (API level 26) 开始所有的通知必须属于一个 channel，channel 被用户看作是 categories，即**通知类别**，用户通过通知类别来精确管理各个应用或一个应用内的通知。一个应用可以有多个通知类别，如私信类别、好友请求类别、应用更新类别等等。可以给每个通知类别指定通知的 importance，即**重要程度**，Urgent（紧急）会发出提示音并在屏幕上弹出通知，High（高）会发出提示音，Medium（中）不发出提示音，Low（低）不发出提示音并且不会出现在状态栏中。在 Android 8.0 (API level 26) 以下的系统中通知的重要程度表现为 priority，即**优先级**。对应关系分别为: `IMPORTANCE_HIGH` 对应 `PRIORITY_HIGH` 或 `PRIORITY_MAX`，`IMPORTANCE_DEFAULT` 对应 `	PRIORITY_DEFAULT`，`IMPORTANCE_LOW` 对应 `		PRIORITY_LOW`，`IMPORTANCE_MIN` 对应 `		PRIORITY_MIN`。在应用启动时可以执行下面的代码创建通知类别，可以无副作用地多次执行
```java
private void createNotificationChannel() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        CharSequence name = getString(R.string.channel_name);
        String description = getString(R.string.channel_description);
        int importance = NotificationManager.IMPORTANCE_DEFAULT;
        NotificationChannel channel = new NotificationChannel(CHANNEL_ID, name, importance);
        channel.setDescription(description);
        NotificationManager notificationManager = getSystemService(NotificationManager.class);
        notificationManager.createNotificationChannel(channel);
    }
}
```
* 通过 `NotificationChannel` 的 `enableLights()`，`setLightColor()` 等方法可以指定该通知类别默认的通知行为，但是一旦创建了应用就不能再对它做任何更改了，只有用户自己可以更改设置。可以通过 Intent 引导用户跳转至对应设置页
```java
Intent intent = new Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS);
intent.putExtra(Settings.EXTRA_APP_PACKAGE, getPackageName());
intent.putExtra(Settings.EXTRA_CHANNEL_ID, myNotificationChannel.getId());
startActivity(intent);
```
* 查询用户当前的通知类别的设置可以通过 `getNotificationChannel()`、`getNotificationChannels()`、`getVibrationPattern()`、`getImportance()` 等方法获取
* 使用 `deleteNotificationChannel(id)` 可以删除通知类别，但是在开发模式下可能需要重装应用或者清除数据才会完全删除
* 通知类别也可以分组
```java
// The id of the group.
String groupId = "my_group_01";
// The user-visible name of the group.
CharSequence groupName = getString(R.string.group_name);
NotificationManager mNotificationManager =
        (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
mNotificationManager.createNotificationChannelGroup(new NotificationChannelGroup(groupId, groupName));
```
* Android 5.0 (API level 21) 开始支持勿扰模式（Do Not Disturb）以禁止任何通知产生的声音和震动。Total silence（完全阻止）会阻止包括闹钟视频游戏在内的所有声音和震动，Alarms only（仅限闹钟）会阻止除了闹钟外的所有声音和震动，Priority only（自订）可以定制要屏蔽的信息通话等系统范围内的通知。`setCategory()` 方法可以设置所属的系统范围的勿扰类别
* 每个通知类别可以选择是否覆盖勿扰模式的设置，当勿扰模式设置为“仅限优先事项”时，可以允许继续接收此类通知
* Android 8.1 (API level 27) 开始每秒最多播放一次通知提示音，如果一秒内有多个通知那么只播放一秒内的第一个通知提示音，如果一秒内多次频繁更新一个通知，那么系统可能会丢弃一些通知更新
* 最好使用 `NotificationCompat` 和 `NotificationManagerCompat` 等兼容库中的类以便方便地适配低版本系统
* `setSmallIcon()` 方法可以设置小图标，应用名和时间是由系统设置的，`setLargeIcon()` 方法可以设置右边大图标，`setContentTitle()` 和 `setContentText()` 方法可以设置通知的标题和内容，`setPriority()` 方法可以为 Android 8.0 (API level 26) 以下的系统设置通知优先级。系统范围的预定义通知类别包括 `NotificationCompat.CATEGORY_ALARM`，`NotificationCompat.CATEGORY_REMINDER` 等类别，这个类别在勿扰模式中有用，可以通过 `setCategory()` 方法指定所属的系统范围通知类别
* 默认的通知内容会收缩成一行，可以通过 `setStyle()` 方法设置其他可展开的通知样式，`.setStyle(new NotificationCompat.BigTextStyle().bigText(emailObject.getSubjectAndSnippet()))` 可以设置大文本块样式。`.setStyle(new NotificationCompat.InboxStyle().addLine(messageSnippet1)` 可以设置多行的 inbox 样式。`.setStyle(new NotificationCompat.MessagingStyle(resources.getString(R.string.reply_name)).addMessage(message1))` 可以设置消息样式，但是此样式会忽略 `setContentTitle()` 和 `setContentText()` 方法的设置，但可以通过 `setConversationTitle()` 方法设置该聊天所属的群组名。`setStyle(new android.support.v4.media.app.Notification.MediaStyle().setShowActionsInCompactView(1).setMediaSession(mMediaSession.getSessionToken()))` 可以设置媒体样式的通知，属于 `CATEGORY_TRANSPORT` 类别。
* 通知的点击事件可以通过 `setContentIntent()` 方法设置 `PendingIntent` 对象完成，`setAutoCancel(true)` 可以在点击后自动移除通知
```java
Intent intent = new Intent(this, AlertDetails.class);
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, intent, 0);
NotificationCompat.Builder mBuilder = new NotificationCompat.Builder(this, CHANNEL_ID)
        .setSmallIcon(R.drawable.notification_icon)
        .setContentTitle("My notification")
        .setContentText("Hello World!")
        .setLargeIcon(myBitmap)
        .setStyle(new NotificationCompat.BigPictureStyle()
                .bigPicture(myBitmap)
                .bigLargeIcon(null))
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true);
```
* 通过 `NotificationManagerCompat#notify()` 方法可以显示通知，你需要定义一个唯一的 int 值的 ID 作为这个通知的 ID，保存这个 ID 以便之后更新或移除这个通知
```java
NotificationManagerCompat notificationManager = NotificationManagerCompat.from(this);
notificationManager.notify(notificationId, mBuilder.build());
```
* 通过 `addAction()` 方法可以添加操作按钮
* 添加回复按钮的流程为:
```java
private static final String KEY_TEXT_REPLY = "key_text_reply";

String replyLabel = getResources().getString(R.string.reply_label);
RemoteInput remoteInput = new RemoteInput.Builder(KEY_TEXT_REPLY)
        .setLabel(replyLabel)
        .build();

PendingIntent replyPendingIntent =
        PendingIntent.getBroadcast(getApplicationContext(),
                conversation.getConversationId(),
                getMessageReplyIntent(conversation.getConversationId()),
                PendingIntent.FLAG_UPDATE_CURRENT);

NotificationCompat.Action action =
        new NotificationCompat.Action.Builder(R.drawable.ic_reply_icon,
                getString(R.string.label), replyPendingIntent)
                .addRemoteInput(remoteInput)
                .build();

Notification newMessageNotification = new Notification.Builder(mContext, CHANNEL_ID)
        .setSmallIcon(R.drawable.ic_message)
        .setContentTitle(getString(R.string.title))
        .setContentText(getString(R.string.content))
        .addAction(action)
        .build();

NotificationManagerCompat notificationManager = NotificationManagerCompat.from(this);
notificationManager.notify(notificationId, newMessageNotification);
```
```java
private CharSequence getMessageText(Intent intent) {
    // 在 BroadcastReceiver 接收的 Intent 中可以根据之前的 KEY 拿到文本框的内容
    Bundle remoteInput = RemoteInput.getResultsFromIntent(intent);
    if (remoteInput != null) {
        return remoteInput.getCharSequence(KEY_TEXT_REPLY);
    }
    return null;
 }
```
```java
// 在回复完成后更新通知表示已经处理这次回复，也可以调用 setRemoteInputHistory() 方法附加回复的内容
Notification repliedNotification = new Notification.Builder(context, CHANNEL_ID)
        .setSmallIcon(R.drawable.ic_message)
        .setContentText(getString(R.string.replied))
        .build();
NotificationManagerCompat notificationManager = NotificationManagerCompat.from(this);
notificationManager.notify(notificationId, repliedNotification);
```
* 通过 `setProgress(PROGRESS_MAX, PROGRESS_CURRENT, false)` 可以给通知添加确定进度条，通过 `setProgress(0, 0, true)` 可以添加不确定进度条，通过 `setProgress(0, 0, false)` 可以在完成后移除进度条
* `setVisibility()` 方法可以设置锁屏时的通知显示策略，`VISIBILITY_PUBLIC`（显示所有通知）表示完整地显示通知内容，`VISIBILITY_SECRET`（完全不显示内容）表示不显示通知的任何信息，`VISIBILITY_PRIVATE`（隐藏敏感通知内容）表示只显示图标标题等基本信息
* `NotificationManagerCompat#notify()` 方法传递之前的通知 ID 可以更新之前的通知，调用 `setOnlyAlertOnce()` 方法以便只在第一次出现通知时提示用户
* 用户可以主动清除通知，创建通知时调用 `setAutoCancel()` 方法可以在用户点击通知后清除通知，创建通知时调用 `setTimeoutAfter()` 方法可以在超时后由系统自动清除通知，可以随时调用 `cancel()` 或 `cancelAll()` 方法清除之前的通知
* 点击通知后启动的 `Activity` 分为两种，一种是应用的正常用户体验流中的常规 `Activity`，拥有任务完整的返回栈。还有一种是仅仅用来展示通知的详细内容的特殊`Activity`，它不需要返回栈。
* 对于常规 `Activity` 需要先通过 `android:parentActivityName` 属性或者 `android.support.PARENT_ACTIVITY` 的 `<meta-data>` 标签指定层级关系，然后
```java
Intent resultIntent = new Intent(this, ResultActivity.class);
TaskStackBuilder stackBuilder = TaskStackBuilder.create(this);
stackBuilder.addNextIntentWithParentStack(resultIntent);
PendingIntent resultPendingIntent =
        stackBuilder.getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT);
```
* 对于特殊 `Activity` 需要先指定 `android:taskAffinity=""` 和 `android:excludeFromRecents="true"` 以避免在之前的任务中启动，然后
```java
Intent notifyIntent = new Intent(this, ResultActivity.class);
notifyIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK);
PendingIntent notifyPendingIntent = PendingIntent.getActivity(
        this, 0, notifyIntent, PendingIntent.FLAG_UPDATE_CURRENT
);
```
* Android 7.0 (API level 24) 开始，如果一个应用同时有 4 个及以上的通知，那么系统会自动将它们合并成一组，应用也可以自己定义和组织分组，用户点击后可以展开成一些单独的通知，老版本可以考虑使用 inbox 样式代替。每个通知可以通过 `setGroup()` 方法指定所属分组，通过 `setSortKey()` 方法排序，通过 `setGroupAlertBehavior()` 指定通知行为，默认是 `GROUP_ALERT_ALL` 表示组内所有的通知都可能产生声音和震动。系统默认会自动生成通知组的摘要，你也可以单独创建一个表示通知组摘要的通知
```java
int SUMMARY_ID = 0;
String GROUP_KEY_WORK_EMAIL = "com.android.example.WORK_EMAIL";

Notification newMessageNotification1 =
    new NotificationCompat.Builder(MainActivity.this, CHANNEL_ID)
        .setSmallIcon(R.drawable.ic_notify_email_status)
        .setContentTitle(emailObject1.getSummary())
        .setContentText("You will not believe...")
        .setGroup(GROUP_KEY_WORK_EMAIL)
        .build();

Notification newMessageNotification2 =
    new NotificationCompat.Builder(MainActivity.this, CHANNEL_ID)
        .setSmallIcon(R.drawable.ic_notify_email_status)
        .setContentTitle(emailObject2.getSummary())
        .setContentText("Please join us to celebrate the...")
        .setGroup(GROUP_KEY_WORK_EMAIL)
        .build();

Notification summaryNotification =
    new NotificationCompat.Builder(MainActivity.this, CHANNEL_ID)
        .setContentTitle(emailObject.getSummary())
        //set content text to support devices running API level < 24
        .setContentText("Two new messages")
        .setSmallIcon(R.drawable.ic_notify_summary_status)
        //build summary info into InboxStyle template
        .setStyle(new NotificationCompat.InboxStyle()
                .addLine("Alex Faarborg  Check this out")
                .addLine("Jeff Chang    Launch Party")
                .setBigContentTitle("2 new messages")
                .setSummaryText("janedoe@example.com"))
        //specify which group this notification belongs to
        .setGroup(GROUP_KEY_WORK_EMAIL)
        //set this notification as the summary for the group
        .setGroupSummary(true)
        .build();

NotificationManagerCompat notificationManager = NotificationManagerCompat.from(this);
notificationManager.notify(emailNotificationId1, newMessageNotification1);
notificationManager.notify(emailNotificationId2, newMessageNotification2);
notificationManager.notify(SUMMARY_ID, summaryNotification);
```
* Android 8.0 (API level 26) 开始应用启动图标可以自动添加一个小圆点表示有新的通知，用户长按应用启动图标可以查看和处理通知，调用 `mChannel.setShowBadge(false)` 可以禁用小圆点标志，调用 `setNumber(messageCount)` 可以设置长按后显示给用户的消息数，调用 `setBadgeIconType(NotificationCompat.BADGE_ICON_SMALL)` 可以设置长按后的图标样式，通过 `setShortcutId()` 可以隐藏重复的 shortcut
* 自定义通知内容的样式需要 `setStyle(new NotificationCompat.DecoratedCustomViewStyle())` 样式，然后调用 `setCustomContentView()` 和 `setCustomBigContentView()` 方法指定自定义的折叠和展开布局（一般折叠布局限制高度为 64 dp，展开布局高度限制为 256 dp），布局中的控件要使用兼容库的样式，如 `style="@style/TextAppearance.Compat.Notification.Title"`。如果不想使用标准通知模板，不调用 `setStyle()` 只调用 `setCustomBigContentView()` 即可
* Android 5.0 (API level 21) 开始支持顶部弹出的 heads-up 通知，可能触发 heads-up 通知的条件有: 用户的 `Activity` 正处于全屏模式并使用了 `fullScreenIntent`；Android 8.0 (API level 26) 及更高的设备上通知的重要程度为 `IMPORTANCE_HIGH`；Android 8.0 (API level 26) 以下的设备上通知的优先级为 `PRIORITY_HIGH` 或 `PRIORITY_MAX` 并且开启了铃声或震动
## Service 相关
- `Service` 运行在所在进程的主线程中，它不会自动创建线程，如果你不指定进程的话它甚至不会创建额外的进程，所以如果要执行耗时操作的话应该创建一个新的线程去执行
- `Service` 有三种类型，一种是 **Foreground service**，用来执行用户能从通知栏感知到的后台操作，一种是 **Background service**，是用户感知不到的，但是 Android 8.0 (API level 26) 开始系统会对这类 `Service` 进行各种限制，一种是 **Bound service**，是通过 `bindService()` 方法绑定到其他组件的服务，Bound service 基于 C/S 架构的思想，被绑定的组件可以向这个服务发请求、接收响应数据、甚至 IPC 交互，只要有一个组件绑定了它，他就会马上运行，多个组件可以同时绑定它，当所有都解绑时，它就会被销毁  
- 使用 `startService()` 方法启动 `Service` 时会导致它收到 `onStartCommand()` 回调，服务会一直运行直到你主动调用它的 `stopSelf()` 方法或其它组件主动调用 `stopService()` 方法。如果它只是 Bound service 可以不是实现该方法  
- 使用 `bindService()` 方法启动 `Service` 时不会回调 `onStartCommand()` 方法，而会回调 `onBind()` 方法，为了与它的 client 通信这个方法需要返回 `IBinder`，如果你不允许它绑定就返回空，解绑使用 `unbindService()` 方法
- 一个 `Service` 可以启动多次，但是只能停止一次，`stopSelf(int)` 方法可以在满足指定的 `startId` 时停止
- `Service` 和 `Activity` 一样其所在的的进程可能随时被系统杀掉，同样需要做好销毁重建的工作  
- `IntentService` 作为 `Service` 的子类可以方便地在工作线程中完成多个任务，多个任务是一个接一个的执行，所以不会存在线程安全问题，内部是借助一个 `HandlerThread` 实现异步处理的，当所有请求都完成后会自动销毁，`onBind()` 方法返回了空，为了方便调试可以在构造器中指定工作线程的名字，如果想重写 `onCreate()`、`onStartCommand()`、`onDestroy()` 方法的实现必须调用父类的实现，`IntentService` 是在一个工作线程中完成多个任务，所以如果想在多个线程中完成多个任务可以直接继承 `Service` 并借助 `HandlerThread` 等线程技术实现
- 相对于 `IntentService` 更推荐使用 `JobIntentService`，`JobIntentService` 借助了 `JobScheduler` 和 `AsyncTask` 完成更灵活的任务调度和处理，只需要申请好 `WAKE_LOCK` 权限 `JobScheduler` 就可以完成 `WakeLock` 的管理，使用 `enqueueWork(Context, Class, int, Intent)` 静态方法提交任务就可以让 `onHandleWork(Intent)` 回调中的代码被更好地调度执行了
- 不能绑定的 `Service` 只能通过 `PendingIntent` 进行组件间的通信
- Foreground service 的通知栏通知只能通过停止服务或者从前台移除来解除  
- Android 9 (API level 28) 开始 Foreground service 必须请求 `FOREGROUND_SERVICE` 权限
- 使用 `startForeground()` 方法可以向系统请求以 Foreground service 模式运行，`stopForeground()` 可以请求退出该模式
## 后台任务
- 每个进程都有一个主线程用来完成任务，一般主线程结束了那么意味着整个任务完成了，进程就会自动结束退出了 
- Android 应用的主线程用来进行测量绘制 UI、协调用户操作、接收生命周期事件等工作，是与用户的感知直接关联的，所以通常也被叫做 UI 线程，如果在这个线程中做太多工作，那么会导致这个线程挂起或者卡顿，导致糟糕的用户体验。所以像解码 bitmap、读写磁盘、执行网络请求等需要长时间计算和处理的操作都应该放到单独的后台线程中去做  
- 后台线程虽然是用户感觉不到的，但通常却是最消耗系统资源的，有的线程大部分时间都在占用 CPU 完成复杂的计算，我们管这种称为 CPU 密集型操作，有的线程大部分时间都在进行 I/O 的读写操作，我们管这种叫做 I/O 密集型操作。我们可以根据不同的操作类型选择不同的策略来处理以便最大化系统的吞吐量同时最小化所需代价。同时长时间运行的后台线程也加剧了电量的消耗，所以不管是操作系统还是开发者都需要 **对这些后台线程的行为进行限制**  
- 在创建一个后台任务之前，我们需要先要对它分析一下，它是要马上执行还是可以延迟执行？它需要系统满足指定条件才能执行吗？它需要在精确的时间点执行吗？  
### WorkManager
- 通过 WorkManager 可以优雅地执行 **可延迟执行的** 异步任务，当应用退出后仍然可以继续执行，当满足系统条件（联网、充电、重启）时仍然可以触发任务的执行  
- 特别适合用来向后台发送日志或分析数据，或者用来周期性的与服务器同步数据
- WorkManager 在 Android 6.0 (API level 23) 及以上系统上借助 `JobScheduler` 实现，在之前的系统上借助 `BroadcastReceiver` 和 `AlarmManager` 实现
- WorkManager 可以对任务添加网络条件和充电状态等条件限制，可以调度一次性的或周期性的任务，可以监听和管理被调度的任务，可以将多个任务连在一起
- 一次性的任务可以使用 `OneTimeWorkRequest`，周期性的任务使用 `PeriodicTimeWorkRequest`
- 如果指定了多个限制，那么只有在所有限制都满足的情况下任务才会执行:  
```java
Constraints constraints = new Constraints.Builder()
    .setRequiresDeviceIdle(true)
    .setRequiresCharging(true)
     .build();
OneTimeWorkRequest compressionWork =
                new OneTimeWorkRequest.Builder(CompressWorker.class)
     .setConstraints(constraints)
     .build();
```
- 任务交给系统后可能会马上被执行，可以通过 `setInitialDelay(10, TimeUnit.MINUTES)` 设置一个最小延时  
- 如果需要重试任务可以在 `Worker` 中使用 `Result.retry()` 完成，采用的补偿策略默认是 `EXPONENTIAL` 指数级的，可以使用 `setBackoffCriteria()` 方法调整策略  
- 可以通过 `setInputData()` 方法为任务设置输入数据，在 `Worker` 中可以通过 `getInputData()` 方法获取到输入数据，`Result.success()` 和 `Result.failure()` 可以携带输出数据。数据应该尽可能的简单，不能超过 10KB  
- `addTag` 方法可以给任务打 Tag，然后就可以使用 `WorkManager.cancelAllWorkByTag(String)` 和 `WorkManager.getWorkInfosByTagLiveData(String)` 等方法方便操作任务了
- 如果一个任务的先决任务没有完成那么会被认为是 `BLOCKED` 态
- 如果任务的限制和定时满足要求那么会被认为是 `ENQUEUED` 态
- 如果任务正在执行那么会被认为是 `RUNNING` 态
- 如果任务返回了 `Result.success()` 那么会被认为是 `SUCCEEDED` 态，这是最终态，只有 `OneTimeWorkRequest` 可能进入这个状态
- 如果任务返回了 `Result.failure()` 那么会被认为是 `FAILED` 态，这是最终态，只有 `OneTimeWorkRequest` 可能进入这个状态，所有相关的任务也会被标记为 `FAILED` 且不会被执行
- 显式取消一个没终止的 `WorkRequest` 会被认为是 `CANCELLED` 态，所有相关的任务也会被标记为 `CANCELLED` 且不会被执行
- `WorkManager.getWorkInfoById(UUID)` 和 `WorkManager.getWorkInfoByIdLiveData(UUID)` 等方法可以定位想要的任务进行观察
- 可以将任务连在一起:  
```java
WorkManager.getInstance()
    .beginWith(Arrays.asList(filter1, filter2, filter3))
    .then(compress)
    .then(upload)
    .enqueue();
```
### Foreground service
对于用户触发的必须马上执行且必须执行完的后台任务，需要使用 Foreground services 实现，它既告诉系统这个应用正在执行重要的任务不能被杀掉，又通过通知栏告诉用户有后台工作正在执行  
### AlarmManager
如果任务需要在精确的时间点执行，可以使用 `AlarmManager`
### DownloadManager
如果需要执行一个长时间的 HTTP 下载任务，可以使用 `DownloadManager`。`DownloadManager` 独立于应用之外，可以在下载失败、更改网络连接、系统重启后进行重试  
```java
public static long downloadApk(String url, String title, String desc) {
    DownloadManager.Request request = new DownloadManager.Request(Uri.parse(url));
    request.setAllowedNetworkTypes(DownloadManager.Request.NETWORK_MOBILE | DownloadManager.Request.NETWORK_WIFI)
            .setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE)
            .setTitle(title)
            .setDescription(desc)
            .setDestinationInExternalFilesDir(MyApplication.getInstance(), null, "apks")
            .allowScanningByMediaScanner();
    DownloadManager downloadManager = (DownloadManager) MyApplication.getInstance().getSystemService(Context.DOWNLOAD_SERVICE);
    return downloadManager.enqueue(request);
}
```
# 小技巧
* 测试 Deep links:
```shell
adb shell am start
    -W -a android.intent.action.VIEW
    -d "example://gizmos" com.example.android
```
* 测试 Android App Links:
```shell
adb shell am start -a android.intent.action.VIEW
    -c android.intent.category.BROWSABLE
    -d "http://domain.name:optional_port"
```
* 应用安装完 20s 后获取所有应用的链接处理策略:
```shell
adb shell dumpsys package domain-preferred-apps
```
* 模拟系统杀掉应用进程:
```shell
adb shell am kill com.some.package
```
* 将文件导入手机:
```shell
adb push com.some.package /sdcard/
```
* `.nomedia` 文件会导致其所在目录不被 Media Scanner 扫描到
# 附录
## 系统栏适配
```java
/**
 * 华为手机刘海屏适配
 *
 * @author frank
 * @see <a href="https://developer.huawei.com/consumer/cn/devservice/doc/50114">《华为刘海屏手机安卓O版本适配指导》</a>
 */
public class HwNotchSizeUtil {

    private static final int FLAG_NOTCH_SUPPORT = 0x00010000;

    /**
     * 是否是刘海屏手机
     *
     * @param context Context
     * @return true：刘海屏 false：非刘海屏
     */
    public static boolean hasNotchInScreen(Context context) {
        boolean ret = false;
        try {
            ClassLoader cl = context.getClassLoader();
            Class HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
            Method get = HwNotchSizeUtil.getMethod("hasNotchInScreen");
            ret = (boolean) get.invoke(HwNotchSizeUtil);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return ret;
    }

    /**
     * 获取刘海尺寸
     *
     * @param context Context
     * @return int[0]值为刘海宽度 int[1]值为刘海高度
     */
    public static int[] getNotchSize(Context context) {
        int[] ret = new int[]{0, 0};
        try {
            ClassLoader cl = context.getClassLoader();
            Class HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
            Method get = HwNotchSizeUtil.getMethod("getNotchSize");
            ret = (int[]) get.invoke(HwNotchSizeUtil);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return ret;
    }

    /**
     * 设置应用窗口在华为刘海屏手机使用刘海区
     *
     * @param window Window
     */
    public static void setFullScreenWindowLayoutInDisplayCutout(Window window) {
        if (window == null) {
            return;
        }
        WindowManager.LayoutParams layoutParams = window.getAttributes();
        try {
            Class layoutParamsExCls = Class.forName("com.huawei.android.view.LayoutParamsEx");
            Constructor con = layoutParamsExCls.getConstructor(ViewGroup.LayoutParams.class);
            Object layoutParamsExObj = con.newInstance(layoutParams);
            Method method = layoutParamsExCls.getMethod("addHwFlags", int.class);
            method.invoke(layoutParamsExObj, FLAG_NOTCH_SUPPORT);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 设置应用窗口在华为刘海屏手机不使用刘海区显示
     *
     * @param window Window
     */
    public static void setNotFullScreenWindowLayoutInDisplayCutout(Window window) {
        if (window == null) {
            return;
        }
        WindowManager.LayoutParams layoutParams = window.getAttributes();
        try {
            Class layoutParamsExCls = Class.forName("com.huawei.android.view.LayoutParamsEx");
            Constructor con = layoutParamsExCls.getConstructor(ViewGroup.LayoutParams.class);
            Object layoutParamsExObj = con.newInstance(layoutParams);
            Method method = layoutParamsExCls.getMethod("clearHwFlags", int.class);
            method.invoke(layoutParamsExObj, FLAG_NOTCH_SUPPORT);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```

```java
/**
 * 小米手机刘海屏适配
 *
 * @author frank
 * @see <a href="https://dev.mi.com/console/doc/detail?pId=1293">《小米刘海屏 Android O 适配》</a>
 * @see <a href="https://dev.mi.com/console/doc/detail?pId=1341">《小米刘海屏 Android P 适配》</a>
 */
public class XiaomiNotchSizeUtil {

    private static final int FLAG_NOTCH_OPEN = 0x00000100;
    private static final int FLAG_NOTCH_PORTRAIT = 0x00000200;
    private static final int FLAG_NOTCH_LANDSCAPE = 0x00000400;

    /**
     * 是否是刘海屏手机
     *
     * @param context Context
     * @return true：刘海屏 false：非刘海屏
     */
    public static boolean hasNotchInScreen(Context context) {
        boolean ret = false;
        try {
            ret = "1".equals(getSystemProperty("ro.miui.notch"));
        } catch (Exception e) {
            e.printStackTrace();
        }
        return ret;
    }

    /**
     * 获取刘海尺寸
     *
     * @param context Context
     * @return int[0]值为刘海宽度 int[1]值为刘海高度
     */
    public static int[] getNotchSize(Context context) {
        int[] ret = new int[]{0, 0};
        try {
            int widthResId = context.getResources().getIdentifier("notch_width", "dimen", "android");
            if (widthResId > 0) {
                ret[0] = context.getResources().getDimensionPixelSize(widthResId);
            }
            int heightResId = context.getResources().getIdentifier("notch_height", "dimen", "android");
            if (heightResId > 0) {
                ret[1] = context.getResources().getDimensionPixelSize(heightResId);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return ret;
    }

    /**
     * 横竖屏都绘制到耳朵区
     *
     * @param window Window
     */
    public static void setFullScreenWindowLayoutInDisplayCutout(Window window) {
        if (window == null) {
            return;
        }
        try {
            Method method = Window.class.getMethod("addExtraFlags",
                    int.class);
            method.invoke(window, FLAG_NOTCH_OPEN | FLAG_NOTCH_PORTRAIT | FLAG_NOTCH_LANDSCAPE);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 横竖屏都不会绘制到耳朵区
     *
     * @param window Window
     */
    public static void setNotFullScreenWindowLayoutInDisplayCutout(Window window) {
        if (window == null) {
            return;
        }
        try {
            Method method = Window.class.getMethod("clearExtraFlags",
                    int.class);
            method.invoke(window, FLAG_NOTCH_OPEN | FLAG_NOTCH_PORTRAIT | FLAG_NOTCH_LANDSCAPE);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static String getSystemProperty(String key) {
        String ret = null;
        BufferedReader bufferedReader = null;
        try {
            Process process = Runtime.getRuntime().exec("getprop " + key);
            bufferedReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
            String line;
            StringBuilder stringBuilder = new StringBuilder();
            while ((line = bufferedReader.readLine()) != null) {
                stringBuilder.append(line);
            }
            ret = stringBuilder.toString();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (bufferedReader != null) {
                try {
                    bufferedReader.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        return ret;
    }

}
```

```java
/**
 * OPPO手机刘海屏适配
 *
 * @author frank
 * @see <a href="https://open.oppomobile.com/wiki/doc#id=10159">《OPPO凹形屏适配说明》</a>
 */
public class OppoNotchSizeUtil {

    /**
     * 是否是刘海屏手机
     *
     * @param context Context
     * @return true：刘海屏 false：非刘海屏
     */
    public static boolean hasNotchInScreen(Context context) {
        return context.getPackageManager().hasSystemFeature("com.oppo.feature.screen.heteromorphism");
    }

}
```

```java
/**
 * VIVO手机刘海屏适配
 *
 * @author frank
 * @see <a href="https://dev.vivo.com.cn/documentCenter/doc/103">《异形屏应用适配指南》</a>
 */
public class VivoNotchSizeUtil {

    private static final int MASK_NOTCH_IN_SCREEN = 0x00000020;
    private static final int MASK_ROUNDED_IN_SCREEN = 0x00000008;

    /**
     * 是否是刘海屏手机
     *
     * @param context Context
     * @return true：刘海屏 false：非刘海屏
     */
    public static boolean hasNotchInScreen(Context context) {
        boolean ret = false;
        try {
            ClassLoader cl = context.getClassLoader();
            Class FtFeature = cl.loadClass("android.util.FtFeature");
            Method get = FtFeature.getMethod("isFeatureSupport", int.class);
            ret = (boolean) get.invoke(FtFeature, MASK_NOTCH_IN_SCREEN);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return ret;
    }

}
```

```java
/**
 * 锤子手机刘海屏适配
 *
 * @author frank
 * @see <a href="https://resource.smartisan.com/resource/61263ed9599961d1191cc4381943b47a.pdf">《Smartisan 开发者文档》</a>
 */
public class SmartisanNotchSizeUtil {

    private static final int MASK_NOTCH_IN_SCREEN = 0x00000001;

    /**
     * 是否是刘海屏手机
     *
     * @param context Context
     * @return true：异形屏 false：非异形屏
     */
    public static boolean hasNotchInScreen(Context context) {
        boolean ret = false;
        try {
            ClassLoader cl = context.getClassLoader();
            Class DisplayUtilsSmt = cl.loadClass("smartisanos.api.DisplayUtilsSmt");
            Method get = DisplayUtilsSmt.getMethod("isFeatureSupport", int.class);
            ret = (boolean) get.invoke(DisplayUtilsSmt, MASK_NOTCH_IN_SCREEN);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return ret;
    }

}
```
## 通过相机或相册选择一张图片
```java
findViewById(R.id.chooseImg).setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.setType("image/*");
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivityForResult(intent, REQUEST_IMAGE_GET);
        }
    }
});
findViewById(R.id.takePicture).setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
            File photoFile = null;
            try {
                photoFile = createImageFile();
            } catch (IOException ex) {
                ex.printStackTrace();
            }
            if (photoFile != null) {
                Uri photoURI = FileProvider.getUriForFile(MainActivity.this,
                        "com.your.package.fileprovider",
                        photoFile);
                takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoURI);
                startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);
            }
        }
    }
});

...

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_GET && resultCode == RESULT_OK) {
        Uri fullPhotoUri = data.getData();
        ParcelFileDescriptor descriptor;
        try {
            descriptor = getContentResolver().openFileDescriptor(fullPhotoUri, "r");
            FileDescriptor fd = descriptor.getFileDescriptor();
            Bitmap bitmap = BitmapFactory.decodeFileDescriptor(fd);
            imageView.setImageBitmap(bitmap);
            processImage();
        } catch (Exception e) {
            e.printStackTrace();
        }
    } else if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        int targetW = imageView.getWidth();
        int targetH = imageView.getHeight();
        BitmapFactory.Options bmOptions = new BitmapFactory.Options();
        bmOptions.inJustDecodeBounds = true;
        BitmapFactory.decodeFile(currentPhotoPath, bmOptions);
        int photoW = bmOptions.outWidth;
        int photoH = bmOptions.outHeight;
        int scaleFactor = Math.min(photoW / targetW, photoH / targetH);
        bmOptions.inJustDecodeBounds = false;
        bmOptions.inSampleSize = scaleFactor;
        Bitmap bitmap = BitmapFactory.decodeFile(currentPhotoPath, bmOptions);
        imageView.setImageBitmap(bitmap);
        processImage();
    }
}
private File createImageFile() throws IOException {
    String timeStamp = new SimpleDateFormat("yyyyMMdd_HHmmss", Locale.US).format(new Date());
    String imageFileName = "JPEG_" + timeStamp + "_";
    File storageDir = getExternalFilesDir(Environment.DIRECTORY_PICTURES);
    File image = File.createTempFile(imageFileName, ".jpg", storageDir);
    currentPhotoPath = image.getAbsolutePath();
    return image;
}
```
```xml
<paths>
    <external-path name="my_images" path="Android/data/com.your.package/files/Pictures" />
</paths>
```
