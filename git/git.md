[TOC]



##### **禁用ssl认证**

```shell
#有公司的私有仓库的权限，但是go mod总是无法获取私有仓库，需要禁用ssl认证
git config --global http.sslVerify false

#PS：如果go.mod文件中包含有公司的其他私有仓库，而且我没有权限，但是它已经被包含在govendor中。那么就可以在goland：File--Settings--Go--Go Modules中，取消enable go modules integration的打勾。并把项目放在$GOPATH下。

#再或者，vendor目录下已经有了这个私有库，那么可以手动把go.mod文件中的那一行注释掉。
```

##### 查看branch与tag

```shell
#获取所有分支
git branch -a
#获取本地分支
git branch
#本地分支之间进行切换
git checkout <branchName>

#获取所有tag
git tag
```

##### 基于某个tag进行开发

```shell
git clone <registryName>
#查看远端所有发布的tag
git tag
#获得指定tag的代码，但是此时只能看，不能修改提交
git checkout tag
#本地新建一个分支
git checkout -b <newBranchName>
#拉取时，提示你：需要绑定。按照提示操作后，本地的分支就和远端的tag关联了。就可以基于这个tag进行开发了。
git pull
```

##### 基于某个branch进行开发

```shell
#本地会新建一个同名分支，并且自动跟踪远程的那个分支
git checkout --track origin/<branchName>
```

##### 一次完整的提交合并请求

```shell
使用beyondCompare+复制项目的模式。

#拉取最新代码
git checkout master
git pull

#切换至本地我的分支
git checkout glnie
#beyondCompare比较合并我的新代码
git add .
git commit -m "comment"

#再切换master，拉取最新代码，保证master是最新的
git checkout master
git pull

git checkout glnie
#1.master在我上次合并之后，到这次合并之前，没有被修改过。那么无论这次修改的变化有多大，都不会冲突。
#  master在我上次合并之后，到这次合并之前，别的同事也有提交，但他们没有修改我这次提交的文件，也不会冲突。
#2.相反，我这次提交的文件如果被修改了（这是非常常见的现象），即使你有意识把同事的代码全量复制进我的提交，只保留我的业务实现，也会冲突！
#3.所以，必须执行这一步本地合并！提示哪里有冲突，就解决哪里。
git merge master
#解决冲突后一定要再add，commit。
git add .
git commit -m "comment"

#push到远程的我的分支，然后就可以在图形界面提交合并请求了
git push origin glnie:glnie
```

```shell
直接在本地仓库目录中开发，提交。

#开发自己的代码，完成后，提交
git add .
git commit -m "comment"

#拉取远程代码，可能会冲突
git pull

#有冲突就解决冲突，然后再提交
git add .
git commit -m "comment"
git push origin glnie:glnie
```

git submodule update --init



### git config

git config --global --list

git config --global user.name xxx

git config --global user.email yyy@y.cm

这里配置的用户和用户邮箱，仅仅起到标识用户的作用。与登录代码托管平台的账户没有关系。只是多数情况下，它俩是一致的。

另外，配置公私钥的时候，使用的是ssh命令行，与git命令行无关。



### git add

工作区添加到暂存区。

### git rm --cached \<file>

把add添加到暂存区的**文件**，撤销到工作区。

### git checkout -- \<file>

把add添加到暂存区的**文件内容**，撤销到工作区。

与git add互逆。



### git commit

暂存区提交到本地仓库。

### git reset

把commit到本地仓库的内容，撤销到暂存区，最常用的--hard选项支持：同时修改暂存区和工作区。

与git commit互逆。

#### git reset --hard 哈希值

最推荐使用这个。

#### git reset --hard HEAD~n

#### git reset --hard HEAD若干个^

只能后退。



### git log

多屏显示控制方式：

空格向下翻页；b向上翻页；q退出。

#### git log --pretty=oneline

相对于log，单行显示哈希，message，简洁一些。

#### git log --oneline

相对于--pretty，哈希值只显示一小部分。

#### git reflog

它会提示：移动到该版本，需要几步。



### git diff

不带文件名比较多个文件

#### git diff [文件名]

将工作区中的文件和暂存区进行比较

#### git diff [本地库中历史版本] [文件名]

将工作区中的文件和本地库历史记录比较









