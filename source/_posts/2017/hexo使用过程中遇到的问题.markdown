---
layout: "post"
title: "hexo使用过程中遇到的问题"
date: "2017-03-28 16:49"
---
### hexo TypeError: Cannot read property 'offset' of null
昨天晚上心血来潮的改了下配置文件，结果悲剧了，hexo g 的时候无法生成模版了，提示 *hexo TypeError: Cannot read property 'offset' of null* 后来上网搜了下发现是时区 *timezone* 写错了，改成 *Asia/Shanghai* 问题解决。
