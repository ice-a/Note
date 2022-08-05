参考文章：

打开git bash

# 1. Github新建仓库

新建私人或公共仓库

# 2. 本地初始化仓库

```shell
# 在需要初始化仓库的目录下
git init
```

# 3. 将机器与Github仓库关联起来

生成ssh key，并添加到github，建立ssh协议

```shell
ssh -keygen -t rsa -C "邮箱地址"
# 验证是否设置成功
ssh -T git@github.com
```

一张图，路径/c/Users/fy/.ssh/id_rsa.pub,全选拷贝，到github的SSH keys进行添加

```shell
git remote add origin git@github.com:[githubID]/[repositoryName].git
# 验证是否设置成功
git remote -v
```

# 4.配置用户名和邮箱

```shell
git config --global user.name "username" //用户名
git config --global user.email "useremail" //邮箱
```

# 5. 配置.gitignore

过滤规则

# 6. 开始上传

github由于zzzq默认分支名由master改为main，而git的默认分支名还是master，解决办法

```shell
git pull
git add .
git commit -m ""
git push -u origin main
```

github电脑客户端使用GitHub Desktop

ios端github下载，使用自己创的美服appleid

# 7.git挂代理
