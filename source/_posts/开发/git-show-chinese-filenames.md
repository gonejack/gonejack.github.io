---
title: 解决git status不能显示中文
date: 2020-12-01 12:39:34
categories:
  - 技术
tags:
  - git
---
* 现象
status查看有改动但未提交的文件时总只显示数字串，显示不出中文文件名，非常不方便。如下图：
![](assets/status-dig.jpg)

* 原因
在默认设置下，中文文件名在工作区状态输出，中文名不能正确显示，而是显示为八进制的字符编码。

* 解决办法
将 git 配置文件`core.quotepath`项设置为`false`
```shell script
git config —global core.quotepath false
```
