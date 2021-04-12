[TOC]

### 1. git基本操作

```bash
#添加到暂存区
git add <change_file>
#提交暂存区内容到本地
git commit -m 'msg'
#添加到暂存区并提交到本地仓库
git commit -a -m 'msg'
```

### 2. git分支操作

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
#创建并切换分支

git chekcout -b <branchname>
#本地新建分支并下载远程分支到本地分支
git fetch origin <remote_branchname>:<local_branchname>
#将本地的分支版本上传到远程并合并,若本地分支和远程分支名相同，可省略:,origin其实是主机名
git push -u origin <local_branchname>:<remote_branchname>
-------------------------合并分支------------------------
#将本地<branchname>分支合并到当前分支，合并后需要push当前分支到远程分支，才能将远程分支也合并
git merge <branchname>
-------------------------删除分支------------------------
#删除本地分支
git branch -d <branchname>
#删除远程分支
git push origin --delete <remote_branchname>
```

### 3. https切换git提交

通过密钥认证，可解决https每次push需要输入username和password的问题

```bash
#不是必须(猜测)
git config user.username '<username>'
git config user.email '<email>'

#1.查看远程push及fetch方式，会显示https或git
git remote -v
#2.生成密钥，会在用户路径生成 .ssh/id_rsa 和 .ssh/id_rsa.pub
ssh-keygen -t rsa -C "<email>" #中间输入回车即可
#3.github的settings->ssh key中添加.ssh/id_rsa.pub的全部内容
#4.本地仓库修改连接方式
git remote rm origin
git remote add origin git@github.com:<username>/<repo>.git
git push -u origin main
```

