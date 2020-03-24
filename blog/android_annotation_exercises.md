# Android 中优雅地使用注解

## 概述

注解（Annotation），是源码中特殊的语法元数据，类、方法、变量、参数都可以被注解。利用注解可以标记源码以便编译器为源码生成文档和检查代码，也可以让编译器和注解处理器在编译时根据注解自动生成代码，甚至可以保留到运行时以便改变运行时的行为。
Java 内置了一些注解，如 `@Override` 注解用来表明该方法是重写父类方法，编译器会负责检查该方法与父类方法的声明是否一致。`@Deprecated` 注解用来表明该元素已经被废弃不建议使用了。`@SuppressWarnings` 注解用来表示编译器可以忽略特定警告。  
注解类型的声明和接口的声明类似，不过需要使用 `@interface` 和元注解（用来定义注解的注解）描述，每个方法声明定义了注解类型的一个元素，且方法声明不能包含任何参数或 `throws`，方法的返回类型必须是原语类型、`String`、`Class`、枚举、注解和这些类型的数组，方法可以有默认值，如:  

```java
public @interface RequestForEnhancement {
    int    id();
    String synopsis();
    String engineer() default "[unassigned]";
    String date();    default "[unimplemented]";
}
```

定义完注解类型后，就可以用它去注解一些声明了。注解是一种特殊的修饰符，可以像 `public`、`static` 或 `final` 修饰符一样使用，不过通常注解要写在这些修饰符之前。使用时为 `@` 符号加注解类型加元素值对列表并用括号括起来，如:  

```java
@RequestForEnhancement(
    id       = 2868724,
    synopsis = "Enable time-travel",
    engineer = "Mr. Peabody",
    date     = "4/1/3007"
)
public static void travelThroughTime(Date destination) { ... }
```

注解类型也可以没有方法/元素，被称为标记注解类型，如:  

```java
public @interface Preliminary { }

@Preliminary public class TimeTravel { ... }
```

如果注解类型只有一个元素，那么元素应该命名为 `value`，使用时也就可以忽略元素名和等号了，如:  

```java
public @interface Copyright {
    String value();
}

@Copyright("2002 Yoyodyne Propulsion Systems")
public class OscillationOverthruster { ... }
```

除了这些，很多注解还需要元注解去描述，如:  

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface MethodInfo {
    String author() default "Peabody";
    String date();
    int version() default 1;
}
```

`@Documented` 表明该注解类型可以被 javadoc 等工具文档化
`@Retention` 表明该注解类型可以保留多长时间，值为枚举值 `RetentionPolicy`:  

- `RetentionPolicy.SOURCE`(只保留在源码中，会被编译器丢弃)
- `RetentionPolicy.CLASS`(注解会被编译器记录在class文件中，但不需要被VM保留到运行时，这也是默认的行为)
- `RetentionPolicy.RUNTIME`(注解会被编译器记录在class文件中并被VM保留到运行时，所以可以通过反射获取)  

`@Target` 表明该注解类型可以注解哪些程序元素，如果注解类型不使用 `@Target` 描述那么表明可以注解所有程序元素，值是枚举数组`ElementType[]`:  

- `ElementType.TYPE`(类、接口(包括注解类型)、枚举的声明)
- `ElementType.FIELD`(字段(包括枚举常量)的声明)
- `ElementType.METHOD`(方法的声明)
- `ElementType.PARAMETER`(形参的声明)
- `ElementType.CONSTRUCTOR`(构造器的声明)
- `ElementType.LOCAL_VARIABLE`(本地变量的声明)
- `ElementType.ANNOTATION_TYPE`(注解类型的声明)
- `ElementType.PACKAGE`(包的声明)
- `ElementType.TYPE_PARAMETER`(泛型参数的声明)
- `ElementType.TYPE_USE`(泛型的使用)

`@Inherited` 表明该注解类型将被自动继承。也就是说，如果注解类型被 `@Inherited` 注解，此时用户查询一个类声明的注解，而类声明没被该注解类型注解，那么将自动查询该类父类的注解类型，以此类推直到找到该注解类型或达到顶层 Object 对象。

## Android Support Library 中的注解

Android Support Library 提供了很多实用注解，如可以使用 `@NonNull` 注解进行空检查，使用 `@UiThread`、`@WorkerThread` 注解进行线程检查，使用 `@IdRes` 表明这个整数代表资源引用，还可以通过 `@IntDef`、`@StringDef` 注解自定义注解来代替枚举，如描述应用中使用的字体文件:  

```java
public final class TypefaceManager {

    public static final int FONT_TYPE_ICONIC = 0;
    public static final int FONT_TYPE_IMPACT = 1;
    public static final int FONT_TYPE_HELVETICA = 2;
    public static final int FONT_TYPE_DIN = 3;

    @Retention(RetentionPolicy.SOURCE)
    @IntDef({FONT_TYPE_ICONIC, FONT_TYPE_IMPACT, FONT_TYPE_HELVETICA, FONT_TYPE_DIN})
    @interface FontType {
    }

    private Context mContext;
    private SparseArray<Typeface> mTypefaceSparseArray;

    public TypefaceManager(Context context) {
        this.mContext = context;
        this.mTypefaceSparseArray = new SparseArray<>();
    }

    public static void setTypeface(TextView textView, @FontType int fontType) {
        Typeface localTypeface = MyApplication.getInstance().getTypefaceManager().getTypeface(fontType);
        if (localTypeface != null && localTypeface != textView.getTypeface()) {
            textView.setTypeface(localTypeface);
        }
    }

    public static void setTypeface(Paint paint, @FontType int fontType) {
        Typeface localTypeface = MyApplication.getInstance().getTypefaceManager().getTypeface(fontType);
        if (localTypeface != null && localTypeface != paint.getTypeface()) {
            paint.setTypeface(localTypeface);
        }
    }

    public Typeface getTypeface(@FontType int fontType) {
        Typeface typeface = mTypefaceSparseArray.get(fontType);
        if (typeface == null) {
            try {
                String path = null;
                if (fontType == FONT_TYPE_ICONIC) {
                    path = "fonts/fontawesome-webfont.ttf";
                } else if (fontType == FONT_TYPE_IMPACT) {
                    path = "fonts/impact.ttf";
                } else if (fontType == FONT_TYPE_HELVETICA) {
                    path = "fonts/Helvetica.ttf";
                } else if (fontType == FONT_TYPE_DIN) {
                    path = "fonts/ptdin.ttf";
                }
                typeface = Typeface.createFromAsset(mContext.getAssets(), path);
                this.mTypefaceSparseArray.put(fontType, typeface);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return typeface;
    }

}
```

## 注解的使用与解析

对于 `@Retention(RetentionPolicy.RUNTIME)` 的注解，注解会被编译器记录在 class 文件中并被 VM 保留到运行时，所以可以通过反射获取，如:  

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface MethodInfo {
    String author() default "Peabody";
    String date();
    int version() default 1;
}
```

使用

```java
public class App {
    @MethodInfo(
        author = “frank”,
        date = "2018/02/27",
        version = 2)
    public String getDescription() {
        return "no description";
    }
}
```

可以写个工具在运行时利用反射获取注解:  

```java
public static void main(String[] args) {
    try {
        Class cls = Class.forName("com.frank.App");
        for (Method method : cls.getMethods()) {
            MethodInfo methodInfo = method.getAnnotation(
MethodInfo.class);
            if (methodInfo != null) {
                System.out.println("method author:" + methodInfo.author());
                System.out.println("method version:" + methodInfo.version());
                System.out.println("method date:" + methodInfo.date());
            }
        }
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

对于 `@Retention(RetentionPolicy.CLASS)` 的注解，注解会被编译器记录在 class 文件中，但不需要被 VM 保留到运行时，如:  

```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
    @IdRes int value();
}
```

也就是说，这种编译时注解适合用来在编译时自动生成代码，这就需要 apt(Annotation Processing Tool)工具查找并执行注解处理器(Annotation Processor)以生成源码和文件，最终 javac 会编译这些原始源文件和自动生成的文件。Android Gradle 插件的 2.2 版本开始支持注解处理器，你只需要使用 `annotationProcessor` 依赖注解处理器或者使用 [`javaCompileOptions.annotationProcessorOptions {}`](https://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.AnnotationProcessorOptions.html) DSL指定注解处理器即可。定义注解处理器最简单的方式就是继承 `AbstractProcessor`，在其 `process` 实现方法中实现注解元素的分析和源码文件的生成。

## 自定义注解和注解处理器

以简化一系列 `findViewById` 为例：

```java
package com.frank.simplebutterknife;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    TextView mTitleTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTitleTextView = (TextView) findViewById(R.id.titleTextView);
        mTitleTextView.setText("Hello World!");
    }
}
```

我们希望利用自定义注解和注解处理器后可以这样写:

```java
package com.frank.simplebutterknife;

import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.widget.TextView;

import simplebutterknife.BindView;

public class MainActivity extends AppCompatActivity {

    @BindView(R.id.titleTextView)
    TextView mTitleTextView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        SimpleButterKnife.bind(this);
        mTitleTextView.setText("Hello World!");
    }
}

```

也就是说 `SimpleButterKnife.bind(this);` 一行代码就完成了所有被 `@BindView` 注解的 View 的 `findViewById` 操作。而实现方式就是利用注解和注解编译器在编译时自动生成一个这样的文件:  

```java
package com.frank.simplebutterknife;

import android.view.View;
import android.widget.TextView;

public class MainActivity_ViewBinding {
  public MainActivity target;

  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  public MainActivity_ViewBinding(MainActivity target, View source) {
    this.target = target;

    target.mTitleTextView = (TextView) source.findViewById(2131165300);
  }
}
```

在 `SimpleButterKnife.bind(this);` 的实现中加载这个类并执行构造器就可以了。
实现起来也很简单，先新建一个 `java-library` 的module：`simplebutterknife-annotations`，用来声明注解：  

```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface BindView {
    @IdRes int value();
}
```

`RetentionPolicy.CLASS` 表明这个注解只会在编译时使用，`ElementType.FIELD` 表明这个注解只用于注解字段，`@IdRes` 是 android support library 中的编译时检查注解，表明注解的值必须是资源 ID，所以该 module 的依赖为:  

```gradle
dependencies {
    compileOnly 'com.google.android:android:4.1.1.4'
    api 'com.android.support:support-annotations:27.0.2'
}
```

声明完注解后，再新建一个注解处理器 `java-library` 的module：`simplebutterknife-compiler`，用来对注解的元素进行分析和生成源码文件:  

```java
@AutoService(Processor.class)
public class SimpleButterKnifeProcessor extends AbstractProcessor {

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> types = new LinkedHashSet<>();
        types.add(BindView.class.getCanonicalName());
        return types;
    }

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        ...
        return false;
    }

    class BindingSet {
        TypeName targetTypeName;
        ClassName bindingClassName;
        List<ViewBinding> viewBindings;
    }

    class ViewBinding {
        TypeName type;
        int id;
        String name;
    }
}
```

`@AutoService(Processor.class)` 注解是利用了 Google 的 [AutoService](https://github.com/google/auto/tree/master/service#autoservice) 为注解处理器自动生成 metadata 文件并将注解处理器jar文件加入构建路径，这样也就不需要再手动创建并更新 `META-INF/services/javax.annotation.processing.Processor` 文件了。
覆写 `getSupportedSourceVersion()` 方法指定可以支持最新的 Java 版本，覆写 `getSupportedAnnotationTypes()` 方法指定该注解处理器用于处理哪些注解（我们这里只处理 `@BindView` 注解）。而检索注解元素并生成代码的是 `process` 方法的实现:  

```java
Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
```

需要为每个包含注解的 Activity 都生成一个对应的 _ViewBinding 文件，所以使用 `Map` 来存储。`BindingSet` 存储 Activity 信息和它的 View 绑定信息，View 绑定信息（`ViewBinding`）包括绑定 View 的类型、View 的 ID 以及 View 的变量名。

```java
for (Element element : roundEnvironment.getElementsAnnotatedWith(BindView.class))
```

查找所有被 `@BindView` 注解的程序元素(Element)，为了简化，这里只认为被注解的元素是 View 字段且它的外层元素(EnclosingElement)为 Activity 类:  

```java
// 注解元素的外侧元素，即 View 的所在 Activity 类
TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
// 注解的 value 值，即 View 的 id
int id = element.getAnnotation(BindView.class).value();
// 注解元素的名字，即 View 变量名
Name simpleName = element.getSimpleName();
String name = simpleName.toString();
// 注解元素的类型，即 View 的类型
TypeMirror elementType = element.asType();
TypeName type = TypeName.get(elementType);
```

然后把这些信息存到 Activity 对应的 View 绑定中:  

```java
BindingSet bindingSet = bindingMap.get(enclosingElement);
if (bindingSet == null) {
    bindingSet = new BindingSet();
    TypeMirror typeMirror = enclosingElement.asType();
    TypeName targetType = TypeName.get(typeMirror);
    String packageName = MoreElements.getPackage(enclosingElement).getQualifiedName().toString();
    String className = enclosingElement.getQualifiedName().toString().substring(
            packageName.length() + 1).replace('.', '$');
    ClassName bindingClassName = ClassName.get(packageName, className + "_ViewBinding");
    bindingSet.targetTypeName = targetType;
    bindingSet.bindingClassName = bindingClassName;
    bindingMap.put(enclosingElement, bindingSet);
}
if (bindingSet.viewBindings == null) {
    bindingSet.viewBindings = new ArrayList<>();
}
ViewBinding viewBinding = new ViewBinding();
viewBinding.type = type;
viewBinding.id = id;
viewBinding.name = name;
bindingSet.viewBindings.add(viewBinding);
```

确定完 Activity 信息和它对应的 View 绑定信息后，为每个 Activity 生成对应的 `XXX_ViewBinding.java` 文件，文件内容就是前面所说类似这样的绑定类:  

```java
package com.frank.simplebutterknife;

import android.view.View;
import android.widget.TextView;

public class MainActivity_ViewBinding {
  public MainActivity target;

  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  public MainActivity_ViewBinding(MainActivity target, View source) {
    this.target = target;

    target.mTitleTextView = (TextView) source.findViewById(2131165300);
  }
}
```

虽然通过字符串拼接可以拼出这样的文件内容，但我们还得考虑 import，还得考虑大括号和换行，甚至还得考虑注释和代码美观，所以利用 [JavaPoet](https://github.com/square/javapoet) 来生成 `.java` 文件是个不错的选择:  

```java
for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
            TypeElement typeElement = entry.getKey();
            BindingSet binding = entry.getValue();

            TypeName targetTypeName = binding.targetTypeName;
            ClassName bindingClassName = binding.bindingClassName;
            List<ViewBinding> viewBindings = binding.viewBindings;
            // binding 类
            TypeSpec.Builder viewBindingBuilder = TypeSpec.classBuilder(bindingClassName.simpleName())
                    .addModifiers(Modifier.PUBLIC);
            // public的target字段用来保存 Activity 引用
            viewBindingBuilder.addField(targetTypeName, "target", Modifier.PUBLIC);
            // 构造器
            MethodSpec.Builder activityViewBuilder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC)
                    .addParameter(targetTypeName, "target");
            activityViewBuilder.addStatement("this(target, target.getWindow().getDecorView())");
            viewBindingBuilder.addMethod(activityViewBuilder.build());
            // 第二个构造器
            MethodSpec.Builder viewBuilder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC)
                    .addParameter(targetTypeName, "target")
                    .addParameter(ClassName.get("android.view", "View"), "source");
            viewBuilder.addStatement("this.target = target");
            viewBuilder.addCode("\n");
            for (ViewBinding viewBinding : viewBindings) {
                CodeBlock.Builder builder = CodeBlock.builder()
                        .add("target.$L = ", viewBinding.name);
                builder.add("($T) ", viewBinding.type);
                builder.add("source.findViewById($L)", CodeBlock.of("$L", viewBinding.id));
                viewBuilder.addStatement("$L", builder.build());
            }
            viewBindingBuilder.addMethod(viewBuilder.build());
            // 输出 Java 文件
            JavaFile javaFile = JavaFile.builder(bindingClassName.packageName(), viewBindingBuilder.build())
                    .build();
            try {
                javaFile.writeTo(processingEnv.getFiler());
            } catch (IOException e) {
                processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, e.getMessage());
            }
        }
```

好了，注解处理器已经写完了，再调整一下注解处理器 module 的依赖:  

```gradle
dependencies {
    implementation project(':simplebutterknife-annotations')
    implementation 'com.google.auto:auto-common:0.10'
    api 'com.squareup:javapoet:1.9.0'
    compileOnly 'com.google.auto.service:auto-service:1.0-rc4'
}
```

在 app module 中需要依赖注解 module 并注册注解处理器 module：  

```gradle
dependencies {
    ...
    api project(':simplebutterknife-annotations')
    annotationProcessor project(':simplebutterknife-compiler')
}
```

app module 中的工具类 `SimpleButterKnife` 的 `bind` 方法只需要加载这个自动生成的类并执行它的构造器就行了:  

```java
public final class SimpleButterKnife {

    public static void bind(Activity target) {
        View sourceView = target.getWindow().getDecorView();
        Class<?> targetClass = target.getClass();
        String targetClassName = targetClass.getName();
        Constructor constructor;
        try {
            Class<?> bindingClass = targetClass.getClassLoader().loadClass(targetClassName + "_ViewBinding");
            constructor = bindingClass.getConstructor(targetClass, View.class);
        } catch (ClassNotFoundException e) {
            // TODO Not found. should try search its superclass
            throw new RuntimeException("Not found. should try search its superclass of " + targetClassName, e);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Unable to find binding constructor for " + targetClassName, e);
        }
        try {
            constructor.newInstance(target, sourceView);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Unable to invoke " + constructor, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("Unable to invoke " + constructor, e);
        } catch (InvocationTargetException e) {
            Throwable cause = e.getCause();
            if (cause instanceof RuntimeException) {
                throw (RuntimeException) cause;
            }
            if (cause instanceof Error) {
                throw (Error) cause;
            }
            throw new RuntimeException("Unable to create binding instance.", cause);
        }
    }
}
```

重新构建下工程，就可以在 `build\generated\source\apt\debug` 目录中查看自动生成的文件了:  

```java
package com.frank.simplebutterknife;

import android.view.View;
import android.widget.TextView;

public class MainActivity_ViewBinding {
  public MainActivity target;

  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  public MainActivity_ViewBinding(MainActivity target, View source) {
    this.target = target;

    target.mTitleTextView = (TextView) source.findViewById(2131165300);
  }
}

```

此时，看一下 [Butter Knife](https://github.com/JakeWharton/butterknife) 的源码，其实就是在[此](https://github.com/shangmingchao/simplebutterknife)基础上的补充完善。

## 总结

编译时的注解和注解处理器可以生成一些模板代码，由于不涉及到反射所以也不会影响性能，注解的使用也会让代码得到简化，更加直观优雅，所以很多项目都在使用，包括 Butter Knife、Dagger2、EventBus、Glide 等开源库，所以有必要了解并使用注解，尤其是编译时注解。

## 参考

- [JDK 5.0 Developer's Guide: Annotations](https://docs.oracle.com/javase/1.5.0/docs/guide/language/annotations.html)
- [JakeWharton/butterknife](https://github.com/JakeWharton/butterknife)
