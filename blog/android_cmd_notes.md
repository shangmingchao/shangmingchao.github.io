# Android 开发常用命令

## ADB

```shell
adb devices
adb -s emulator-5554 install a.apk
adb push file /sdcard/
adb shell am start com.a/com.a.MainActivity -e name username
adb shell am kill com.a
```

## Android Studio

Keymap 快捷键使用 `macOS`  
Editor -> General -> Save Files: 勾选中 裁剪末尾空格 和 确保文件末尾有换行  
Editor -> General -> Auto Import: Java 和 Kotlin 都勾选中 打字时优化包的引入   
Editor -> Code Style -> Scheme: 使用 `Default`  
Editor -> Code Style -> Kotlin: 点击 `Set from...` 选择 `Kotlin style guide`，`Blank Lines` 从上到下设置为 2 2 0 0 0 1  
Editor -> File and Code Templates -> Includes -> File Header: 设置为:  

```java
/**
 * 
 * 
 * @author shangmingchao
 */
```
Editor -> Layout Editor -> Default Editor Mode: 设置为 `Split`  

JSON 插件可以搜索 `JSON To Kotlin Class ​(JsonToKotlinClass)`  
快捷键:  

- **向前向后导航**: `command [`, `command ]`
- **回到上次编辑的位置**: `command shift delete`
- 跳转到某一行: `command L`
- 查看最近文件: `command E`
- 弹出补全提示: `control space`, `control shift space`
- **自动补全语句**: `command shift enter`
- 查看参数提示: `command P`
- 实现方法: `control I`
- 覆写方法: `control O`
- 格式化代码: `command option L`
- 删除一行: `command delete`
- 复制一行: `command D`
- 书写下一行: `shift enter`
- **选择当前语句块**: `option up up up`
- 环绕语句: `command option T`
- 优化引入: `control option O`
- 切换大小写: `command shift U`
- **查看定义**: `command Y`

- 切换输入法: `control ;`
- 复制文件路径: `command option C`
- 显示隐藏文件: `command shift .`

