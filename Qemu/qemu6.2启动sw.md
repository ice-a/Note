build sw64 user-static
../configure --extra-cflags="-g -O0 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2" --extra-ldflags="-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,--as-needed" --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib/x86_64-linux-gnu --libexecdir=/usr/lib/qemu --firmwarepath=/usr/share/qemu:/usr/share/seabios:/usr/lib/ipxe/qemu --localstatedir=/var --disable-blobs --disable-strip --interp-prefix=/etc/qemu-binfmt/%M --with-git-submodules=ignore  --static --disable-pie --disable-system --disable-xen --target-list="sw64-linux-user"  --enable-tcg-interpreter

build sw64 system
../configure --extra-cflags="-gdwarf-2 -g3 -O0" --target-list=sw64-softmmu  --enable-spice --disable-kvm --enable-virtfs --extra-cflags="-fpic -fPIC"  --extra-ldflags="-fpic -fPIC" --enable-debug --enable-seccomp  --disable-werror

build x86 user-dynamic
../configure --extra-cflags="-gdwarf-2 -g3 -O2" --enable-linux-user --target-list=x86_64-linux-user,i386-linux-user,aarch64-linux-user,arm-linux-user,mips-linux-user,mips64-linux-user,mipsel-linux-user,mips64el-linux-user --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib/x86_64-linux-gnu --libexecdir=/usr/lib/qemu --interp-prefix=/home/fei/qemu-binfmt/%M --disable-docs --disable-werror --disable-blobs --enable-debug --enable-debug-stack-usage --enable-debug-tcg --enable-debug-info --enable-qom-cast-debug

测试性能需要把-g什么的调试信息去掉
../configure --extra-cflags="-O2" --enable-linux-user --target-list=x86_64-linux-user,sw64-linux-user --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib/x86_64-linux-gnu     --libexecdir=/usr/lib/qemu --interp-prefix=/etc/qemu-binfmt/%M --disable-docs --disable-werror --disable-blobs --enable-debug --enable-debug-stack-usage --enable-debug-tc    g --enable-debug-info --enable-qom-cast-debug

系统级启动 先把core3-hmcode拷贝进/build/pc-bios/文件夹中，运行启动脚本 ./run.sh
 ./run.sh
/home/fei/qemu-6-6.2/qemu6/build/qemu-system-sw64 -machine core3 -m 512 -smp 1 -serial stdio -drive file=./sw_qemu/test6b.raw,if=virtio -kernel ./sw_qemu/vmlinux -append "root=/dev/vda2 rw console=ttyS0,115200n8 ignore_loglevel systemd.unit=multi-user.target numa=off" -vnc :7

file pc-bios写成的raw文件，vmlinux是内核

用户级启动             
qemu-6-6.2/qemu6/build/qemu-sw64 -cpu core3 自己写个Hello

启动虚拟机
图形化界面，看README里有写
