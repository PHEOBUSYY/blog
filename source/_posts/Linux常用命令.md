---
layout: "post"
title: "Linux常用命令"
date: "2016-03-1 22:00"
tags: [linux, blog, github]
---
一些常用的Linux命令

## MV命令----重命名文件和移动文件

  mv [-f | -i | -n][-v] source target 重命名文件
  mv [-f | -i | -n][-v] source ... dictionary 移动文件
  -f: 强制覆盖文件，并且会覆盖之前的-i,-n选项
  -i: 如果有重复文件的话，提示用户
  -n: 如果已经存在文件，不覆盖，并且会覆盖之前的-i，-f选项
  -v：可视化选项

<!--more-->  
## RM命令----删除文件和文件夹
  rm [-dfiPRrvW] file ...
  -d: 尝试删除文件夹  
  -f: 强制删除
  -i: 询问是否删除
  -P: 没看懂
  -r: 递归删除目录及内容
  -R: 等同于-r
  -v: 可视化
  -W: 尝试不删除文件，当前只支持空白文件

  删除一个非空文件夹可以使用 rm -r name
  删除空文件夹 rmdir name
