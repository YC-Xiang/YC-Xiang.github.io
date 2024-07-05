---
title: 命令行的艺术
date: 2024-06-28 17:33:28
tags:
- Tool
categories:
- Misc
draft: true
---

https://github.com/jlevy/the-art-of-command-line/blob/master/README-zh.md

# 日常使用

bash：
`ctrl w` 删除最近的一个单词。
`ctrl u` 删除光标前所有的内容。
`alt b/alt f` 向前/后移动一个单词。
`ctrl a` 光标移动到开头。
`ctrl e` 光标移动到末尾。
`ctrl k` 删除光标到末尾。

`history` + `!n` n为需要执行命令的编号。
`ctrl-a #` 将当前命令注释掉执行，并且可以通过history查看，而不用删除掉。

# 文件管理相关

`less` `ps -ef | less`, `history | less`
`head` 查看文件头部内容，默认参数`-n 10`显示前十行。
`tail` 查看文件尾部内容, 默认显示最后十行。`tail -f`: 可以动态监视文件内容的改变。

`chown`: `chown [-cfhvR] [--help] [--version] user[:group] file...`
将当前前目录下的所有文件与子目录的拥有者皆设为 runoob，群体的使用者 runoobgroup `chown -R runoob:runoobgroup *`

`du -hs *` 查看当前路径下各目录硬盘使用情况。

文件系统：
`df -h` 查看各分区占硬盘的情况。
`mount`
`mkfs`

`grep`
`-i` 忽略大小写。
`-o` 只打印匹配到的字符串，而不是整行。
`-n` 显示行号。

# 网络

`ip`
`ip link show` 显示网络设备的运行状态
`ip addr show` 显示网卡IP信息
`ip route show` 显示系统路由
