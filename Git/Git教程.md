# 第一章 Git概述

# 第二章 Git安装

Linux上安装Git

```shell
sudo apt install git
```

Windows上安装Git

Gitbash美化，根目录C:\Users\fy\AppData\Local\GitHubDesktop\app-2.5.5\resources\app\git

/etc/profile.d/git-prompt.sh备份

```shell
if test -f /etc/profile.d/git-sdk.sh
then
        TITLEPREFIX=SDK-${MSYSTEM#MINGW}
else
        TITLEPREFIX=$MSYSTEM
fi

if test -f ~/.config/git/git-prompt.sh
then
        . ~/.config/git/git-prompt.sh
else
        PS1='\[\033]0;$TITLEPREFIX:$PWD\007\]' # set window title
        PS1="$PS1"'\n'                 # new line
        PS1="$PS1"'\[\033[32m\]'       # change to green
        PS1="$PS1"'\u@\h '             # user@host<space>
        PS1="$PS1"'\[\033[35m\]'       # change to purple
        PS1="$PS1"'$MSYSTEM '          # show MSYSTEM
        PS1="$PS1"'\[\033[33m\]'       # change to brownish yellow
        PS1="$PS1"'\w'                 # current working directory
        if test -z "$WINELOADERNOEXEC"
        then
                GIT_EXEC_PATH="$(git --exec-path 2>/dev/null)"
                COMPLETION_PATH="${GIT_EXEC_PATH%/libexec/git-core}"
                COMPLETION_PATH="${COMPLETION_PATH%/lib/git-core}"
                COMPLETION_PATH="$COMPLETION_PATH/share/git/completion"
                if test -f "$COMPLETION_PATH/git-prompt.sh"
                then
                        . "$COMPLETION_PATH/git-completion.bash"
                        . "$COMPLETION_PATH/git-prompt.sh"
                        PS1="$PS1"'\[\033[36m\]'  # change color to cyan
                        PS1="$PS1"'`__git_ps1`'   # bash function
                fi
        fi
        PS1="$PS1"'\[\033[0m\]'        # change color
        PS1="$PS1"'\n'                 # new line
        PS1="$PS1"'$ '                 # prompt: always $
fi

MSYS2_PS1="$PS1"               # for detection by MSYS2 SDK's bash.basrc

# Evaluate all user-specific Bash completion scripts (if any)
if test -z "$WINELOADERNOEXEC"
then
        for c in "$HOME"/bash_completion.d/*.bash
        do
                # Handle absence of any scripts (or the folder) gracefully
                test ! -f "$c" ||
                . "$c"
        done
fi
```

# 第三章 Git常用命令

## 3.1 设置用户签名

语法

```shell
# 全局的签名设置
$ git config --global user.name "用户名" # 用户名
$ git config --global user.email 邮箱    # 邮箱
# 本地的签名设置，只对当前repository生效
$ git config user.name "用户名" # 用户名  默认--local
$ git config user.email 邮箱    # 邮箱
```

 查看设置

```shell
# 全局设置在用户主目录下$HOME
$ cat ~/.gitconfig
[user]
    name = feiyang
    email = sorachyan.fy@gmail.com
# Linux系统中用户主目录路径为/home/username/
# Windows系统中用户主目录路径为/c/Users/username/

# 本地设置在.git/config中

$ git config --list
```

说明：

签名的作用是区分不同操作者身份。用户的签名信息在每一个版本的提交信息中能够看到，以此确认本次提交是谁做的。Git 首次安装必须设置一下用户签名，否则无法提交代码。

※注意：这里设置用户签名和将来登录 GitHub（或其他代码托管中心）的账号没有任何关系。

## 3.2 初始化本地库

语法

正常库

```shell
$ git init
Initialized empty Git repository in D:/Git-Space/SH0720/.git/
```

本地仓库向远程仓库push时，如果远程仓库正在push的分支上（如果当时不在push的分支。就没有问题），那么push的结果不会反应在work tree上，也即在远程仓库的目录下对应的文件还是之前的内容。
解决方法：

```shell
$ git reset --hard 
```

裸库 

```shell
$ git init --bare
```

.git 文件

删除本地库

删除.git文件

## 3.3 查看本地库状态

语法

```shell
$ git status
On branch master 

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

## 3.4 添加暂存区

语法

```shell
# 将工作区中新建/修改/删除的文件内容，添加到暂存区。
$ git add [options] [<pathspec>]
    <pathspec>      指定要提交的文件路径
    -A,--all        提交所有变化
    -u,--update     提交被修改（modified）和被删除（deleted）的文件，不包括新文件（new）。
    .               提交新文件（new）和被修改（modified）文件，不包括被删除（deleted）文件
    -h,--help       -h查看帮助，--help可以查看更详细内容
# 将暂存区文件撤出
$ git restore --staged [filename]
```

## 3.5 提交本地库

```shell
$ git commit 
$ git commit -m "日志信息" 文件名
$ git commit --amend
```

## 3.6 历史版本

### 3.6.1 查看历史版本

```shell
$ git reflog        # 查看版本信息
$ git log           # 查看版本详细信息
$ git log  文件名    # 查看文件的历史修改版本
$ git log -p 文件名  # 查看文件的历史修改详细信息
```

### 3.6.2 版本穿梭

```shell
$ git reset --hard 版本号
```

## 3.7 比较内容

暂存区vs工作区

```shell
git diff filename
```

# 第四章 Git分支操作

## 4.1 什么是分支

在版本控制过程中，同时推进多个任务，为每个任务，我们就可以创建每个任务的单独分支。使用分支意味着程序员可以把自己的工作从开发主线上分离开来，开发自己分支的时候，不会影响主线分支的运行。对于初学者而言，分支可以简单理解为副本，一个分支就是一个单独的副本。（分支底层其实也是指针的引用）

## 4.2 分支的好处

同时并行推进多个功能开发，提高开发效率。

各个分支在开发过程中，如果某一个分支开发失败，不会对其他分支有任何影响。失败的分支删除重新开始即可。

## 4.3 分支的操作

### 4.3.1 查看分支

语法

```shell
$ git branch -v 
```

案例

```shell
fy@DESKTOP-06PPN7S MINGW64 /d/Git-Space/git-demo (master)
$ git branch -v
* master 087a1a7 my third commit    # *代表当前所在的分支
```

### 4.3.2 创建分支

语法:

```shell
$ git branch 分支名
```

案例:

```shell
$ git branch hot-fix
fy@DESKTOP-06PPN7S MINGW64 /d/Git-Space/git-demo (master)
$ git branch -v
  hot-fix 087a1a7 my third commit   # 刚创建的新的分支，并将主分支 master 的内容复制了一份
* master  087a1a7 my third commit
```

### 4.3.3 切换分支

语法

```shell
$ git checkout 分支名
```

案例

```shell
fy@DESKTOP-06PPN7S MINGW64 /d/Git-Space/git-demo (master)
$ git checkout hot-fix
Switched to branch 'hot-fix'
fy@DESKTOP-06PPN7S MINGW64 /d/Git-Space/git-demo (hot-fix)
$ git branch -v
* hot-fix 087a1a7 my third commit    
  master  087a1a7 my third commit
```

### 4.3.4 合并分支

语法

```shell
$ git merge 分支名
```

案例：将hot-fix分支合并到master分支上

```shell
fy@DESKTOP-06PPN7S MINGW64 /d/Git-Space/git-demo (hot-fix)
$ git checkout master
Switched to branch 'master'
fy@DESKTOP-06PPN7S MINGW64 /d/Git-Space/SH0720 (master)
$ git merge hot-fix
Auto-merging hello.txt
CONFLICT (content): Merge conflict in hello.txt
Automatic merge failed; fix conflicts and then commit the result.
```

### 4.3.5 合并冲突

# 第五章 团队协作工具

# 第六章 GitHub操作

## 6.1 创建远程仓库

## 6.2 远程仓库操作

### 6.2.1 查看远程仓库

语法

```shell
$ git remote -v
```

案例

```shell
fy@DESKTOP-06PPN7S MINGW64 /d/笔记/Note (main)
$ git remote -v
origin  git@github.com:SoraChan-fy/Note.git (fetch)
origin  git@github.com:SoraChan-fy/Note.git (push)
# 别名  远程主机/目录
```

### 6.2.2 添加远程仓库

```shell
git remote add [别名] 远程仓库地址
```

### 6.2.3 推送本地分支到远程仓库

```shell
$ git push <远程主机名> <本地分支名>:<远程分支名>
$ git push <远程主机名> <本地分支名>               # 本地分支名与远程分支名相同
$ git push 远程仓库地址
```

### 6.2.4 拉取远程库到本地库

```shell
# 
git pull <远程主机名> <远程分支名>:<本地分支名>
git fetch
git merge 
```

### 、子项目

（1）为主项目添加子项目

```shell
git submodule add 
git commit
```

主项目目录下 .gitmodule保存子模块的信息 子模块目录下 .git指明子模块.git文件位置在主项目.git/modules/子模块名

（2）克隆含有子模块的代码

```shell
git clone/mkdir
git submodule init（在子模块目录下）
git submodule update
```

（3）修改子模块代码并提交
子模块修改提交，在主项目中还要提交一次，可以用git status查看文件情况

（4）删除子模块
删除子模块文件夹，并且删除以下文件 .gitmodule .git/config .git/modules/子模块名  

By default, updating the current branch in a non-bare repository is denied, because it will make the index and work tree inconsistent with what you pushed, and will require 'git reset --hard' to match the work tree to HEAD.
You can set 'receive.denyCurrentBranch' configuration variable to 'ignore' or 'warn' in the remote repository to allow pushing into its current branch; however, this is not recommended unless you arranged to update its work tree to match what you pushed in some other way.

解决方法：
config
[receive]
    denyCurrentBranch = ignore

### 、查看提交记录

```git
git log -p --oneline
导出重定位 >
git reflog
git diff 
```

### 、合并代码

合并策略
（1）Fast-forward
（2）Recursive
（3）Ours&Theirs
（4）Octopus
（5）git rebase
（6）Resolve

### 、合并两个不同的仓库

见知乎收藏

### Git管理工具

#### Git bash终端中文文件名字显示不正常（显示为数字）

右键options，可以看，经过测试不需要设置，直接执行命令

```shell
git config --global core.quotepath false
```

#### Git Desktop

#### SoureceTree

总是显示完整控制台输出

ssh客户端配置，配置ssh密钥和ssh客户端openssh

代理服务器，还要勾选向Git/~~添加代理服务器信息，才能正常使用