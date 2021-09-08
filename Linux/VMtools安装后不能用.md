# VMtools安装后不能用

安装VMware tools之后从windows复制文件到ubuntu发现没有成功

重新执行vmware-uninstall-tools.pl脚本提示之前已经安装了版本，需要卸载重装

解决方案：

```
1）不需要卸载

2）命令行执行sudo apt-get install open-vm-tools-desktop

3）可能会提示apt-get update或者 --fix-missing

4）然后命令行执行 apt-get update --fix-missing

5）命令行再执行sudo apt-get install open-vm-tools-desktop

6）重启后可以使用
```

