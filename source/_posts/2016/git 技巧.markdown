layout: "post"
title: "git 技巧"
date: "2016-11-09 14:08"
---

## 如何把已经加入暂存区的文件移除
git reset <file name>
如果要移除暂存区全部文件使用
git reset

## 如何把其它分支的提交合并到当前分支
git cherry_pick commitId 就可以了，然后push到远程分支

### 基于某个tag来创建分支
git checkout -b 分支名称 tag名称 就可以了

### 如何删除一个本地仓库
直接把文件目录下的 *.git* 文件夹删除就可以了。
> rm -rf .git

### 如何把一个已经添加到代码库中的文件加入到ignore中
通过下面到命令从版本库中把指定的文件删除
> git rm --cached 文件名
然后把该文件加入到 *.gitignore* 中，提交后就可以生效了
