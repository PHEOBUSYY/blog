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
