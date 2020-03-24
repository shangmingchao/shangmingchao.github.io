# 工程尝试

> 磨刀不误砍柴工

## Code Style

Kotlin 代码的 lint 检查采用 [ktlint](https://github.com/pinterest/ktlint)，插件采用 [spotless](https://github.com/diffplug/spotless)

```groovy
plugins {
    id "com.diffplug.gradle.spotless" version "3.26.0"
}
...
spotless {
    kotlin {
        target "**/*.kt"
        ktlint("0.35.0").userData([
                'max_line_length': '100',
                'disabled_rules' : 'no-wildcard-imports'
        ])
    }
}
```

`max_line_length` 表示限制每行最大长度为 100 个字符，`disabled_rules` 表示禁用通配符引入的限制  
`xml` 代码的格式化需要自定义 scheme 文件，然后导入到 IDE：Preferences 或 Settings -> Editor -> Code Style -> 点击齿轮 -> 选择 Import Scheme... -> 选择自定义的 scheme 文件  
确保 `Strip trailing spaces on Save` 被勾选，保存的时候可以自动删掉行尾多余的空格  
确保 `Ensure line feed at end of file on Save` 被勾选，保存的时候可以在文件末尾自动加个换行以符合 POSIX 标准  
确保 `Optimize imports on the fly` 被勾选，可以让 IDE 在写代码时自动调整 imports  
确保 `File Header` 正确填写为：  

```text
/**
 *
 *
 * @author frank
 * @date ${DATE} ${TIME}
 */
```

确保函数注释格式为：

```text
/**
 * Returns the absolute value of the given [number].
 */
```

## Git

### .gitignore

```groovy
*.keystore
keystore.properties

app/release
*.apk

bin/
gen/
out/
build/

local.properties

.DS_Store

*.iml
.idea/*
!.idea/copyright

.gradle

captures/

.externalNativeBuild
.cxx
```

### Git Hooks

`pre-commit` 钩子在书写提交信息前运行，可以用来检查是不是遗忘了 TODO，测试有没有覆盖，代码规不规范等，可以通过 `git commit --no-verify` 绕过这个钩子

```bash
#!/bin/bash

echo "Executing spotlessCheck"
./gradlew spotlessCheck >/dev/null
check=$?
if [ $check -eq 0 ]; then
  echo "Your code style is perfect!"
  exit 0
fi
echo "Executing spotlessApply"
./gradlew spotlessApply >/dev/null
check=$?
if [ $check -eq 0 ]; then
  echo "Your code style is not good, but has been corrected."
  git add .
  exit 0
fi
exit 1
```

`commit-msg` 钩子用来对提交信息进行调整，它接收一个参数，即存有当前提交信息的临时文件的路径

```bash
#!/bin/bash

if [ ! -f "$1" ]; then
    echo "file does not exist: $1"
    exit 1
fi
msg=$(cat "$1")
if [ ${#msg} -lt 6 ]; then
    echo "Commit message can not be less than 6 characters!"
    exit 1
fi
```

`pre-push` 钩子会在 `git push` 更新了远程引用但是还没传输对象时运行

```bash
#!/bin/bash
./gradlew build
exit $?
```

最后写一个部署 Git Hooks 的脚本：

```bash
#!/bin/bash

RED='\033[0;1;31m'
NC='\033[0m' # No Color

GIT_DIR=$(git rev-parse --git-dir 2>/dev/null)
GIT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)

if [[ ! "$GIT_ROOT" =~ /Han$ ]]; then
  echo -e "${RED}ERROR:${NC} Please run this script from the cloned Han directory."
  exit 1
fi

echo "Installing git pre-commit hook"
echo
cp "${GIT_ROOT}/tools/pre-commit" "${GIT_DIR}/hooks/pre-commit" &&
  chmod +x "${GIT_DIR}/hooks/pre-commit"

echo "Installing git commit-msg hook"
echo
cp "${GIT_ROOT}/tools/commit-msg" "${GIT_DIR}/hooks/commit-msg" &&
  chmod +x "${GIT_DIR}/hooks/commit-msg"

echo "Installing git pre-push hook"
echo
cp "${GIT_ROOT}/tools/pre-push" "${GIT_DIR}/hooks/pre-push" &&
  chmod +x "${GIT_DIR}/hooks/pre-push"

cat <<-EOF
Please import the code style settings in Android Studio:
  * open Settings -> Editor -> Code Style
  * click the gear icon and select "Import Scheme..."
  * find the file ${GIT_ROOT}/tools/iosched-codestyle.xml
Additionally, checking the following settings helps avoid miscellaneous issues:
  * Settings -> Editor -> General -> Strip trailing spaces on Save
  * Settings -> Editor -> General -> Ensure line feed at end of file on Save
  * Settings -> Editor -> General -> Auto Import -> Optimize imports on the fly
EOF
```

## CI

采用 [Travis CI](https://travis-ci.org)

### 缓存

为了避免每次构建都上传 Gradle 缓存，需要添加额外命令：

```yml
before_cache:
  - rm -f  $HOME/.gradle/caches/modules-2/modules-2.lock
  - rm -fr $HOME/.gradle/caches/*/plugin-resolution/
cache:
  directories:
    - $HOME/.gradle/caches/
    - $HOME/.gradle/wrapper/
    - $HOME/.android/build-cache
```

### 加密

使用 `gem install travis` 命令安装 Travis CLI  
cd 到工程根目录进行敏感文件加密，com 域名需要添加 `--com` 或 `--pro` 参数，org 域名需要添加 `--org` 参数，`travis endpoint` 可以查看当前在哪个域名下，`travis endpoint --com --set-default` 可以进行域名设置，如果用 com 域名的话要先使用 `travis login --pro` 命令登录，如果需要 cli 自动将已加密信息添加到 `.travis.yml` 文件的话需要使用 `--add` 参数  
使用 `travis encrypt-file han.keystore` 命令加密签名文件，会生成以 `.enc` 结尾的加密签名文件，并自动在 Travis 脚本中添加解密命令  
使用 `travis encrypt storePassword="android"` 命令加密 `storePassword`  
使用 `travis encrypt keyPassword="android"` 命令加密 `keyPassword`

### 部署

使用 `travis setup releases` 命令自动添加将构建生成的 `app/build/outputs/apk/release/app-release.apk` 文件部署到 GitHub Releases 的脚本，但要添加 `skip_cleanup: true` 以禁止 Travis 构建完后自动删除生成的文件，添加 `tags: true` 以便只有在打 tag 时才进行部署  

### Code Coverage

采用 [CodeCov](https://codecov.io/)  

## Code Analysis

采用 [Codacy](https://www.codacy.com/)  

## 附录

### Shell 语法

- 赋值语句的 `=` 两边不能有空格
- `$(...)` 和反引号 ``` 都可以用来包裹需要被优先执行的命令
- `test` 命令和中括号 `[]` 命令作用是一样的，通常用作控制语句的条件，中括号与里面的元素要用空格隔开
- 字符串可以不加引号，可以加单引号，可以加双引号
- 单引号里的变量无效，会被当做普通字符。里面也不能出现单个无意义的单引号，转义也不行，但可以成对出现以实现字符串拼接
- 字符串截取为 `${string:index:size}`
- 脚本中可以直接使用的系统环境变量包括：`$HOME`，`$PATH`，`$PS1` 命令提示符，`$PS2` 二级命令提示符，`$IFS` 输入的分隔符，`$0` 脚本的名字，`$1`、`$2` ... `${10}` ... 等传给脚本的参数，`$#` 传给脚本的参数个数，`$*` 带分隔符的参数列表串，`$@` 不带分隔符的参数列表，`$$` 脚本的进程号，`$?` 最后一个命令的返回值（0 表示正常退出）
- `echo -e "OK! \n"` 中的 `-e` 表示开启转义
- `#` 表示个数，如 `${#str}` 表示 str 的字符个数，`${#array_name[@]}` 表示数组的个数
- 可以用 `expr` 命令进行一些计算，表达式的各元素要用空格隔开，`&`、`|`、`<`、`<=`、`>`、`>=`、`*` 要用 `\` 进行转义，`&` 和 `|` 只是简单的逻辑运算，`expr1 & expr2` 表示只要有一个表达式为零，则等于零，否则等于 `expr1`，`expr1 | expr2` 表示若 expr1 非零，则等于 expr1 ，否则等于 expr2
- 字符串比较：`=`、`!=`、`-n` 不为空、`-z` 为空
- 数值比较：`-eq`、`-gt` 等等
- 文件测试：`-d` 是目录、`-e` 存在、`-f` 存在且为普通文件、`-r` 可读、`-w` 可写、`-x` 可执行、`-g` set-group-id、`-u` set-user-id
- 逻辑运算：`-a` 与、`-o` 或、`!` 非
- `/dev/null` 是一个黑洞，会丢弃任何写入其中的数据，但会告诉你写成功了，如果试图读取它它会直接告诉你 EOF，也就是读完了
- 标准输入 STDIN 为 0，标准输出 STDOUT 为 1，标准错误 STDERR 为 2
- `>` 重定向输出，`<` 重定向输入，`>>` 输出追加，`>&` 输出合并，`<&` 输入合并，`<< tag` 把 tag 开始到结束直接的内容作为输入
- `command > /dev/null 2>&1` 可以屏蔽错误和输出

### increase-build-version.sh

```shell
#!/bin/bash

while read -r line; do
  if [[ "$line" =~ ^versionBuild ]]; then
    build=$(echo "$line" | awk '/versionBuild/' | awk '{print $3}')
    build=$((build + 1))
    src="$line"
    result="versionBuild = $build"
    break
  fi
done <version.properties
if [[ -n $result ]]; then
  echo "$result"
  sed -i '' 's/'"$src"'/'"$result"'/g' version.properties
fi
```

### dependencies.gradle

```groovy
ext.deps = [:]

def build_versions = [:]
build_versions.min_sdk = 21
build_versions.target_sdk = 29
ext.build_versions = build_versions

def versions = [:]
versions.android_gradle_plugin = '4.1.0-alpha03'
versions.kotlin = "1.3.60"
versions.findbugs = "3.0.2"
versions.androidx = "1.1.0"
versions.androidx_lifecycle = "2.2.0-rc03"
versions.androidx_navigation = "2.2.0-rc03"
versions.androidx_test = "1.3.0-alpha03"
versions.constraint_layout = "1.1.3"
versions.junit = "4.12"
versions.espresso = "3.3.0-alpha02"
versions.uiautomator = "2.2.0"
versions.retrofit = "2.6.1"
versions.okhttp = "4.2.2"
versions.stetho = "1.5.1"
versions.room = "2.2.2"
versions.koin = "2.1.0-alpha-7"
ext.versions = versions

def deps = [:]

deps.android_gradle_plugin = "com.android.tools.build:gradle:$versions.android_gradle_plugin"
deps.kotlin_gradle_plugin = "org.jetbrains.kotlin:kotlin-gradle-plugin:$versions.kotlin"
deps.navigation_gradle_plugin = "androidx.navigation:navigation-safe-args-gradle-plugin:$versions.androidx_navigation"

deps.kotlin = "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$versions.kotlin"

def androidx = [:]
androidx.app_compat = "androidx.appcompat:appcompat:$versions.androidx"
androidx.ktx_core = "androidx.core:core-ktx:$versions.androidx"
androidx.ktx_collection = "androidx.collection:collection-ktx:$versions.androidx"
androidx.ktx_fragment = "androidx.fragment:fragment-ktx:$versions.androidx"
androidx.ktx_lifecycle_runtime = "androidx.lifecycle:lifecycle-runtime-ktx:$versions.androidx_lifecycle"
androidx.ktx_lifecycle_viewmodel = "androidx.lifecycle:lifecycle-viewmodel-ktx:$versions.androidx_lifecycle"
androidx.ktx_lifecycle_livedata = "androidx.lifecycle:lifecycle-livedata-ktx:$versions.androidx_lifecycle"
androidx.ktx_navigation_fragment = "androidx.navigation:navigation-fragment-ktx:$versions.androidx_navigation"
androidx.ktx_navigation_ui = "androidx.navigation:navigation-ui-ktx:$versions.androidx_navigation"
androidx.constraint_layout = "androidx.constraintlayout:constraintlayout:$versions.constraint_layout"
deps.androidx = androidx

deps.junit = "junit:junit:$versions.junit"

def androidx_test = [:]
androidx_test.core = "androidx.test:core:1.2.1-alpha02"
androidx_test.runner = "androidx.test:runner:$versions.androidx_test"
androidx_test.rules = "androidx.test:rules:$versions.androidx_test"
androidx_test.junit = "androidx.test.ext:junit:1.1.2-alpha02"
androidx_test.truth = "androidx.test.ext:truth:$versions.androidx_test"
androidx_test.espresso_core = "androidx.test.espresso:espresso-core:$versions.espresso"
androidx_test.espresso_contrib = "androidx.test.espresso:espresso-contrib:$versions.espresso"
androidx_test.espresso_intents = "androidx.test.espresso:espresso-intents:$versions.espresso"
androidx_test.uiautomator = "androidx.test.uiautomator:uiautomator:$versions.uiautomator"
deps.androidx_test = androidx_test

def retrofit = [:]
retrofit.core = "com.squareup.retrofit2:retrofit:$versions.retrofit"
retrofit.converter_gson = "com.squareup.retrofit2:converter-gson:$versions.retrofit"
deps.retrofit = retrofit

def okhttp = [:]
okhttp.logging_interceptor = "com.squareup.okhttp3:logging-interceptor:$versions.okhttp"
deps.okhttp = okhttp

def room = [:]
room.core = "androidx.room:room-runtime:$versions.room"
room.compiler = "androidx.room:room-compiler:$versions.room"
room.ktx = "androidx.room:room-ktx:$versions.room"
room.testing = "androidx.room:room-testing:$versions.room"
deps.room = room

def koin = [:]
koin.core = "org.koin:koin-core:$versions.koin"
koin.ext = "org.koin:koin-core-ext:$versions.koin"
koin.test = "org.koin:koin-test:$versions.koin"
koin.android = "org.koin:koin-android:$versions.koin"
koin.androidx_scope = "org.koin:koin-androidx-scope:$versions.koin"
koin.androidx_viewmodel = "org.koin:koin-androidx-viewmodel:$versions.koin"
koin.androidx_fragment = "org.koin:koin-androidx-fragment:$versions.koin"
koin.androidx_ext = "org.koin:koin-androidx-ext:$versions.koin"
deps.koin = koin

def stetho = [:]
stetho.core = "com.facebook.stetho:stetho:$versions.stetho"
stetho.okhttp3_helper = "com.facebook.stetho:stetho-okhttp3:$versions.stetho"
deps.stetho = stetho

ext.deps = deps
```

### build.gradle(project)

```groovy
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    apply from: 'dependencies.gradle'
    repositories {
        google()
        jcenter()
        maven { url 'https://dl.bintray.com/kotlin/kotlin-eap' }
    }
    dependencies {
        classpath deps.android_gradle_plugin
        classpath deps.kotlin_gradle_plugin
        classpath deps.navigation_gradle_plugin
    }
}

plugins {
    id "com.diffplug.gradle.spotless" version "3.26.0"
}

allprojects {
    repositories {
        google()
        jcenter()
        maven { url 'https://dl.bintray.com/kotlin/kotlin-eap' }
    }
}

spotless {
    kotlin {
        target "**/*.kt"
        ktlint("0.35.0").userData([
                'max_line_length': '100',
                'disabled_rules' : 'no-wildcard-imports'
        ])
    }
}
```

### build.gradle(app module)

```groovy
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
apply plugin: 'kotlin-kapt'
apply plugin: "androidx.navigation.safeargs.kotlin"

def keystorePropertiesFile = rootProject.file("keystore.properties")
def keystoreProperties = new Properties()
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

def versionPropertiesFile = rootProject.file("version.properties")
def versionProperties = new Properties()
versionProperties.load(new FileInputStream(versionPropertiesFile))
def versionMajor = versionProperties['versionMajor'].toInteger()
def versionMinor = versionProperties['versionMinor'].toInteger()
def versionPatch = versionProperties['versionPatch'].toInteger()
def versionBuild = versionProperties['versionBuild'].toInteger()

android {
    compileSdkVersion build_versions.target_sdk

    defaultConfig {
        applicationId "com.frank.han"
        minSdkVersion build_versions.min_sdk
        targetSdkVersion build_versions.target_sdk
        versionCode versionMajor * 10000 + versionMinor * 1000 + versionPatch * 100 + versionBuild
        versionName "${versionMajor}.${versionMinor}.${versionPatch}"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    signingConfigs {
        release {
            keyAlias "han"
            keyPassword keystoreProperties.containsKey("keyPassword") ? keystoreProperties['keyPassword'] : System.getenv("keyPassword")
            storeFile file("../han.keystore")
            storePassword keystoreProperties.containsKey("storePassword") ? keystoreProperties['storePassword'] : System.getenv("storePassword")
        }
    }

    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
        }
        debug {
            testCoverageEnabled true
        }
    }

    testOptions {
        unitTests.returnDefaultValues = true
        unitTests.includeAndroidResources = true
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "1.8"
    }

    applicationVariants.all { variant ->
        variant.outputs.all {
            outputFileName = "${variant.name}_${versionMajor}.${versionMinor}.${versionPatch}.${versionBuild}.apk"
        }
    }
}

configurations.all {
    resolutionStrategy {
        eachDependency { details ->
            if (details.requested.group == 'androidx.appcompat'
                    && details.requested.name != 'multidex'
                    && details.requested.name != 'multidex-instrumentation') {
                details.useVersion versions.androidx
            }
            if (details.requested.group == 'com.google.code.findbugs') {
                details.useVersion versions.findbugs
            }
        }
    }
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation deps.kotlin
    implementation deps.androidx.app_compat
    implementation deps.androidx.ktx_core
    implementation deps.androidx.ktx_collection
    implementation deps.androidx.ktx_fragment
    implementation deps.androidx.ktx_lifecycle_runtime
    implementation deps.androidx.ktx_lifecycle_viewmodel
    implementation deps.androidx.ktx_lifecycle_livedata
    implementation deps.androidx.ktx_navigation_fragment
    implementation deps.androidx.ktx_navigation_ui
    implementation deps.androidx.constraint_layout

    implementation deps.retrofit.core
    implementation deps.retrofit.converter_gson

    implementation deps.okhttp.logging_interceptor

    implementation deps.room.core
    kapt deps.room.compiler
    implementation deps.room.ktx

    implementation deps.koin.core
    implementation deps.koin.ext
    implementation deps.koin.android
    implementation deps.koin.androidx_scope
    implementation deps.koin.androidx_viewmodel
    implementation deps.koin.androidx_fragment
    implementation deps.koin.androidx_ext

    implementation deps.stetho.core
    implementation deps.stetho.okhttp3_helper

    testImplementation deps.junit
    testImplementation deps.androidx_test.core
    testImplementation deps.room.testing
    testImplementation deps.koin.test

    androidTestImplementation deps.androidx_test.core
    androidTestImplementation deps.androidx_test.runner
    androidTestImplementation deps.androidx_test.rules
    androidTestImplementation deps.androidx_test.junit
    androidTestImplementation deps.androidx_test.truth
    androidTestImplementation deps.androidx_test.espresso_core
    androidTestImplementation deps.androidx_test.espresso_contrib
    androidTestImplementation deps.androidx_test.espresso_intents
    androidTestImplementation deps.androidx_test.uiautomator
}
```

### .travis.yml

```groovy
os: linux
language: android
android:
  components:
    - tools
    - platform-tools
    - build-tools-28.0.3
    - android-29
    - android-21
    - addon-google_apis-google-29
    - extra-google-m2repository
    - extra-android-m2repository
    - sys-img-armeabi-v7a-android-21
  licenses:
    - 'android-sdk-preview-license-52d11cd2'
    - 'android-sdk-license-.+'
    - 'google-gdk-license-.+'
before_install:
  - openssl aes-256-cbc -K $encrypted_5ad7d9eff25f_key -iv $encrypted_5ad7d9eff25f_iv -in han.keystore.enc -out han.keystore -d
  - chmod +x gradlew
before_script:
  - echo no | android create avd --force -n test -t android-21 --abi armeabi-v7a -c 100M
  - emulator -avd test -no-audio -no-window &
  - android-wait-for-emulator
  - adb shell settings put global window_animation_scale 0 &
  - adb shell settings put global transition_animation_scale 0 &
  - adb shell settings put global animator_duration_scale 0 &
  - adb shell input keyevent 82 &
script:
  - ./gradlew createDebugCoverageReport
  - ./gradlew assembleRelease
before_cache:
  - rm -f  $HOME/.gradle/caches/modules-2/modules-2.lock
  - rm -fr $HOME/.gradle/caches/*/plugin-resolution/
cache:
  directories:
    - $HOME/.gradle/caches/
    - $HOME/.gradle/wrapper/
    - $HOME/.android/build-cache
env:
  global:
    - DEPLOY_FILE=app/build/outputs/apk/release/app-release.apk
    - secure: jA+AJTv9pYnt76eYkLZaNBR2WfBH6ZdK/qxf8WxwlJ3GQ72TYFToDZgKr+ADWQAWDu6v5PdSJ9C4ORseTYa30ZxTZ4nN14+qsPrj3mVRbazD/ngQUPPeEtND+aDbcH+hLZoakrgp/aj5XznwUDoyujwOummHzk9xEPwNTtewNzzfcNBK6ToE5DBcTP2d1HC1YI7dod3SFgOkoVVLC9Wz+Jmdd9cQ2xPSZhvsXAbOE2Ctyh1e1jpWVI5l0ddgGmNx7l+KbHWDxQAk+DC4XEXIJpYsbpeowyEgqWRo4R4K1KDUqaGRZ+q/ic9Wd2mqbcDvsdneo5ZfrP6ILETku0GGjd4A7DsKSltujl/sHCyaXyTlzvPBc0TecyhKtyg/JXDVlev5uc+OW+5P7l/OVLeRE6Du5SJgoraanwTMnTdW8QcsdZgF7NMk+GZfeeZVnhJ7IDsekQzrRfk7cvikWqhklWRZXgSm/eoedBmfKamp9hRrbxs5gZwH+PD4yq1IT7Mhdc8Um2OAtOmbgQLhHXA84XS4XgjmuCz8TAJkYyiUBRuMTJGY8o6Q5XTzoDg/bnAvdovBjc+Y12vE3guLwegk2URPjYE113C9zEUQp9J4HWSNOVVpz4KoY7MY9sadUH892drK77Knx6oX073QV6AT6r1hJ/5K8+YoYX8fYVw1w1A=
    - secure: ZVnidBJHL4Si/hWln3tg9SR4masaddxWkzWeZ+rt825BOy1DzKLxx2CZxT2cz5Traojp1E2YMnKvZPIXzy+e5k9YdRs2gmN53guCBS14r49T8vMU+GZ/ypdmDnCxXBW412dDOCwgOXxeVCVa+lNKSkMYppE6cMNLUxCCDogwgaezzsZFWHfi2V20LZhqidYiMMtlxgQmSJRIPEH9ncwpDXjgIP6avnOOIaXUUN0acPB93I0zjJGr63BhcVDg0clCn9asjgIHz6LPR9b+y+WWviw+8ck+YE1HngFkMpjEyeK6OB01uYd8fxXiZUuDo57p6c1USfz8LoQsdcEDeL48Gme2ovcQ411gbLqhWVwbRrj7r0S8XUCkNtI0OnZkeWIZxYXF22bL1RkeTsZVDCeKQuIkgaP+5LQyy8j82i1xUFIVvHa+oLX+3TZt6/m0LjhjjJm5uBoSOLDa7ofJ8Pr7tlOzkQbXYPDOitB84ip6w9THl/G2GxUY6NpgFEQ6jKsKPfZNj/YQg9+TndILt95ywXLqfh3UB6jy8kvwBdo/LhQQ9Ax5DOs28adFSKmsKpbNDo1OtoEhehwwH/HfowhzuRUsarBqMY/qD7r7lwNFXT834hpg+5qFjnHQwNjzNhBo4OScdLY/aCrz7jswvnNK9h3l3I/l+fQclKoFxEdcCx8=
    - secure: ksz3MZiiqHboYr766JnS0ByTuOj5gvpCBQUajlxdLPo0D/bmlUvrrafAYjGtRYDgyfDhqPeBae69klB1n0sr10AqDhUnhLe2P88aTRrfytZAkdF3j01wRgcm+GIANIFtMHrypqyt8K1yulU8KoLcMgK45OUj+mzZXIZB1a2na0Xbxjaj6IpU+R/TEAozCvbZpixyrngauMJweiX/qbcK76g0Vj7RpEZWBq4NVjbnbsOI+HMIyKuawSSXUmxTjyhz4+CwcYGRLiLwY+6zcoLdsBUL7F+fl5yJpieX3vuMqeoIHRghVLaFrfZiN8Id6Fk3jHP1kq07hJUuAgYUarowYFGH+s1yhAppZ/7wX/qQFdnLlwKEdAjveXpKpKr+nLlU1oj9mZDD0W25JDV3537VH+qu1T8zq3sj+NO7d6dcq1JD6CviqqyblHJKMAhaz1O++oEbNEWatR4C2WY2X7KShuPctrwwoDpqd5Wkw7m/YOj0s4MZX80GUOBRVjbB8H8WjGrOGWhUn9hxPZHlVDUX5md13VTj4+L8vZ4T32/knCU0nbYxue/S3nr51P8XmjBNQwOlt/RooaWwdrlSdeLRICJwm49dytIJMRhiKInVfnszruCrB1pnGgsN7IP+vhL1ECfvwQe81RBUolTC2WD9Fi2U9aWovG06N6Ste3NyQbM=
    - secure: AjazliXHI0DOExUUhPAjSS6puK2Ic6b2+vFHUp9XqPp/j1PgOYjWSStUtNwxljgGYHYFJFYbnifOl2cOfcS8S7rUTEp8FqHeqMzIofijz+Gf9MDKcMbdcd3nEIq1FZOvXYxolwdfJNgUQRkBkVuro9ifMdsj2mmrIR8/W3HnHh1fgPmwA+DIuYC7Fimj1CRFYfpLKka6Q2jK0pR2PzZldlkP2SJvuSo3uiQC9Dm+fFo47C0C/I4c98MHQtvJzKHEbi3cHI0p6G8T6EaStqrn/kImpB/MyZFFq01A26YNaiJl50oCaUk/VZapf1jU3UJOS6i5VaGqWoBvY6wYH28grr9xGCzonijRGjSTPxdJDILXcdbgTighrDB2DXrzeT4Up7+iLiBINgFNN1Vl2otEcV4nJfuUsbNtJz17aEY8bkPCpb2UHm9hJLs+QvEAXZXxXH3TiWfxNppgltU+Fu0OTD4ANeBUREuZHm6pWfslMJYuOAUUATLTo4rzlfHW968HiM/TJpGWrPFAUXg3YruBCraP2x0LsC0+imvqtJgi54IPLjBwcOeWqzvJNTc+PkPnk7Dka642vmDl4WKtK67y8IzC0E5kt+9Ir6MPF1Hpa14BMvPpMQLKNE5X16J90tbQ/WG+HsfAyp/D88yowIQurL6e4gGJ5d31oI2LZOoFqHQ=
after_success:
  - bash <(curl -s https://codecov.io/bash)
before_deploy:
  - export DEPLOY_FILE=$(ls -1 app/build/outputs/apk/release/*.apk | sort -u | head -1)
deploy:
  provider: releases
  api_key:
    secure: n9nZknN6hw2Otn+g1LyOAQgCKEz5MpiHQdUe4VIjb0Rj9MfuwmcB5tAZZs+gDF47b+dY8sSdf0x5eXkCuQKmfrX2ha7NhEoitiMIvZ0Z41DYQtYeHdqotCtMFFSDBe9XtKIZWteNUVxr5d1fAovJ6D1rZNe3RrTN8WOhR2OLNg19fZsyXJCTwfp6QyfRNfuqn1xHwUjyIFsuhbIst14wg8u2oz910AhSHYo76XYCeGBdc5WdQWM0tTOgpqS38KJSys40+YDLtLHRZ77xiRkoRi7NePcKjmJ7MR9ySxGSnw8Vjz2YjTjMpQ6R2feR/ZXe6kbNmqLVof0DlqQ+TbpS1SgWdwRAi33PUQ5sW16MMkoxHiFetVNOA3S2cq4GTDfxs0uTpggp5XAuq455/zD5/Pd0sRQ3Vrl2pNk9rVb9SdpMN1Yva0uzQ1UsbTa/Jv/kq4f2D5u09iZy5FLjGdtxdAdMfebiVGAy640gW7MIej1dFGTeAz8Tx77F/4hzHD05wqKDKt1XASSoJAZmzyu+AdaA7T+FgSRU3AmMosAe5VL0ph0BR+A0EB4t8gQ/WQIz+5kDKPOUH3qPwewYLYvWDtNHLNc3q8oz6QuwhWPxeFw2D6cViOzaDdLl+mHColMaPKOSs6nz5KAPB3fzLnH1a2UDaZ2vgM/yRv9Xmifg2Sc=
  file: $DEPLOY_FILE
  skip_cleanup: true
  on:
    repo: shangmingchao/Han
    tags: true
```

### version.properties

```text
versionMajor = 1
versionMinor = 0
versionPatch = 0
versionBuild = 0
```

### checkstyle.xml

```xml
<code_scheme name="google"
    version="173">
    <option name="FIELD_NAME_PREFIX"
        value="m" />
    <option name="STATIC_FIELD_NAME_PREFIX"
        value="s" />
    <option name="INSERT_INNER_CLASS_IMPORTS"
        value="true" />
    <option name="CLASS_COUNT_TO_USE_IMPORT_ON_DEMAND"
        value="9999" />
    <option name="NAMES_COUNT_TO_USE_IMPORT_ON_DEMAND"
        value="9999" />
    <option name="IMPORT_LAYOUT_TABLE">
        <value>
            <package name="android"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="com.android"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="dalvik"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="libcore"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="com"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="gov"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="junit"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="net"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="org"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="java"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="javax"
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name=""
                withSubpackages="true"
                static="true" />
            <emptyLine />
            <package name="android"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="com.android"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="dalvik"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="libcore"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="com"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="gov"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="junit"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="net"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="org"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="java"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name="javax"
                withSubpackages="true"
                static="false" />
            <emptyLine />
            <package name=""
                withSubpackages="true"
                static="false" />
        </value>
    </option>
    <option name="RIGHT_MARGIN"
        value="100" />
    <option name="JD_P_AT_EMPTY_LINES"
        value="false" />
    <option name="JD_DO_NOT_WRAP_ONE_LINE_COMMENTS"
        value="true" />
    <option name="JD_KEEP_EMPTY_PARAMETER"
        value="false" />
    <option name="JD_KEEP_EMPTY_EXCEPTION"
        value="false" />
    <option name="JD_KEEP_EMPTY_RETURN"
        value="false" />
    <option name="JD_PRESERVE_LINE_FEEDS"
        value="true" />
    <option name="KEEP_CONTROL_STATEMENT_IN_ONE_LINE"
        value="false" />
    <option name="KEEP_BLANK_LINES_IN_CODE"
        value="1" />
    <option name="BLANK_LINES_AROUND_FIELD"
        value="1" />
    <option name="BLANK_LINES_AFTER_CLASS_HEADER"
        value="1" />
    <option name="ALIGN_MULTILINE_PARAMETERS"
        value="false" />
    <option name="ALIGN_MULTILINE_FOR"
        value="false" />
    <option name="CALL_PARAMETERS_WRAP"
        value="1" />
    <option name="METHOD_PARAMETERS_WRAP"
        value="1" />
    <option name="EXTENDS_LIST_WRAP"
        value="1" />
    <option name="THROWS_LIST_WRAP"
        value="1" />
    <option name="EXTENDS_KEYWORD_WRAP"
        value="1" />
    <option name="THROWS_KEYWORD_WRAP"
        value="1" />
    <option name="METHOD_CALL_CHAIN_WRAP"
        value="1" />
    <option name="BINARY_OPERATION_WRAP"
        value="1" />
    <option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE"
        value="true" />
    <option name="TERNARY_OPERATION_WRAP"
        value="1" />
    <option name="TERNARY_OPERATION_SIGNS_ON_NEXT_LINE"
        value="true" />
    <option name="FOR_STATEMENT_WRAP"
        value="1" />
    <option name="ARRAY_INITIALIZER_WRAP"
        value="1" />
    <option name="ASSIGNMENT_WRAP"
        value="1" />
    <option name="PLACE_ASSIGNMENT_SIGN_ON_NEXT_LINE"
        value="true" />
    <option name="WRAP_COMMENTS"
        value="true" />
    <option name="IF_BRACE_FORCE"
        value="3" />
    <option name="DOWHILE_BRACE_FORCE"
        value="3" />
    <option name="WHILE_BRACE_FORCE"
        value="3" />
    <option name="FOR_BRACE_FORCE"
        value="3" />
    <AndroidXmlCodeStyleSettings>
        <option name="USE_CUSTOM_SETTINGS"
            value="true" />
    </AndroidXmlCodeStyleSettings>
    <JavaCodeStyleSettings>
        <option name="FIELD_NAME_PREFIX"
            value="m" />
        <option name="STATIC_FIELD_NAME_PREFIX"
            value="s" />
        <option name="ANNOTATION_PARAMETER_WRAP"
            value="1" />
        <option name="INSERT_INNER_CLASS_IMPORTS"
            value="true" />
        <option name="CLASS_COUNT_TO_USE_IMPORT_ON_DEMAND"
            value="9999" />
        <option name="NAMES_COUNT_TO_USE_IMPORT_ON_DEMAND"
            value="9999" />
        <option name="IMPORT_LAYOUT_TABLE">
            <value>
                <package name="android"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="com.android"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="dalvik"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="libcore"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="com"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="gov"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="junit"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="net"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="org"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="java"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="javax"
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name=""
                    withSubpackages="true"
                    static="true" />
                <emptyLine />
                <package name="android"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="com.android"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="dalvik"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="libcore"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="com"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="gov"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="junit"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="net"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="org"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="java"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name="javax"
                    withSubpackages="true"
                    static="false" />
                <emptyLine />
                <package name=""
                    withSubpackages="true"
                    static="false" />
            </value>
        </option>
        <option name="JD_P_AT_EMPTY_LINES"
            value="false" />
        <option name="JD_DO_NOT_WRAP_ONE_LINE_COMMENTS"
            value="true" />
        <option name="JD_KEEP_EMPTY_PARAMETER"
            value="false" />
        <option name="JD_KEEP_EMPTY_EXCEPTION"
            value="false" />
        <option name="JD_KEEP_EMPTY_RETURN"
            value="false" />
        <option name="JD_PRESERVE_LINE_FEEDS"
            value="true" />
    </JavaCodeStyleSettings>
    <JetCodeStyleSettings>
        <option name="PACKAGES_TO_USE_STAR_IMPORTS">
            <value />
        </option>
        <option name="NAME_COUNT_TO_USE_STAR_IMPORT"
            value="99" />
        <option name="NAME_COUNT_TO_USE_STAR_IMPORT_FOR_MEMBERS"
            value="99" />
        <option name="IMPORT_NESTED_CLASSES"
            value="true" />
        <option name="CONTINUATION_INDENT_IN_PARAMETER_LISTS"
            value="false" />
        <option name="CONTINUATION_INDENT_IN_ARGUMENT_LISTS"
            value="false" />
        <option name="CONTINUATION_INDENT_FOR_EXPRESSION_BODIES"
            value="false" />
        <option name="CONTINUATION_INDENT_FOR_CHAINED_CALLS"
            value="false" />
        <option name="CONTINUATION_INDENT_IN_SUPERTYPE_LISTS"
            value="false" />
        <option name="CONTINUATION_INDENT_IN_IF_CONDITIONS"
            value="false" />
        <option name="WRAP_EXPRESSION_BODY_FUNCTIONS"
            value="1" />
        <option name="IF_RPAREN_ON_NEW_LINE"
            value="true" />
    </JetCodeStyleSettings>
    <Objective-C-extensions>
        <file>
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Import" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Macro" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Typedef" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Enum" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Constant" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Global" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Struct" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="FunctionPredecl" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Function" />
        </file>
        <class>
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Property" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="Synthesize" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="InitMethod" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="StaticMethod" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="InstanceMethod" />
            <option name="com.jetbrains.cidr.lang.util.OCDeclarationKind"
                value="DeallocMethod" />
        </class>
        <extensions>
            <pair source="cpp"
                header="h"
                fileNamingConvention="NONE" />
            <pair source="c"
                header="h"
                fileNamingConvention="NONE" />
        </extensions>
    </Objective-C-extensions>
    <Properties>
        <option name="KEEP_BLANK_LINES"
            value="true" />
    </Properties>
    <XML>
        <option name="XML_ATTRIBUTE_WRAP"
            value="2" />
        <option name="XML_ALIGN_ATTRIBUTES"
            value="false" />
        <option name="XML_SPACE_INSIDE_EMPTY_TAG"
            value="true" />
        <option name="XML_LEGACY_SETTINGS_IMPORTED"
            value="true" />
    </XML>
    <ADDITIONAL_INDENT_OPTIONS fileType="java">
        <option name="TAB_SIZE"
            value="8" />
    </ADDITIONAL_INDENT_OPTIONS>
    <ADDITIONAL_INDENT_OPTIONS fileType="js">
        <option name="CONTINUATION_INDENT_SIZE"
            value="4" />
    </ADDITIONAL_INDENT_OPTIONS>
    <codeStyleSettings language="JAVA">
        <option name="ALIGN_MULTILINE_PARAMETERS"
            value="false" />
        <option name="ALIGN_MULTILINE_FOR"
            value="false" />
        <option name="CALL_PARAMETERS_WRAP"
            value="1" />
        <option name="PREFER_PARAMETERS_WRAP"
            value="true" />
        <option name="METHOD_PARAMETERS_WRAP"
            value="1" />
        <option name="RESOURCE_LIST_WRAP"
            value="1" />
        <option name="EXTENDS_LIST_WRAP"
            value="1" />
        <option name="THROWS_LIST_WRAP"
            value="1" />
        <option name="EXTENDS_KEYWORD_WRAP"
            value="1" />
        <option name="THROWS_KEYWORD_WRAP"
            value="1" />
        <option name="METHOD_CALL_CHAIN_WRAP"
            value="1" />
        <option name="BINARY_OPERATION_WRAP"
            value="1" />
        <option name="BINARY_OPERATION_SIGN_ON_NEXT_LINE"
            value="true" />
        <option name="TERNARY_OPERATION_WRAP"
            value="1" />
        <option name="TERNARY_OPERATION_SIGNS_ON_NEXT_LINE"
            value="true" />
        <option name="FOR_STATEMENT_WRAP"
            value="1" />
        <option name="ARRAY_INITIALIZER_WRAP"
            value="1" />
        <option name="ASSIGNMENT_WRAP"
            value="1" />
        <option name="IF_BRACE_FORCE"
            value="3" />
        <option name="DOWHILE_BRACE_FORCE"
            value="3" />
        <option name="WHILE_BRACE_FORCE"
            value="3" />
        <option name="FOR_BRACE_FORCE"
            value="3" />
        <option name="WRAP_LONG_LINES"
            value="true" />
        <option name="PARAMETER_ANNOTATION_WRAP"
            value="1" />
        <option name="VARIABLE_ANNOTATION_WRAP"
            value="1" />
        <option name="ENUM_CONSTANTS_WRAP"
            value="1" />
    </codeStyleSettings>
    <codeStyleSettings language="JSON">
        <indentOptions>
            <option name="CONTINUATION_INDENT_SIZE"
                value="4" />
            <option name="TAB_SIZE"
                value="2" />
        </indentOptions>
    </codeStyleSettings>
    <codeStyleSettings language="XML">
        <option name="FORCE_REARRANGE_MODE"
            value="1" />
        <indentOptions>
            <option name="CONTINUATION_INDENT_SIZE"
                value="4" />
        </indentOptions>
        <arrangement>
            <rules>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>xmlns:android</NAME>
                                <XML_NAMESPACE>^$</XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>xmlns:.*</NAME>
                                <XML_NAMESPACE>^$</XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:id</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:name</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>name</NAME>
                                <XML_NAMESPACE>^$</XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>style</NAME>
                                <XML_NAMESPACE>^$</XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*</NAME>
                                <XML_NAMESPACE>^$</XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:layout_width</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:layout_height</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:layout_.*</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:width</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:height</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:viewportWidth</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:viewportHeight</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res/android
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:layout_.*</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res-auto
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*</NAME>
                                <XML_NAMESPACE>http://schemas.android.com/apk/res-auto
                                </XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*:layout_.*</NAME>
                                <XML_NAMESPACE>.*</XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
                <section>
                    <rule>
                        <match>
                            <AND>
                                <NAME>.*</NAME>
                                <XML_NAMESPACE>.*</XML_NAMESPACE>
                            </AND>
                        </match>
                        <order>BY_NAME</order>
                    </rule>
                </section>
            </rules>
        </arrangement>
    </codeStyleSettings>
    <codeStyleSettings language="kotlin">
        <option name="KEEP_BLANK_LINES_IN_DECLARATIONS"
            value="1" />
        <option name="KEEP_BLANK_LINES_IN_CODE"
            value="1" />
        <option name="KEEP_BLANK_LINES_BEFORE_RBRACE"
            value="0" />
        <option name="ALIGN_MULTILINE_PARAMETERS"
            value="false" />
        <option name="CALL_PARAMETERS_WRAP"
            value="1" />
        <option name="CALL_PARAMETERS_LPAREN_ON_NEXT_LINE"
            value="true" />
        <option name="CALL_PARAMETERS_RPAREN_ON_NEXT_LINE"
            value="true" />
        <option name="METHOD_PARAMETERS_WRAP"
            value="5" />
        <option name="METHOD_PARAMETERS_LPAREN_ON_NEXT_LINE"
            value="true" />
        <option name="METHOD_PARAMETERS_RPAREN_ON_NEXT_LINE"
            value="true" />
        <option name="EXTENDS_LIST_WRAP"
            value="1" />
        <option name="METHOD_CALL_CHAIN_WRAP"
            value="1" />
        <option name="ASSIGNMENT_WRAP"
            value="1" />
        <option name="FIELD_ANNOTATION_WRAP"
            value="1" />
        <option name="PARAMETER_ANNOTATION_WRAP"
            value="1" />
        <option name="VARIABLE_ANNOTATION_WRAP"
            value="1" />
        <option name="ENUM_CONSTANTS_WRAP"
            value="5" />
        <indentOptions>
            <option name="CONTINUATION_INDENT_SIZE"
                value="4" />
        </indentOptions>
    </codeStyleSettings>
</code_scheme>
```
