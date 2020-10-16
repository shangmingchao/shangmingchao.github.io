# Android 开发常用命令

## ADB

```shell
adb devices
adb -s emulator-5554 install a.apk
adb push file /sdcard/
adb shell am start com.a/com.a.MainActivity -e name username
adb shell am kill com.a
```
