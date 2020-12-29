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
Editor -> Code Style -> Kotlin: 点击 `Set from...` 选择 `Kotlin style guide`。`Blank Lines` 从上到下设置为 1 1 0 0 0 1。`Imports` 勾选中 使用单命名引入  
Editor -> File and Code Templates -> Includes -> File Header: 设置为:  

```java
/**
 * 
 * 
 * @author shangmingchao
 */
```

Editor -> Layout Editor -> Default Editor Mode: 全部设置为 `Split`  
JSON 插件可以搜索 `JSON To Kotlin Class ​(JsonToKotlinClass)`  
macOS Big Sur 11.1 操作系统可以运行 `defaults write com.google.android.studio AppleWindowTabbingMode manual` 命令以禁止 Android Studio 以标签页的方式打开新窗口  

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
- ==========================
- 切换输入法: `control ;`
- 复制文件路径: `command option C`
- 显示隐藏文件: `command shift .`

## Git

### 基础

- `HEAD` 指针指向当前所在的本地分支
- 创建分支 `git branch testing`，但不会自动切换到新创建的分支
- 切换到给定的分支 `git checkout testing`
- 创建分支后自动切到新的分支 `git checkout -b testing`
- 删除分支 `git branch -d testing`

### 合并分支

#### merge

本地提交了 C3 和 C5，同时远程 master 提交了 C4

![basic_merging_1](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/basic_merging_1.png)  
此时，需要把本地的 C3 和 C5 提交到远程 master:  

```shell
git checkout master
git merge iss53
```

![basic_merging_2](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/basic_merging_2.png)  

也就是说，这种情况，Git 会根据 C4，C5 和它们的共同祖先 C2 做一次合并，会出现一个 **C6 合并提交**。如果合并的过程中出现冲突，Git 会暂停下来，等待手动解决冲突。可以使用 `git status` 查看冲突的文件，解决完冲突后使用 `git add` 命令对每个文件标记为冲突已解决。很多时候手动删除冲突文件中的 `<<<<<<<`，`=======`，`>>>>>>>` 标记既繁琐又不直观，可以使用一些可视化工具完成（`git mergetool` 命令启动）。解决完冲突后就可以使用 `git commit` 命令完成合并提交了  

> 远程仓库的默认名字是 `origin`，`origin/master` 指向的就是上次拉取远程仓库代码时的位置，如果要同步本地仓库和远程仓库可以使用 `git fetch origin`  
![remote_branches_1](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/remote_branches_1.png)  
![remote_branches_2](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/remote_branches_2.png)  
`git fetch` 只会更新本地 `origin/master`，对工作区 `master` 没有影响，你可以之后手动 `merge` 这两者。而另一个魔法命令 `git pull` 会在更新本地 `origin/master` 时同时更新工作区 `master`，它很像 fetch + merge  

#### rebase

![basic_rebase_1](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/basic_rebase_1.png)  
可以提取 C4 的更改，然后在 C3 基础上应用一次。在 Git 中这种操作被称为 rebase 变基  

```shell
git checkout experiment
git rebase master
```

![basic_rebase_2](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/basic_rebase_2.png)  
原理就是把 `experiment` 相应的修改存为临时文件，然后指向目标 `master` 基底 C3，最后将临时文件的修改依次应用  
然后就可以回到 `master` 进行一次快速合并了  

```shell
git checkout master
git merge experiment
```

![basic_rebase_3](https://raw.githubusercontent.com/shangmingchao/shangmingchao.github.io/master/images/basic_rebase_3.png)  

#### merge 和 rebase

rebase 和 merge 虽然最终结果没有区别。rebase 的提交是一条直线，更加清晰整洁，但是它改变了历史，让一个分支的历史在目标分支上重演了一遍，所以它一定程度上亵渎了历史掩盖了真相。merge 的提交记录了实际上发生了什么，但是它让整个提交历史看起来错综复杂，而且很多 merge 没法直观地表明项目过程中发生了什么  
所以，不管是使用 merge 也好还是使用 rebase 也好，都没有对错，根据自己的喜好、团队的决策、具体的场景选择一种就行了  

> 不要对别人已经提交到远程仓库中的一些提交进行 rebase，否则整个仓库和团队中的所有人都可能陷入混乱。如果确实发生了这件事，尽快通知团队所有人使用 `git pull --rebase` 以缓解一些伤痛  

### 其它

- 查看分支列表 `git branch`
- 查看各个分支详情 `git log --oneline --decorate --graph --all`
- 删除远程分支 `git push origin --delete serverfix`
