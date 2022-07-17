静态编译 gcc  源文件 -o 生成 --static
动态编译 gcc  源文件 -o 生成
strace  ./qemu-sw64 -cpu core3 /home/fei/hello-sw-dynamic   >log 2>&1 可以查看正确的错误的日志
找到access那一栏，可以看到去哪个目录下找动态库
目前6.2没做动态这一块

lld 可执行文件
可以它的看到依赖库文件

查看文件所在路径

ldconfig -p|grep fortran

查看软件是怎么搜索需要链接的库文件的，内部链接脚本的运行过程

ld -lgfortran --verbose 

gcc -g 表示可以调试
