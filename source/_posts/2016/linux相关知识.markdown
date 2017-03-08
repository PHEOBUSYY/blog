layout: "post"
title: "linux相关知识"
date: "2016-10-05 14:45"
---
### linux硬链接(Hard link)和符号链接(Symbolic link)的作用和创建方式
  在linux文件系统中,保存在磁盘分区中的文件不管是什么类型都给它分配了一个编号,称为索引节点(Inode index).硬链接指的就是多个文件名指向了同一个索引节点,硬链接的作用就是允许一个文件有多个指向的有效路径名称,这样用户就可以通过硬链接来指向重要文件,当删除了其中一个节点并不会影响文件本身和其他的连接,保证文件的安全,只有当所有的连接全部都删除后,文件才会被真正删除.类似于文件"指针".符号链接又叫软链接,有点类似于windows的快捷方式,他实际上是一个特殊的文件,文件实际上是一个文本文件,其中包含另一个文件的位置信息.

  创建一个空文件f1     touch f1
  创建一个硬链接f2     ln f1 f2
  创建一个软链接f3     ln -s f1 f3
  查看文件属性         ls -li
  可以看到f1,f2文件的索引节点相同,文件大小相同,而f1,f3文件索引和文件大小都不相同

  输入文本到f1         echo "I am f1 file" >> f1
  输出f1,f2,f3文本    cat f1
                     "I am f1 file"
                     cat f2
                     "I am f1 file"
                     cat f3
                     "I am f1 file"
  删除f1              rm f1
  输出f2,f3文本       cat f2
                     "I am f1 file"
                     cat f3
                     No such file or directory

### basename 和 dirname
basename 命令可以获取到路径最后的文件名
dirname 命令可以获取到路径除了文件名的前面部分

比如:
basename /user/yy ---> yy
dirname /user/yy ---> /user                     
