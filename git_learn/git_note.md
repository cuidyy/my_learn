# git结构
![alt text](iamge.png)

### 查看用户名和地址
git config user.name  
git config user.email

### 修改用户名和地址
git config --global user.name "your name"  
git config --global user.email "your email"

### 把目录变成Git可以管理的仓库
git init

### 把文件添加到仓库(暂存区)
git add filename

### 将工作目录中所有更改添加到暂存区
git add .

### 文件提交到仓库(将暂存区内容提交到版本库)
git commit  -m "文件说明"

### 查看仓库当前的状态
git status

### 查看尚未暂存的文件更新了哪些内容（比较工作目录中当前文件和暂存区快照之间的差异，即修改后还没暂存变化的内容）
git diff

### 查看已暂存文件与最后一次提交文件的差异
git diff --staged

### 查看工作区和版本库里面最新版本的区别
git diff HEAD

### 查看提交历史 以便确定要回退到哪个版本
git log  
简略信息 --pretty=oneline  
图形化显示 --graph

### 查看命令历史 以便确定要回到未来的哪个版本
git reflog

### 版本回退
git reset --hard HEAD^(HEAD：当前版本，HEAD^：上一个版本，HEAD^^：上上一个版本)  
--hard：回退到上个版本的已提交状态，--soft：会回退到上个版本的未提交状态，--mixed：回退到上个版本已添加但未提交的状态。

### 撤销（丢弃）工作区的修改(用版本库里的版本替换工作区的版本)
git checkout -- filename  
分两种情况：一种是文件自修改后还没有被放到暂存区，撤销修改就回到和版本库一模一样的状态   
一种是文件自修改后已经添加到暂存区后，又作了修改，撤销修改就回到添加到暂存区后的状态。

### 撤销（丢弃）暂存区的修改(把暂存区的修改撤销掉，重新放回工作区)
git reset HEAD filename

### 从工作目录和暂存区中删除文件(删除文件并将修改添加到暂存区)
git rm filename  
删除文件并保留在工作目录（使用--cached选项）

### 添加（关联）远程仓库(origin 指定的远程仓库名)
git remote add origin URL

### 查看远程仓库
git remote -v

### 删除远程库
git remote rm name

### 本地库的内容推送到远程
git push -u origin master (把当前分支master推送到远程)  
-u选项是--set-upstream的缩写，它的作用是设置当前分支跟踪远程仓库的同名分支（即远程仓库的master分支）,下一次push则可以省略

### 从远程仓库获取新的提交
git fetch 

### 从远程仓库获取最新的提交，并将这些更改合并到本地当前分支
git pull  
= git fetch + git merge
### 将远程库复制到本地库
git clone URL

### 查看当前分支
git branch

### 创建分支
git branch name

### 删除分支
git branch -d name  
如果删除一个没有合并的分支要用参数 -D

### 重命名分支
git branch -m oldname newname

### 切换分支
git checkout -b name(-b参数表示创建并切换)  
git switch -c name(-c参数表示创建并切换)

### 合并指定分支到当前分支
git merge name  
--no-ff参数可以用普通模式合并，合并后的历史有分支，能看出来曾经做过合并,而fast forward合并就看不出来曾经做过合并  
**当Git无法自动合并分支时，就必须首先解决冲突。解决冲突后，再提交，合并完成。**

### 临时保存当前工作目录的修改(在一个分支上进行开发，但是还没有完成相关的修改，又需要切换到其他分支或者处理其他紧急任务时)
git stash

### 列出所有暂存的更改
git stash list

### 恢复指定的stash（stash内容并不删除，需要用git stash drop来删除）
git stash apply stash@{index}

### 回复指定的stash（并删除stash内容）
git stash pop

### 复制一个特定的提交到当前分支
git cherry-pick commit_id

### 建立本地分支和远程分支的关联
git branch --set-upstream branch-name origin/branch-name

# 多人协作的工作模式

1. 首先，可以尝试用git push origin <branch-name>推送自己的修改；
2. 如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；
3. 如果合并有冲突，则解决冲突，并在本地提交；
4. 没有冲突或者解决掉冲突后，再用git push origin <branch-name>推送就能成功！
5. 如果git pull提示no tracking information，则说明本地分支和远程分支的链接关系没有创建，用命令git branch --set-upstream-to <branch-name> origin/<branch-name>。

### 打标签
git tag tagname（默认最新提交上打上标签）  
git tag tagname commit_id  
-a指定标签名，-m指定说明文字
-d删除标签
### 查看标签
git tag  

### 查看标签信息
git show  
git show tagname
