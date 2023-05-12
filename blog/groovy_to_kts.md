# 是时候把构建脚本从 Groovy 迁移至 Kotlin 了

## Kotlin scripting 的优势

对于开发者来说，保证整个项目中语言和代码的一致性是很重要的。尤其是对于 Android 开发者来说，使用 Kotlin 语言编写业务代码，同样也希望使用 Kotlin 语言来构建项目。虽然 [Kotlin 脚本](https://kotlinlang.org/docs/custom-script-deps-tutorial.html) 目前还是 Experimental 的，但是 [Gradle Kotlin DSL](https://docs.gradle.org/current/userguide/kotlin_dsl.html#kotlin_dsl) 已经足够成熟和稳定，可以胜任 Android 的构建管理  
除此之外，Kotlin 脚本相对于 Groovy 来说可读性更高，也更利于 IDE 的语法检查、自动补全和跳转。所以迁移至 Kotlin 脚本是一个很好的选择  

## 基础使用

### 顶级 settings.gradle.kts

顶级 `settings.gradle.kts` 用来定义工程级的仓库配置以及用来构建应用的 module  

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "My Application"
include(":app")
```

`pluginManagement()` 用来配置各种 Gradle 插件的配置，包括插件所属仓库、依赖处理策略、依赖版本等  
`dependencyResolutionManagement()` 用来配置 Project 以及各个子 module 共用的仓库和依赖（module 独有的依赖最好在 module 自己的 `build.gradle.kts` 中配置）  

### 顶级 build.gradle.kts

顶级 `build.gradle.kts` 用来定义 Project 以及各个子 module 共用的插件和属性  

```kotlin
plugins {
    id("com.android.application") version "8.2.0-alpha02" apply false
    id("org.jetbrains.kotlin.android") version "1.8.10" apply false
}
```

尽管可以使用如下的方法定义 module 共用的属性，但是为了解耦，尽量不要这样用  

```kotlin
ext {
    extra["sdkVersion"] = 33
    extra["appcompatVersion"] = "1.6.1"
}
// compileSdk = rootProject.extra["sdkVersion"]
```

### module 级 build.gradle.kts

module 级 `build.gradle.kts` 用来配置 module 自己的依赖和配置信息  

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.yourapp.myapplication"
    compileSdk = 33

    defaultConfig {
        applicationId = "com.yourapp.myapplication"
        minSdk = 24
        targetSdk = 33
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables {
            useSupportLibrary = true
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
    }
    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.4.3"
    }
    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.9.0")
}
```

## Groovy 迁移

补充赋值的 `=`，如 `compileSdkVersion 30` 变为 `compileSdk = 33`  
调整字符串语法，如 `"$project.rootDir/tools/proguard-rules-debug.pro"` 变为 `"${project.rootDir}/tools/proguard-rules-debug.pro"`  
用 `val`/`var` 替换 `def` 进行变量声明，如 `def building64Bit = false` 变为 `val building64Bit = false`  
调整 list 和 map 语法，如 `jvmOptions += ["-Xms4000m", "-Xmx4000m"]` 变为 `jvmOptions += listOf("-Xms4000m", "-Xmx4000m")`  

## 最佳实践

上面我们已经提到，在顶级 `build.gradle.kts` 中定义 module/subproject 共用的属性不利于解耦，那各个 module 怎么优雅地添加中央化的 dependency 和 version 呢？  
一种比较好的方式是使用 [Gradle version catalog](https://docs.gradle.org/current/userguide/platforms.html)，使用方式就像直接使用目录层级引用一样，如:  

```kotlin
implementation(libs.accompanist.systemuicontroller)
implementation(libs.androidx.activity.compose)
implementation(libs.androidx.appcompat)
implementation(libs.androidx.core.ktx)
```

前提是需要在根目录下的 gradle 目录下创建一个 `libs.versions.toml` 文件，它使用 [TOML](https://toml.io/en/)（一种简洁高效的配置文件格式）格式来书写，在这里咱们可以用来声明 `[versions]`, `[libraries]`, `[bundles]`, `[plugins]`，如:  

```toml
[versions]
accompanist = "0.28.0"
androidGradlePlugin = "8.0.0"
androidxActivity = "1.7.0"

[libraries]
accompanist-systemuicontroller = { group = "com.google.accompanist", name = "accompanist-systemuicontroller", version.ref = "accompanist" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "androidxActivity" }

[plugins]
android-application = { id = "com.android.application", version.ref = "androidGradlePlugin" }
android-library = { id = "com.android.library", version.ref = "androidGradlePlugin" }
```

依赖可以用多种形式来声明，`"com.mycompany:mylib:1.4"`, `{ module = "com.mycompany:other", version = "1.4" }` 或者 `{ group = "com.mycompany", name = "alternate", version = "1.4" }` 都可以  
如果你不想用默认的 `libs` 目录名，可以这样来自定义名字:  

```kotlin
dependencyResolutionManagement {
    defaultLibrariesExtensionName.set("projectLibs")
}
```

version catalog 是类型安全的，编译器会自动检查和自动补全。如果在某些场景下不可用，可以尝试使用不安全的 API:  

```kotlin
val versionCatalog = extensions.getByType<VersionCatalogsExtension>().named("libs")
dependencies {
    versionCatalog.findLibrary("accompanist-systemuicontroller").ifPresent {
        implementation(it)
    }
}
```

如果想要在多个团队或者多个项目中共享一个 version catalog 文件，可以这样依赖本地文件:  

```kotlin
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
```

或者更灵活一点，把 version catalog 以插件的形式使用:  

```kotlin
plugins {
    `version-catalog`
    `maven-publish`
}

catalog {
    versionCatalog {
        library("my-lib", "com.mycompany:mylib:1.2")
    }
}

publishing {
    publications {
        create<MavenPublication>("maven") {
            from(components["versionCatalog"])
        }
    }
}

dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from("com.mycompany:catalog:1.0")
            version("accompanist", "0.28.0")
        }
    }
}
```

总之，我们最终期望的，都是一个类型安全的、中央化的、灵活解耦的依赖和版本管理，使用 Kotlin + version catalog 目前来说是一个比较好的方案  

## 参考

 - [Configure your build](https://developer.android.com/build)
 - [Gradle Kotlin DSL Primer](https://docs.gradle.org/current/userguide/kotlin_dsl.html#kotlin_dsl)
 - [Sharing dependency versions between projects](https://docs.gradle.org/current/userguide/platforms.html)
 


