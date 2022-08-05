```shell
mkdir build
cd build
vim config.sh
chomd +x config.sh
./config.sh
make -j
sudo make install
```

git clone下载源代码，读Readme.rst有详细说明Building怎么编译，Readme有测试命令，编译之前改文件拥有者sudo chown -R fei:fei /home/fei/qemu-6.2.0-work1/。还要更改文件权限，增加执行权限chmod +x。去configure中Standard options中可以看到编译参数（./configure --help），重点看 --interp-prefix=PREFIX，动态库的路径。 --target-list=LIST，目标架构。然后新建build文件夹，新建config.sh脚本，添加相关参数，运行的话参数设置O2，调试程序参数设置O0。./config.sh配置环境，make -j编译。

build user-static
#../configure --extra-cflags="-g -O0 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2" --extra-ldflags="-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,--as-needed" --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib/x86_64-linux-gnu --libexecdir=/usr/lib/qemu --firmwarepath=/usr/share/qemu:/usr/share/seabios:/usr/lib/ipxe/qemu --localstatedir=/var --disable-blobs --disable-strip --interp-prefix=/etc/qemu-binfmt/%M --with-git-submodules=ignore  --static --disable-pie --disable-system --disable-xen --target-list="sw64-softmmu sw64-linux-user i386-linux-user x86_64-linux-user aarch64-linux-user arm-linux-user mips-linux-user mipsel-linux-user mips64-linux-user mips64el-linux-user riscv64-linux-user"
#../configure --extra-cflags="-g -O0 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2" --extra-ldflags="-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,--as-needed" --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib/x86_64-linux-gnu --libexecdir=/usr/lib/qemu --firmwarepath=/usr/share/qemu:/usr/share/seabios:/usr/lib/ipxe/qemu --localstatedir=/var --disable-blobs --disable-strip --interp-prefix=/etc/qemu-binfmt/%M --with-git-submodules=ignore  --static --disable-pie --disable-system --disable-xen --target-list="sw64-softmmu,sw64-linux-user,x86_64-linux-user,x86_64-softmmu "
../configure --extra-cflags="-g -O0 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2" --extra-ldflags="-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,--as-needed" --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib/x86_64-linux-gnu --libexecdir=/usr/lib/qemu --firmwarepath=/usr/share/qemu:/usr/share/seabios:/usr/lib/ipxe/qemu --localstatedir=/var --disable-blobs --disable-strip --interp-prefix=/etc/qemu-binfmt/%M --with-git-submodules=ignore  --static --disable-pie --disable-system --disable-xen --target-list="sw64-linux-user"

build system
../configure --extra-cflags="-gdwarf-2 -g3 -O0" --target-list=sw64-softmmu  --enable-spice --disable-kvm --extra-cflags="-fpic -fPIC"  --extra-ldflags="-fpic -fPIC" --enable-debug --enable-seccomp  --disable-werror

```shell
which qemu-xx # /usr/bin/qemu-xx
md5sum qemu-xx
md5sum /usr/bin/qemu-xx
# 两个md5知不一样是因为make install中有一个stripped操作，这一步并不是每一回都有。
```

SWREACH编译器优化

申威编译器目前支持O0、O1、O2、Os、O3选项

-O或-O0生成没有优化的可执行代码

-O1或-O2生成优化的可执行代码

-O3生成高级优化的可执行代码

-Os在支持-O2所有选项的同时不会显著增加代码规模，其可以有更多优化代码空间
