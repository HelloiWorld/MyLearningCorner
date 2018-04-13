## 安装git
1. 安装homebrew

		/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)
	
2. 通过homebrew安装git：`brew install git`

## 基本命令
* `git init`：初始化一个git仓库
* `git add <file>`/`git add .`： 添加指定/所有修改过的文件到暂存区(Stage)
* `git commit -m '<description>'`：将暂存区所有修改提交到分支
* `git status`：查看当前工作区状态
* `git diff <file>`：查看修改内容
* `git log`：查看提交历史（显示从最近到最远的提交日志）
  - `git log --graph`：查看分支合并图
  - `git log --pretty=oneline`：查看commit id和简短的日志信息
  - `git log --pretty=oneline --abbrev-commit`：查看精简的commit id和日志信息
* `git reset --hard <commit_id>`/`git reset --hard HEAD^`：回退到指定版本/上一个版本，将之后的提交都删除
  - `git revert HEAD^`/`git revert <commit_id> -m <branch_number>`：创建一个新的提交，将某次提交反转以达到消除的目的（可用于出错内容已合并到`master`的情况）
* `git reflog`：查看命令历史（包括`commit`，`pull`，`reset`命令记录）
* `git checkout -- <file>`：丢弃工作区的修改，回到最近一次`git add`或`git commit`时的状态（版本库替换工作区文件）
* `git reset HEAD <file>`：撤销暂存区(unstage)的修改，重新放回工作区
* `git rm <file>`：删除版本库指定文件

## 远程仓库
> 创建SSH Key: <br/>
> 1. `ssh-keygen -t rsa -C "youremail@example.com"`<br/>
> 2. `/Users/用户名/.ssh/id_rsa.pub`文件内容即为Key内容
 
* `git remote add <origin> git@github.com:HelloiWorld/gitmemo.git`：将本地仓库关联到远程仓库(名为origin)
* `git push -u origin master`：将本地库内容推送到远程
    
    > `-u origin master`：把本地的`master`分支（如果当前不在`master`分支，还包含一个切换操作）内容推送到远程新的`master`（如果不为`master`且不存在，会创建）分支，，并把本地的`master`分支和远程的`master`分支关联起来(设置为默认主机)，之后推送或拉取可简化命令
* `git clone git@github.com:HelloiWorld/gitmemo.git`：克隆远程仓库到本地并建立关联

## 分支管理
* `git checkout <branch>`：切换到指定分支
  - `git checkout -b <branch>`：创建并切换到指定分支，相当于`git branch <branch>`+`git checkout <branch>`
* `git branch`：列出所有分支，并标记*当前分支
  - `git branch <branch>`：创建新分支
  - `git branch -d <branch>`：删除指定分支
  - `git branch -D <branch>`：如果新分支还未被合并，可强行删除该分支
  - `git branch --set-upstream <local-branch> origin/<remote-branch>`：建立本地分支与远程分支的链接
* `git merge <branch>`：合并指定分支到当前分支
  - `git merge --no-ff -m "merge with no-ff" <branch>`：合并时禁用`Fast forward`
  - `git rebase`：作用与`git merge`相同，但能让提交历史变得“干净”（去掉多余分叉）
* `git stash`：“储藏”当前工作现场
  - `git stash list`：查看储存处工作现场
  - `git stash apply stash@{<number>}/git stash drop stash@{<number>}`：恢复/删除某次储存的工作现场
  - `git stash pop`：恢复工作现场，并将储存区最上面的工作现场删除
* `git remote`：查看远程库信息
  - `git remote -v`：显示远程库详细信息
  - `git remote rm <origin>`：删除已关联的名为origin的远程库
* `git pull <remote> <branch>`：把最新的提交从某分支上拉取下来

## 标签管理
* `git tag`：查看所有标签
  - `git tag <name>`：为当前分支最新提交打一个标签
  - `git tag <name> <commit-id>`：为某次提交打一个标签
  - `git tag -a <tagname> -m "<description>" <commit-id>`：`-a`指定标签名，`-m`指定说明文字
  - `git tag -d <tagname>`：删除本地指定标签
* `git show <tagname>`：查看指定标签信息
* `git push origin <tagname>`／`git push origin --tags`：将本地某个标签／所有未被推送到远程的标签推送到远程
  - `git push origin :refs/tags/<tagname>`：删除指定远程标签（先将本地该标签删除）

## 自定义Git
* `git config --global`：设置本机Git仓库全局配置
  - `git config --global user.name "Your Name"`：设置本机名
  - `git config --global user.email "email@example.com"`：设置本机email
  - `git config --global color.ui true`：显示彩色UI
  - `git config --global alias.<alias> '<command>'`：为某条命令设置别名，类似宏
* `.gitignore`：[https://github.com/github/gitignore](https://github.com/github/gitignore)
  - `git add -f <file>`：无视忽略强制添加到Git
  - `git check-ignore -v <file>`：检查某文件对应的忽略规则

### 搭建Git服务器
假设你已经有`sudo`权限的用户账号
第一步，安装`git`：

	$ sudo apt-get install git
第二步，创建一个`git`用户，用来运行`git`服务：

	$ sudo adduser git
第三步，创建证书登录：

收集所有需要登录的用户的公钥，就是他们自己的`id_rsa.pub`文件，把所有公钥导入到`/home/git/.ssh/authorized_keys`文件里，一行一个。

第四步，初始化Git仓库：

先选定一个目录作为Git仓库，假定是`/srv/sample.git`，在`/srv`目录下输入命令：

	$ sudo git init --bare sample.git
Git就会创建一个裸仓库，裸仓库没有工作区，因为服务器上的Git仓库纯粹是为了共享，所以不让用户直接登录到服务器上去改工作区，并且服务器上的Git仓库通常都以`.git`结尾。然后，把owner改为`git`：

	$ sudo chown -R git:git sample.git
第五步，禁用shell登录：

出于安全考虑，第二步创建的git用户不允许登录shell，这可以通过编辑`/etc/passwd`文件完成。找到类似下面的一行：

	git:x:1001:1001:,,,:/home/git:/bin/bash
改为：

	git:x:1001:1001:,,,:/home/git:/usr/bin/git-shell
这样，`git`用户可以正常通过ssh使用git，但无法登录shell，因为我们为`git`用户指定的`git-shell`每次一登录就自动退出。

第六步，克隆远程仓库：

现在，可以通过`git clone`命令克隆远程仓库了，在各自的电脑上运行：

	$ git clone git@server:/srv/sample.git
	Cloning into 'sample'...
	warning: You appear to have cloned an empty repository.



## git开发流程
1. `fork`请把项目`fork`到自己的仓库
2. `git clone git@github.com:HelloiWorld/gitmemo.git`把项目克隆到本地
3. `git remote rename origin upstream`把`origin`仓库名改为`upstream`
4. `git remote add origin [自己的仓库地址]`

规范的开发形式应该为`pull request`形式

1. 本地`git checkout`一个新的分支，修改完commit以后，`git push`(默认就是`push`到`origin`仓库的对应分支，若没有则新建一个分支)
2. 此时打开`https://github.com/HelloiWorld/gitmemo`会自动有一个生成pr的按钮，然后生成一个pr，pr会跑ci，然后通过了ci以后可以merge.
3. 回到本地，`git pull upstream master`保持本地`master`为最新状态。

tips:

1. 目前基本是一个人负责一个项目的情况下，允许本地直接修改`master`，然后`git push upstream master`这个操作（会直接覆盖主仓库的`master`）， 由每个人对所写项目的主仓库负责。
2. 每次开发前最好`git pull upstream master`，这个习惯会让你避免开发了一会发现跟主仓库代码不一致还要去`git rebase`
3. 一旦人员数超过3人开发同一个项目，禁止1操作。