系统级启动 先把core3-hmcode拷贝进/build/pc-bios/文件夹中，运行启动脚本 ./run.sh
 ./run.sh
/home/fei/qemu-6-6.2/qemu6/build/qemu-system-sw64 -machine core3 -m 512 -smp 1 -serial stdio -drive file=./sw_qemu/test6b.raw,if=virtio -kernel ./sw_qemu/vmlinux -append "root=/dev/vda2 rw console=ttyS0,115200n8 ignore_loglevel systemd.unit=multi-user.target numa=off" -vnc :7

file pc-bios写成的raw文件，vmlinux是内核

用户级启动             
qemu-6-6.2/qemu6/build/qemu-sw64 -cpu core3 自己写个Hello

启动虚拟机
图形化界面，看README里有写
