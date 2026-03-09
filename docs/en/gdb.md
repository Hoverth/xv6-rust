#GDB debugging

We need to use `riscv64-unknown-elf-gdb` to debug our programs.

1. Go to [Tsinghua Mirror Station] (https://mirrors.tuna.tsinghua.edu/gnu/gdb/?C=M&O=D) to download the latest GDB source code
2. Uncoup the source code and locate it to the directory
3. To execute the following order:

```shell
mkdir build
cd build
../configure --prefix=/usr/local  --target=riscv64-unknown-elf
```

4. Compilation of installation

```shell
make -j$(nproc)
sudo make install
```

5. Using the `make debug` directive to debug programs by gdb

