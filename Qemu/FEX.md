>资料：

env.sh

```
export C_INCLUDE_PATH=/usr/lib/llvm-10/lib/clang/10.0.0/include:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=/usr/lib/llvm-10/lib/clang/10.0.0/include:$CPLUS_INCLUDE_PATH
```

报错：too mang arguments for function-like macro \_FD_ZERO()
/usr/includex86_64-linux-gnu/bits/select.h
与__GNUC__宏相关，函数宏_FD_ZERO()在22.04中只有一个分支，而在20.04中有两个分支，注释掉if分支后解决。