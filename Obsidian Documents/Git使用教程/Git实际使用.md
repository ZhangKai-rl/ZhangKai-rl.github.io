>查看连接的远程仓库:`git remote -v`
>断开远程仓库：
  连接新远程仓库：`git remote add origin 远程仓库地址`

> 从远程仓库拉取文件： git pull origint 分支名

> 提交文件到远程仓库：git add .提交全部文件到暂存区->git commit -m "此版本注释信息"->git push origin 分支名。

> 创建分支：git branch 分支名。
> 切换分支：git checkout 分支名。
> 查看所有分支：git branch

> 使文件夹具有git功能（创建本地git仓库）：git init
> 查看状态（查看修改的文件）：git status，可以与git add一起用

git版本号

> 拉取远程仓库的某一个版本
需要远程仓库的一个commit版本时，使用git checkout commit-id切换过去，此时head detached at commit-id，使用`git checkout -b temp`来将这个commit绑定到新分支上。

> 创建远程分支
git push origin 本地分支：新建远程分支