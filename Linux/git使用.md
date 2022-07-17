测试：@172.16.129.111 作为远程仓库 @1782.16.129.172有subgit，@172.16.11.127 作为开发地

1、初始化仓库
正常库git init          
本地仓库向远程仓库push时，如果远程仓库正在push的分支上（如果当时不在push的分支。就没有问题），那么push的结果不会反应在work tree上，也即在远程仓库的目录下对应的文件还是之前的内容。
解决方法：git reset --hard 
裸库 git init --bare

.git 文件

2、删除仓库
删除.git文件

3、查询远程主机
git remote -v
添加远程主机
git remote add <远程主机名> 地址

4、拉取代码
git fetch
git merge = git pull <远程主机名> <远程分支名>:<本地分支名>

5、提交代码
git add
git commmit -m ""

6、回退版本
git reset --hard/soft HEAD

6、上传代码
git push <远程主机名> <本地分支名>:<远程分支名>
git push <远程主机名> <本地分支名>               本地分支名与远程分支名相同

6、子项目
（1）为主项目添加子项目
git submodule add 
git commit

主项目目录下 .gitmodule保存子模块的信息 子模块目录下 .git指明子模块.git文件位置在主项目.git/modules/子模块名

（2）克隆含有子模块的代码
git clone/mkdir
git submodule init（在子模块目录下）
git submodule update

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

7、查看提交记录
git log -p -online
导出重定位 >
git reflog
git diff 

8、合并代码
合并策略
（1）Fast-forward
（2）Recursive
（3）Ours&Theirs
（4）Octopus
（5）git rebase
（6）Resolve

9、合并两个不同的仓库
见知乎收藏