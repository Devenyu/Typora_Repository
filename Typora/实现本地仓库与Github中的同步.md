# 实现本地仓库与Github中的同步

## 一、在Github中新建一个仓库

![image-20210908153010523](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210908153010523.png)

## 二、建立本地仓库

- 在对应的项目目录中打开Git Bush
- 输入` git init` 并执行
- 此时我们就成功新建了一个空的本地仓库，同时在目录下也会新增一个隐藏格式的.git的文件

## 三、关联

- 在目录的命令行工具中执行 `git remote add origin git@github.com:[githubID]/[repositoryName].git`, 关联过程中会要求你输入账号密码, 当然如果已经与本地计算机建立了 **SSH**协议就不用了
- 关联成功的话，我们可以通过 `git remote -v` 进行查看

## 四、将本地文件同步到远程仓库中

- 在目录的命令行工具中依此执行`git add .`
- `git commit -m "some description"`
- `git push -u origin master`

## 五、将远程文件同步到本地

- 当我们使用不同的电脑进行编辑时候, 这时候就需要保持不同电脑本地文件保持统一, 这时候, 首先我们需要做的是在新的电脑上**拷贝远程仓库至本地**, 执行`git clone [remoteRepoURL] [localRepoName]`

![image-20200603224738635](http://qyateyap7.hn-bkt.clouddn.com/img/20200603224753.png)

- 后续, 当我们需要在不同的本地进行文档书写时, 每次只需要在对应的目录中执行`git pull`同步最新的远程仓库文件就OK了

>遇到如下问题：
>
>fatal: Could not read from remote repository.（致命：不能读远端仓库。）
>
>![img](http://qyateyap7.hn-bkt.clouddn.com/img/1297117-20190529214018754-27174754.png)
>
>**A**. 如果你也是初次使用Github，且没有设置过ssh key，请按照如下：
>
>　　 　　1.键入：ssh-keygen -t rsa -C "你注册Github的邮箱" ，然后一路"Enter”
>
>　　　　 2.打开：C:\Users\你的用户名/.ssh/id_rsa.pub
>
>　　　　 3.复制：Notepad++打开上述文件，全部复制
>
>**B**. 在Github的个人设置中添加SSH公钥
>
>**C.** 再次键入“git push -u origin master”，若出现如下，git pull, 就OK了

