# Getting Started

## First-time Git setup

`/etc/gitconfig`: system-wide configuration file. 影响整个系统的 git 配置. `git config --system` 选项.

`~/.gitconfig` 或 `~/.config/git/config`: user-specific configuration file. 影响当前用户的 git 配置. `git config --global` 选项.

`.git/config`: repository-specific configuration file. 影响当前仓库的 git 配置. `git config --local` 选项.

通过下面这个命令可以查看所有的配置, 以及在上述哪个文件中:

```shell
$ git config --list --show-origin
```

配置邮箱和姓名, 默认编辑器:

```shell
$ git config --global user.name "John Doe"
$ git config --global user.email johndoe@example.com
$ git config --global core.editor vim
```

# Git Basics

## short status

`git status -s`: git status 的简化版.

## ignore files

1. 以斜杠开头 (`/`): 避免递归匹配

   - 例如：`/TODO` 只会匹配根目录下的 TODO 文件
   - 而不带斜杠的 `TODO` 会匹配所有目录下的 TODO 文件

2. 以斜杠结尾 (`/`): 指定目录
   - 例如：`build/` 表示忽略 build 目录及其内所有内容
   - 而 `build` 会匹配名为 build 的文件和目录

示例：

```gitignore
# ignore all .a files
*.a
# but do track lib.a, even though you're ignoring .a files above
!lib.a
# only ignore the TODO file in the current directory, not subdir/TODO
/TODO
# ignore all files in any directory named build
build/
# ignore doc/notes.txt, but not doc/server/arch.txt
doc/*.txt
# ignore all .pdf files in the doc/ directory and any of its subdirectories
doc/**/*.pdf
```

github 维护的一个.gitignore 模板:https://github.com/github/gitignore

## git log

`git log -p -n`: 查看最近 n 次提交的差异.

`git log -S "string"`: 查看包含指定字符串的提交, 这样可以查看每个变量/函数的修改记录.

`git log -- path/to/file`: 查看指定文件或目录的提交记录.
