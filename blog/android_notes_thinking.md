# Android 学习笔记思考篇

## 概述

Android 系统从 2008 年正式发布到现在已经过去了 10 年，系统版本也来到了 9，作为开发者，或者作为用户，我们见证了系统一次次大大小小的改动，见证了系统的不断完善，见证了我们写的每个 Android 小程序给我们带来的成就感。但是，当我们写的程序越来越多时，当我们对 Android 应用开发越来越了解时，我们发现它并不完美，甚至有些简陋：  
`Service` 从字面上理解就是后台服务，一个看不见的服务不应该运行在后台吗？不应该运行在独立的进程中吗？就算运行在主进程中那不应该运行在后台线程中吗？  
文档中确实提醒过不要在主线程中进行耗时操作，那为什么在主线程中读写文件没有问题？甚至连警告都没有？读写 `SharedPreferences` 文件算不算读写文件？算不算耗时操作？  
把耗时操作放在后台线程中执行，那意味着我们需要精通 JUC？需要创建线程，维护线程，把线程变成什么 `Looper` 线程才能用 `Handler` 通信，还得考虑线程安全，什么？为了性能和防止无限创建线程引发问题还要了解并使用线程池技术？用线程池就不会有问题了么？我们能不能不关心线程、线程池、`Looper`、`Handler` 什么的，我们就是想单纯地让这段代码异步执行而已，奥，原来有 `AsyncTask` 就不用关心这些了啊，那我们还需要维护这些 `AsyncTask` 吗？这些异步任务的生命周期能跟视图组件绑定吗？不能的话怎么手动维护这些 `AsyncTask` 啊？  
异步任务执行完之后我们想直接显示个对话框行不行？什么？得先判断 `Activity` 的状态才能显示？不判断好像也没什么问题啊？退出 `Activity` 的时候还需要手动关闭各种对话框？不关闭好像也没什么问题啊？  

## 异步

Android 中的异步操作基本都是使用 Java 语言内置的，唯一的简单封装的异步类 `AsyncTask` 有几个主要回调，我们可以通过这些回调指定那些代码在异步任务开始之前执行，哪些代码在异步任务中执行，哪些代码在任务执行完成后执行：  

```java
static class Task extends AsyncTask<Integer, Integer, String> {
    String taskDesc;
    public Task(String taskDesc) {
        this.taskDesc = taskDesc;
    }
    @Override
    protected void onPreExecute() {
        super.onPreExecute();
        Log.e(TAG, taskDesc + ": " + "onPreExecute");
    }
    @Override
    protected String doInBackground(Integer... integers) {
        Log.e(TAG, taskDesc + ": " + "doInBackground " + Thread.currentThread());
        String ret = null;
        int[] array = new int[1000000];
        for (int i = 0; i < array.length; i++) {
            array[i] = i;
        }
        for (int i = 0; i < 1000000; i++) {
            long sum = 0;
            for (int j = 0; j < integers[0]; j++) {
                sum += array[j];
            }
            ret = String.valueOf(sum);
            mTotalCount++;
        }
        return ret;
    }
    @Override
    protected void onPostExecute(String s) {
        super.onPostExecute(s);
        Log.e(TAG, taskDesc + ": " + "onPostExecute " + s + ", " + mTotalCount);
    }
}
```

我们在异步任务中执行一个很简单但很耗时的计算：算一百万次数组的区间和，现在我们来执行一下这个异步任务：  

```java
mTask = new Task("task-1").execute(300);
...
@Override
protected void onDestroy() {
    super.onDestroy();
    mTask.cancel(true);
}
```

```text
16:24:40.361 E/task: task-1: onPreExecute
16:24:40.365 E/task: task-1: doInBackground Thread[AsyncTask #1,5,main]
16:24:46.778 E/task: task-1: onPostExecute 44850, 1000000
```

从输出日志中可以看到大约 6 秒后异步任务执行完了，算出了从 0 加到 300 的结果是 44850（如果还记得等差数列的求和公式那么你肯定已经知道了 44850 确实是个正确的计算结果），我们用来统计计算次数的变量也是正确的，确实是一百万次。现在我们同时执行 10 个这样的任务再看一下：  

```java
for (int i = 0; i < 10; i++) {
    mTaskList.add(new Task("task-" + i).execute(300));
}
...
@Override
protected void onDestroy() {
    super.onDestroy();
    for (AsyncTask task : mTaskList) {
        task.cancel(true);
    }
}
```

```text
16:42:06.313 E/task: task-0: onPreExecute
16:42:06.316 E/task: task-1: onPreExecute
16:42:06.316 E/task: task-2: onPreExecute
16:42:06.316 E/task: task-3: onPreExecute
16:42:06.316 E/task: task-4: onPreExecute
16:42:06.316 E/task: task-5: onPreExecute
16:42:06.316 E/task: task-6: onPreExecute
16:42:06.316 E/task: task-7: onPreExecute
16:42:06.317 E/task: task-0: doInBackground Thread[AsyncTask #1,5,main]
16:42:06.317 E/task: task-8: onPreExecute
16:42:06.317 E/task: task-9: onPreExecute
16:42:12.724 E/task: task-0: onPostExecute 44850, 1000000
16:42:12.726 E/task: task-1: doInBackground Thread[AsyncTask #2,5,main]
16:42:17.712 E/task: task-1: onPostExecute 44850, 2000000
16:42:17.715 E/task: task-2: doInBackground Thread[AsyncTask #3,5,main]
16:42:22.706 E/task: task-2: onPostExecute 44850, 3000000
16:42:22.708 E/task: task-3: doInBackground Thread[AsyncTask #4,5,main]
16:42:27.710 E/task: task-3: onPostExecute 44850, 4000000
16:42:27.710 E/task: task-4: doInBackground Thread[AsyncTask #4,5,main]
16:42:32.698 E/task: task-4: onPostExecute 44850, 5000000
16:42:32.698 E/task: task-5: doInBackground Thread[AsyncTask #4,5,main]
16:42:37.682 E/task: task-5: onPostExecute 44850, 6000000
16:42:37.683 E/task: task-6: doInBackground Thread[AsyncTask #4,5,main]
16:42:42.672 E/task: task-6: onPostExecute 44850, 7000000
16:42:42.672 E/task: task-7: doInBackground Thread[AsyncTask #4,5,main]
16:42:47.661 E/task: task-7: onPostExecute 44850, 8000000
16:42:47.663 E/task: task-8: doInBackground Thread[AsyncTask #5,5,main]
16:42:52.655 E/task: task-8: onPostExecute 44850, 9000000
16:42:52.657 E/task: task-9: doInBackground Thread[AsyncTask #6,5,main]
16:42:57.644 E/task: task-9: onPostExecute 44850, 10000000
```

什么情况？所有的异步任务为什么是一个接一个执行的啊？这个设定真的是太难以接受了  
作者在封装 `AsyncTask` 这个类时多个任务是在一个后台线程中串行执行的，后来才意识到这样效率太低了就从 Android 1.6（API Level 4）开始改成并行执行了，但是从 Android 3.0（API Level 11）开始又改成默认串行执行了，Google 给的解释是为了避免并行执行可能带来的错误？？？如果你一定要并行执行，需要使用 `executeOnExecutor()` 方法并使用类似 `AsyncTask.THREAD_POOL_EXECUTOR` 这样的线程池去执行任务。既然这样，我们试一下：  

```java
for (int i = 0; i < 10; i++) {
    mTaskList.add(new Task("task-" + i).executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, 300));
}
...
@Override
protected void onDestroy() {
    super.onDestroy();
    for (AsyncTask task : mTaskList) {
        task.cancel(true);
    }
}
```

```text
17:26:26.867 E/task: task-0: onPreExecute
17:26:26.870 E/task: task-1: onPreExecute
17:26:26.870 E/task: task-2: onPreExecute
17:26:26.870 E/task: task-0: doInBackground Thread[AsyncTask #1,5,main]
17:26:26.871 E/task: task-3: onPreExecute
17:26:26.871 E/task: task-1: doInBackground Thread[AsyncTask #2,5,main]
17:26:26.874 E/task: task-4: onPreExecute
17:26:26.874 E/task: task-5: onPreExecute
17:26:26.874 E/task: task-6: onPreExecute
17:26:26.874 E/task: task-3: doInBackground Thread[AsyncTask #4,5,main]
17:26:26.875 E/task: task-7: onPreExecute
17:26:26.875 E/task: task-8: onPreExecute
17:26:26.875 E/task: task-9: onPreExecute
17:26:26.875 E/task: task-2: doInBackground Thread[AsyncTask #3,5,main]
17:26:33.434 E/task: task-4: doInBackground Thread[AsyncTask #2,5,main]
17:26:33.434 E/task: task-5: doInBackground Thread[AsyncTask #4,5,main]
17:26:33.436 E/task: task-1: onPostExecute 44850, 3951253
17:26:33.436 E/task: task-3: onPostExecute 44850, 3951347
17:26:33.485 E/task: task-6: doInBackground Thread[AsyncTask #1,5,main]
17:26:33.486 E/task: task-0: onPostExecute 44850, 3984209
17:26:33.528 E/task: task-7: doInBackground Thread[AsyncTask #3,5,main]
17:26:33.529 E/task: task-2: onPostExecute 44850, 4014638
17:26:38.641 E/task: task-8: doInBackground Thread[AsyncTask #4,5,main]
17:26:38.643 E/task: task-9: doInBackground Thread[AsyncTask #2,5,main]
17:26:38.643 E/task: task-5: onPostExecute 44850, 7900003
17:26:38.644 E/task: task-4: onPostExecute 44850, 7900500
17:26:38.720 E/task: task-7: onPostExecute 44850, 7958289
17:26:38.757 E/task: task-6: onPostExecute 44850, 7974684
17:26:43.671 E/task: task-8: onPostExecute 44850, 9928411
17:26:43.673 E/task: task-9: onPostExecute 44850, 9928698
```

我们发现任务确实并行执行了，但是我们统计的计算次数却不是一百万次（9928698）了，出现了错误，我们这里不讨论这个错误出现的原因和怎么避免，我们更关心的是我们使用的 API 是不是符合我们正常的思维习惯，很显然这个 API 并不符合  
你可能会说了，你看源码啊，但是我们先思考一下，一个需要通过阅读完整文档和阅读源码才能正确使用的 API 真的是个好的 API 吗？思考完我们再来看一下源码，比如这篇文章 [《Android 多线程：AsyncTask的原理 及其源码分析》](https://www.jianshu.com/p/37502bbbb25a)，看完了有什么感想么？这篇文章像其他源码分析的文章一样，用了大量的代码片段和极其详细的代码注释说明源码的大概结构和逻辑，但是没有任何对于源码的个人见解，总结 `AsyncTask` 实现原理的时候说是用两个线程池 + `Handler` 实现的，但是我们想一下，如果我们不使用 `AsyncTask` 而是自己封装一个异步任务执行的辅助类，我们该怎么设计？如果任务是串行执行的，我们会用两个线程池去实现吗？`while` 和 `for` 循环难道不能用么？队列不能用么？既然 `AsyncTask` 是为了方便主线程执行异步任务的，那我们怎么避免 `AsyncTask` 在其他线程中创建和执行呢？  
我们再来看一下网络请求，Android 有网络请求的 API 吗？没有，最开始大家只能用 Java 最原始的 `URLConnection` 或者 Apache 的 `HttpClient` 做网络请求，这两个 API 不但配置复杂使用困难，出现 Bug 的风险也高，而且由于这两个 API 都没有提供异步支持所以还得通过线程、线程池或者 `AsyncTask` 等技术才能进行异步请求，所以各个公司和个人开发者都封装了自己的一套网络请求 API，或者直接使用 Android-Async-Http 或 Volley 这些别人封装的，这种情况一直持续到 Square 公司贡献了优秀的 OkHttp 和 Retrofit，现在几乎所有公司和个人开发者都在用 OkHttp 做网络请求，也享受着它带来的便利。现在我们来思考一下，Google 在这方面做了什么？Google 没有实力写出 OkHttp 这样的库么？  
像网络请求这种 I/O 密集型的操作很适合用协程去实现，然而 Java 本身不支持协程，就只能用线程去写异步代码了么？  
相对于写异步代码我们更习惯于写同步代码，但不幸的是我们连 `async` / `await` 这样的关键字都没有  

## 内存泄漏

内存泄漏是 Android 开发者讨论最多的话题之一，为什么 Android 开发者讨论的多？因为写 Android 程序很容易写出内存泄漏的代码，不管是对于新手还是有经验的开发者  

```java
// 错误的用例

private Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        resultsTextView.setText((String) msg.obj);
    }
};
...
Message message = Message.obtain();
message.obj = "Hello World!";
mHandler.sendMessageDelayed(message, 3000);
```

```java
// 错误的用例

resultsTextView.postDelayed(new Runnable() {
    @Override
    public void run() {
        resultsTextView.setText(R.string.app_name);
    }
}, 3000);
```

像上边这样的代码看上去没什么问题，就是一个文本控件 3 秒后显示一个新的文本，但是在 Android 中却是一个 “错误” 的用例，对于新手来说很容易写出上面的代码，它们可以正常编译运行且大部分情况下功能良好，如果像上面一样仅仅设置文本而不是显示对话框甚至不会出现崩溃，所以即使有些情况下出现了内存泄漏也察觉不到，除非使用分析工具进行分析  
除了上边两种用例还有一种常见的错误用例：  

```java
// 错误的用例

resultsTextView.animate().alpha(.5f).start();
```

你可能会问了，连执行一个简单的动画都会出现内存泄漏吗？是的，在动画执行结束之前，如果你退出了 Activity，这个 View 的动画不会被终止，因此这个已经退出的 Activity 也不会被回收  
还有一种比较有趣的用例是，在使用单例的时候你无意或者有意引用了 Activity 也会导致内存泄漏：  

```java
// 错误的用例

public class TypefaceManager {

    public static final int FONT_TYPE_ICONIC = 0;
    private volatile static TypefaceManager instance;
    private Context context;

    private TypefaceManager(Context context) {
        this.context = context;
    }

    public static TypefaceManager getInstance(Context context) {
        if (instance == null) {
            synchronized (TypefaceManager.class) {
                if (instance == null) {
                    instance = new TypefaceManager(context);
                }
            }
        }
        return instance;
    }

    public void setTypeface(TextView textView, int fontType) {
        ...
    }
}
...
TypefaceManager.getInstance(MainActivity.this)
        .setTypeface(resultsTextView, TypefaceManager.FONT_TYPE_ICONIC);
```

因为单例的生命周期跟应用一样长，所以当它强引用的 Activity 退出后它依然引用着这个 Activity，导致这个 Activity 即使退出了也无法被回收  
其它内存泄漏的用例我们就不一一列举，因为真的很多，我们也意识到，只要稍微不小心就很容易写出内存泄漏的代码，就算是有过几年经验的开发者也可能依然写着 `new Thread().start()` 这样的代码，但我们不能把所有的责任都推给开发者，我们思考一下，如果 API 设计的合理一点、编译器的代码检测更智能一点，可以避免多少常见的内存泄漏代码？  

## 设计缺陷

Android 系统最受人诟病的问题就是卡，为什么 iOS 那么流畅而 Android 这么卡顿呢？卡顿的原因有很多，直接原因可能是硬件性能低或者开发者水平参差不齐写出来的应用卡，但根本原因我觉得就是 Android 的设计缺陷问题，思考一下，为什么系统的应用或者 Google 的应用相对来说就很流畅呢？  
就像我们上面讨论的那样，异步困难加上很容易写出内存泄漏的代码让应用的质量很难保证，即使我们认认真真费尽力气地管理资源（如在 `onDestroy()` 生命周期方法中停止所有动画的执行、停止所有的网络请求、注销监听器、释放暂时不用的资源）也可能因为其他的原因导致应用卡顿，如过度绘制、布局层级深、序列化复杂对象、创建多个重量级对象，内存占用过高、频繁创建回收资源引发的 GC 等等都可能导致应用产生卡顿，而只有丰富经验的开发者才可能在这些方面做得很好，写出来的应用才可能很流畅  
Google 也意识到了这些，所以给 Android（或者说是 SDK）打了个补丁，还给它取了个名字，叫 Jetpack：  
> Jetpack is a suite of libraries, tools, and guidance to help developers write high-quality apps easier. These components help you follow best practices, free you from writing boilerplate code, and simplify complex tasks, so you can focus on the code you care about.

在 Jetpack 中 Google 提供了一些工具可以让开发者不再很容易写出内存泄漏和卡顿的代码了，也就是说，开发者只要使用 Jetpack 就基本可以写出不卡顿的高质量应用了  
Jetpack 中确实提供了很多很基本很有趣甚至很优秀的实现，如 `LiveData` 不但实现了像 Rx 一样的可观察数据源，还可以自动跟观察者（Activity/Fragment）的生命周期绑定，`ViewModel` 让 Android 的 MVVM 变为可能，Data Binding 让数据驱动视图的思想变为可能，`Lifecycle` 让我们可以从臃肿的生命周期方法中解脱出来，`Room` 让我们可以方便且安全地持久化数据  
Jetpack 确实有很多优点，但并不完美，你可以使用它也可以不使用它，它的学习成本也很高，很多人排斥使用 Data Binding，因为布局的 XML 文件和源码的 Java 文件离的太远了，XML 文件中也可能包含简单的业务代码，所以一个业务逻辑可能需要同时阅读这些文件才能知道详细的信息，代码可读性可能会降低，这在一些开发者看来是无法接受的  

## 下一个十年

Android 的首个十年已经过去了，历史也证明了它是个成功的移动操作系统，这要归功于它的开放和自由，归功于无数的 Android 开发者为它开发的应用，归功于手机厂商们对它的支持，下一个十年，Android 系统依然会是除了 iOS 外最受欢迎的操作系统。但是下下个十年，下下下个十年它还会是吗？从技术上来说没有比它更优秀的移动操作系统吗？  
你可能会说了，一个成功的操作系统光从技术上优秀是远远不够的，是这样的，Windows Phone 就是最好的例子，甚至连 Google 自己都无法马上用新的操作系统取代 Android 操作系统。但是历史总是在进步的，技术在进步，人们的需求在提高，上个世纪的语言 Java 语言越来越难以满足开发者尤其是 Android 开发者的需要，所以 Google 和开发者很想逐渐用新的语言（如 Kotlin）替代它，就像 Swift 替代 OC 一样，而 Android 操作系统亦是如此，Google 难道没有意识到 Android 的设计缺陷吗？Google 难道没有想过用新的操作系统替代 Android 吗？  
你可能已经想到了，Flutter 啊，Flutter 不是操作系统，它是一个 UI 框架，一个 Fuchsia 操作系统使用的 UI 框架，而 Google 对于正在研发的 Fuchsia 操作系统一直很低调，它的内核采用的是微内核计划中的一个名字叫 Zircon 的微内核，是一个对硬件要求很低的高效内核，一个非类 UNIX 的全新内核，内核源码的提交最近几年也越来越频繁。Flutter 可以写 Android 和 iOS 应用，虽然看起来像 React 一样是个跨平台的框架，但是却有几分兵马未动粮草先行的味道  

## 思考

几年前刚自学几个月 Java 和 Android 的我就使用了它参加了比赛，写了第一个让我很有成就感的应用，写了我的第一篇技术博客，直到现在，我依旧享受着开发的 Android 应用带给我的成就感，带给我的一切。然而技术之路尤其是 Android 技术之路向来就不平坦，经历过 Eclipse 安装 ADT 插件的艰难，经历过十几分钟才能启动且严重卡顿的 Android 模拟器，经历过修改一行代码需要编译几分钟的煎熬，经历过适配各个机型 ROM 的痛苦，经历过进阶的迷茫，经历过莫名其妙的系统 Bug 的无奈  
无论如何，希望以后依然能够保持对技术的热情，保持对技术的宽容，更重要的是保持对生活的热爱，愿出走半生，归来仍是少年  
