---
title: Tmux
date: 2023-12-13 9:31:28
tags:
- Tool
categories:
- Notes
---

# 常用快捷键

~/.tmux.conf 可修改 tmux 配置。

`tmux`: open a new session.  
`C-b` -> `C-a`  
`C-b %`: 左右分屏。改为-> `C-a |`  
`C-b "`: 上下分屏。-> `C-a -`  
`C-b <arrow key>`:在 panes 间移动。-> `alt + arrow`  
`exit` or hit `Ctrl-d`：退出当前 pane。  
`C-b c`: new window.  
`C-b p`: previous window.  
`C-b n`: next window.  
`C-b <number>` : move to window n.  
`tmux ls`: list sessions.  
`tmux attach -t 0`: attach to 0 session.  
`C-b ?`: help message.  
`C-b z`: make a pane go full screen. Hit `C-b z` again to shrink it back to its previous size  
`C-b C-<arrow key>`: 调整当前 window 的大小。  
`C-b ,`: 重命令当前 window。  
`<C-b> [` Start scrollback. You can then press `<space>` to start a selection and `<enter>` to copy that selection.

# 自定义快捷键

`C-b` -> `C-a`  
`C-b %` -> `C-a |`  
`C-b "` -> `C-a -`  
`C-b <arrow key>` -> `alt <arrow key>`

# oh-my-zsh tmux 插件

zsh 自带了一个 tmux 插件，只要在.zshrc plugins=(... tmux) 增加 tmux.

再在`source $ZSH/oh-my-zsh.sh` 之前加上`ZSH_TMUX_AUTOSTART=true`, 就可以在 zsh 启动时自动启动 tmux.
