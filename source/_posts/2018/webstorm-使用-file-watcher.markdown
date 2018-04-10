layout: "post"
title: "webstorm 使用 File Watcher"
date: "2018-04-10 17:37"
---
在我们使用『webstorm』的时候，当你创建一个新的文件的时候，如果需要编译的文件类型，比如『pug』、『sass/scss』『JavaScript』等，会在文件顶部出现是否需要开启『File Watcher』选项。这个功能相当于每当我们修改当前文件的时候，webstorm会自动帮我们自动编译这个文件到指定格式。比如『pug』-> 『html』，『sass/scss』-> 『css』等。

<!--more-->

![](/images/2018/04/webstorm_enable_file_watcher.png)

如果我们想要开启『File Watcher』的话，点击『Yes』弹出对话框：

![](/images/2018/04/webstorm_file_watcher_scss.png)

在这个对话框中，核心的参数都在『Watcher Settings』中。其中：

1. Program
这里的『Program』表示编译器或者命令行的本地路径。
当我们通过命令：
```sh
npm install sass -g
```
全局安装sass到本地后，npm会在**/usr/local/bin**中自动生成一个sass命令的软链接。这个软连接路径就是这里『program』的输入

2. Arguments
当我们在终端中输入『sass』时：提示用法为：
```sh
sass <input> [output]
```
这里sass后面的部分就是『Arguments』的输入值。
我这里使用的值为：
```
$FileName$ $FileNameWithoutExtension$.css
```
其中『$FileName$』表示当前文件名，『$FileNameWithoutExtension$』表示当前文件名去掉后缀
对应： index.sass 转换为 index.css。

配置完『Program』和『Arguments』之后，点击ok，完成『File Watcher』配置。
这个时候编辑当前文件，会发现在当前文件的统一目录下生成编译完成后的文件。

实际上『Program』 + 『Arguments』 就相当于上面完成的『sass』命令。
```sh
sass <input> [output]
```
---
下面是pug/jade的配置：注意：要使用『pug』命令，需要额外安装『pug-cli』。

![](/images/2018/04/webstorm_pug_file_watcher.png)

```sh
npm install pug-cli -g
```
