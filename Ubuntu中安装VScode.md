# Ubuntu中安装VScode

## 1、下载VScode的安装包

我安装的是1.58.2，如果需要安装我的这个版本可以在我分享的链接中下载：

>我用阿里云盘分享了「VScode」，你可以不限速下载🚀 复制这段内容打开「阿里云盘」App 即可获取 链接：https://www.aliyundrive.com/s/NXQzzZcQYDp

## 2、打开命令行输入以下命令

```markdown
dpkg -i 安装包名字
```

## 3、安装一些必要的插件

![在这里插入图片描述](http://qyateyap7.hn-bkt.clouddn.com/img/2021052116422683.png)

常用的插件有：
	1)C/C++，这个肯定是必须的。
	2)C/C++ Snippets，即 C/C++重用代码块。
	3)C/C++ Advanced Lint,即 C/C++静态检测 。 4)、Code Runner，即代码运行。
	5)Include AutoComplete，即自动头文件包含。
	6)Rainbow Brackets，彩虹花括号，有助于阅读代码。
	7)One Dark Pro，VSCode 的主题。
	8)GBKtoUTF8，将 GBK 转换为 UTF8。 9)、ARM，即支持 ARM 汇编语法高亮显示。
	10)Chinese(Simplified)，即中文环境。
	11)vscode-icons，VSCode 图标插件，主要是资源管理器下各个文件夹的图标。
	12)compareit，比较插件，可以用于比较两个文件的差异。
	13)DeviceTree，设备树语法插件。
	14)TabNine，一款 AI 自动补全插件，强烈推荐。
	15)如果需要某些语法错误提示，比如提示你敲错某个变量名、头文件错误等，要安装：c/c++ clang command adapter。安装完这个插件后可能会提示你“”Please install clang or check configuration clang.executable”的错误 ，解决办法：安装Clang。安装完Clang后重启VScode，这个时候你敲代码VScode就能给你语法检查，但是我们linux的话一般是不在VScode编译代码的，仅仅用作编辑器，如果你包含某个头文件如<stdio.h>它会一直提醒你找不到这个头文件，不影响你代码的编写但是这会让你很烦，这个时候需要给clang添加如下配置，如下图，记住VScode的设置可以通过UI或者json两种方式进行设置，数字2那步就是用于切换这两种方式的，json用得比较多。

![img](http://qyateyap7.hn-bkt.clouddn.com/img/20210521172627855.png)

