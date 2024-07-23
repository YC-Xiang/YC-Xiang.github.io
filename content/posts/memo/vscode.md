---
title: Vscode备忘录
date: 2024-01-23 16:38:28
tags:
- Misc
categories:
- Misc
---

`keybindings.json`: 键盘快捷方式json文件。  
`setting.json`: vscode设置文件。

块选择：`shift+箭头` /  `shift+Alt+鼠标`

增加光标：`Ctrl+Alt+上/下箭头` / `Alt+click`

复制一行：`Shift+Alt+上/下箭头`

移动一行：`Alt+箭头`

删除一行：`Ctrl+Shift+K`

F2可以代码重构，对project下所有的函数名替换名称

格式化文档：`Shift+Alt+F` 格式化整个文档 / `Ctrl+K → Ctrl+F` 格式化某一行

`Ctrl+Shift+[` / `Ctrl+Shift+]` 折叠函数

`alt + <n>` 切换标签页

## 自定义的一些快捷键

在vscode键盘快捷方式中自定义的一些快捷键：

`alt + ↑/↓`：调整终端大小。

## VSCode 调试

watch内存地址的数据，以16进制打印(加上`,h`)

或者也可以直接执行gdb命令`-exec x/16x 0x5561dc78`

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240723215959.png)

### launch.json

```json
	"version": "0.2.0",
	"configurations": [
		{
			"name": "(gdb) Launch",
			"type": "cppdbg",
			"request": "launch", // 指明当前程序是launch还是attach
			"program": "${workspaceFolder}/ctarget", // 要调试的可执行文件
			"args": [], // 传入的参数e.g. ["arg1", "arg2"]
			"stopAtEntry": false, // 停在main入口？
			"cwd": "${fileDirname}", // 设置当前工作目录
			"environment": [], // 环境变量e.g. [{ "name": "xxx", "value": "yyy" }]
			"externalConsole": false,
			"MIMode": "gdb", // gdb or lldb
			"preLaunchTask": "attack_phase3", // 调试开始前需要运行的task
			"postDebugTask": "", // 调试结束后需要运行的task
			"setupCommands": [ // 启动gdb/lldb的参数
			    {
				"description": "Enable pretty-printing for gdb",
				"text": "-enable-pretty-printing",
				"ignoreFailures": true
			    },
			]
		}

	]
```

比如调试csapp的ctarget

```json
	"version": "0.2.0",
	"configurations": [
		{
			"name": "(gdb) Launch",
			"type": "cppdbg",
			"request": "launch",
			"program": "${workspaceFolder}/attack_lab/ctarget",
			"args": ["-q", "<", "phase3-raw.txt"],
			"stopAtEntry": false,
			"cwd": "${workspaceFolder}/attack_lab/",
			"environment": [
				{
					"name": "LD_PRELOAD",
					"value": "${workspaceFolder}/attack_lab/printf.so"
				}
			],
			"externalConsole": false,
			"MIMode": "gdb",
			"preLaunchTask": "attack_phase3",
			"postDebugTask": "",
			"setupCommands": [
			    {
				"description": "Enable pretty-printing for gdb",
				"text": "-enable-pretty-printing",
				"ignoreFailures": true
			    },
			]
		}

	]
```

注意要设置`cwd`current work directory, 默认的`${fileDirname}`会找不到`phase3-raw.txt`。

![](https://xyc-1316422823.cos.ap-shanghai.myqcloud.com/20240722222717.png)

step over: nexti  
step into: stepi  
step out: finish

### task.json

```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"label": "task1",
			"type": "shell",
			"command": "touch 1.c"
		},
		{
			"label": "task2",
			"type": "shell",
			"command": "touch 2.c"
		},
		{
			"label": "attack_phase3",
			"dependsOn": ["task1", "task2"],
			"type": "shell"
		}
	]
}
```
