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
git config --global user.name "username" # 用户名
git config --global user.email "useremail" # 邮箱
```

# 5. 配置.gitignore

过滤规则

# 6. 开始上传

github由于zzzq默认分支名由master改为main，而git的默认分支名还是master，解决办法

```shell
git pull
git add .
git commit -m "日志信息"
git push -u origin main
```

github电脑客户端使用GitHub Desktop

ios端github下载，使用自己创的美服appleid

# 7.git设置代理

```shell
# git config --global http.proxy http://proxyUsername:proxyPassword@proxy.server.com:port
# git config --global https.proxy http://proxyUsername:proxyPassword@proxy.server.com:port
# git config --global 协议.proxy 协议://代理服务器ip地址:端口号
git config --global http.proxy http://127.0.0.1:7890   # 代理服务器的ip地址
git config --global https.proxy http://127.0.0.1:7890
git config --global http.proxy 'socks5://127.0.0.1:7891'
git config --global https.proxy 'socks5://127.0.0.1:7891'

# 取消代理
git config --global --unset http.proxy
git config --global --unset https.proxy


# 只对github.com
git config --global http.https://github.com.proxy http://127.0.0.1:7890
git config --global http.https://github.com.proxy socks5://127.0.0.1:7891

# 取消代理
git config --global --unset http.https://github.com.proxy


# $ git clone https://仓库地址 --config "https.proxy=proxyHost:proxyPort"
git clone https://github.com/skywind3000/asyncrun.vim.git --config https.proxy=https://127.0.0.1:7890
```

注意：代理只对http、https、sock5协议有效，对ssh协议无效

```shell
[user]
    name = feiyang
    email = sorachyan.fy@gmail.com
[http]
    proxy = http://192.168.47.1:7890
[https]
    proxy = http://192.168.47.1:7890
```
