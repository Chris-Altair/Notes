[TOC]

### 1. git基本操作

```bash
#添加到暂存区
git add <change_file>
#提交暂存区内容到本地
git commit -m 'msg'
#添加到暂存区并提交到本地仓库
git commit -a -m 'msg'

#撤销本地commit
git reset --soft HEAD~1 # –-soft只回退commit不回退修改；1表示回退1次commit
```

### 2. git分支操作

**pull和fetch区别**：git pull是拉取远程分支更新到本地仓库的操作。事实上，git pull是相当于从远程仓库获取最新版本，然后再与本地分支merge（合并）。即：***git pull = git fetch + git merge***

> 注：git fetch不会进行合并，执行后需要手动执行git merge合并，而git pull拉取远程分之后直接与本地分支进行合并。更准确地说，git pull是使用给定的参数运行git fetch，并调用git merge将检索到的分支头合并到当前分支中。

```bash
-------------------------分支操作------------------------
#查看本地分支，并显示当前分支
git branch
#查看远程分支
git branch -r
#创建本地分支，注意不会切换
git branch <branchname>
#切换分支
git checkout <branchname>
#已当前分支为基准，创建并切换分支
git chekcout -b <branchname>

-------------------------切换分支暂存修改（非commit方式）操作------------------------
#1.暂存本地修改，回到上一个commit，注意这种方式是独立于分支的
git stash
#2.可切到其他分支修改，之后切回原来的分支
#3.恢复本地修改
git stash pop

-------------------------远程分支操作------------------------
#拉取远程分支并创建本地分支
git checkout -b <branchname> origin/<remote_branchname>

#本地新建分支并下载远程分支到本地分支
git fetch origin <remote_branchname>:<local_branchname>
#拉取远程分支并与本地分支merge,即git pull = git fetch + git merge
git pull origin <remote_branchname>:<local_branchname>
#将本地的分支版本上传到远程并合并,若本地分支和远程分支名相同，可省略:,origin其实是主机名
git push -u origin <local_branchname>:<remote_branchname>

#比较本地分支与远程分支差异
git diff <local_branchname> <remote>/<remote_branchname>
-------------------------合并分支------------------------
#将本地branch分支合并到当前分支current_branch，合并后需要push当前分支到远程分支，才能将远程分支也合并
git checkout <current_branchname>
git merge <branchname>
git push origin <current_branchname>

-------------------------删除分支------------------------
#1.删除本地分支
git branch -d <branchname>
#2.删除远程分支,主机名也可放到--delete后
git push origin --delete <remote_branchname>

-------------------------重命名分支------------------------
#1.删除远程分支xxx
git push --delete origin xxx
#2.重命名本地分支xxx->yyy
git branch -m xxx yyy
#3.推送分支到远程
git push origin yyy
```

### 3. https切换git提交

通过密钥认证，可解决https每次push需要都输入username和password的问题

本质是修改.git路径下的config中的url

```bash
#不是必须（貌似），本质是修改.git/config文件的user属性
git config user.username '<username>'
git config user.email '<email>'

#1.查看远程push及fetch方式，会显示https或git
git remote -v
#2.生成密钥，会在用户路径生成 .ssh/id_rsa 和 .ssh/id_rsa.pub
ssh-keygen -t rsa -C "<email>" #中间输入回车即可
#3.github的settings->ssh key中添加.ssh/id_rsa.pub的全部内容
#4.本地仓库修改连接方式,本质是修改.git/config中的url属性
git remote rm origin
git remote add origin git@github.com:<username>/<repo>.git
git push -u origin main
#5.测试切换是否成功
ssh -T git@github.com
```

