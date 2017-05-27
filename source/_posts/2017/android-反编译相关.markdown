layout: "post"
title: "android 反编译相关"
date: "2017-05-27 15:02"
---

  今天单位的考勤软件更新了，怀着好奇的心情想看下这个软件内部的代码结构代码。拿到手的是一个 **apk** 文件，那么下面要做的就是如何处理这个 **apk** 文件了。主要分这么三步：
  1. 通过 **apktool** 查看里面的资源和图片文件
  2. 通过 **dex2jar** 把 **dex** 文件转换为 **jar**
  3. 通过 **jd2gui** 来查看java文件代码结构

### apktool

  apktools的官方下载目录在 [这里][e7d5af69] ，不同的操作系统对应不同的安装方法。这里用的mac所以分为两步
  1. 右键保存一个脚本文件，命名为 **apktool**
  2. 下载最新的apktool jar包
  3. 重命名下载的jar包为 **apktool.jar**
  4. 把上面的脚本文件和jar包放入目录 */usr/local/bin*
  5. 保证两个文件都是可执行的 (可以通过 **chmod +x** 命令修改可执行)
  6. 在终端中输入 **apktool** 来确认安装是否成功。

  apktool 有两个常用的命令 *d* *b* 。
  1. *d* 代表反编译apk ，格式为 *apktool d apk制定路径名称*
  2. *b* 代表重打包apk ，格式为 *apktool b apk反编译文件夹路径*
  3.
  [e7d5af69]: https://ibotpeaches.github.io/Apktool/ "apktool官网"

### dex2jar

  通过 dex2jar 这个软件来转换dex为jar包。dex2jar 的下载目录在 [这里][0fb5b2ed]。
  下载下来之后是一个文件夹里面有一堆 sh 脚本，通过脚本的名字可以很容易找到他们的作用。注意这里的sh脚本不一定都是可执行的，所以如果发现没有执行权限的时候可以通过 （chmod +x ）来修改。

  注意：dex2jar在v2.1版本之前是不支持 android multiDex 的，也就是不管你有多少个dex文件，他都是只会处理第一个。如果使用v2.1版本的话dex2jar会自动合并所有的dex文件到一个jar包中。在下载的时候要注意这个版本，上面给出的链接是v2.1版本的所有是可以直接使用的。

  使用方法也很简单：
  ```
  sh ~/dex2jar-2.1-SNAPSHOT/d2j-dex2jar.sh apk路径名称
  ```
  最后在同一个目录中会生成一个jar包。

### JD-GUI  

  JD-GUI是一个可视化的jar包查看工具，通过它可以看到jar包中的源代码。
  下载地址在 [这里][c6948913] ，找到对应的平台下载使用就可以。

  注意：这里我用的mac sierra系统下载了对应的dmg文件之后无法使用，点击直接报错了。后来没办法通过 java -jar的方法是直接调用jar包的方法来解决。具体步骤就是直接把JD-GUI的jar下载下来，然后通过 **java -jar jar对应路径** 的方式来启动。





  [0fb5b2ed]: https://github.com/pxb1988/dex2jar/releases "dex2jar下载"
  [c6948913]: http://jd.benow.ca/ "jd-gui下载"
