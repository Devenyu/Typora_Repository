# Windows10子系统WSL修改默认安装目录到其他盘

## 1、查看WSL分发版本

```markdown
wsl -l --all  -v
```

## 2、导出分发版为tar文件到D盘

```markdown
wsl --export Ubuntu-20.04 d:\wsl-ubuntu20.04.tar
```

## 3、注销当前分发版

```markdown
wsl --unregister Ubuntu-20.04
```

## 4、重新导入并安装WSL在D:\wsl-ubuntu20.04

```markdown
wsl --import Ubuntu-20.04 d:\wsl-ubuntu20.04 d:\wsl-ubuntu20.04.tar --version 2
```

## 5、设置默认登录用户为安装时用户名

```markdown
ubuntu2004 config --default-user Username
```

## 6.删除tar文件(可选)

```markdown
del d:\wsl-ubuntu20.04.tar
```

