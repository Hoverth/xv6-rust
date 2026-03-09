# Environmental configuration

#### QEMU Configuration

```shell
wget https://download.qemu.org/qemu-5.0.0.tar.xz
tar xvJf qemu-5.0.0.tar.xz  
cd qemu-5.0.0  
./configure --target-list=riscv32-softmmu, riscv64-softmmu   
make -j$ (nproc)  
sudo make install  
```

**Note**: We use QEMU-5.0.0 for program build, and it is recommended not to use other versions, which may cause our program not to run

You may have some problems when installing QEMU, and we’ve given a list of possible problems and solutions:

`ERROR: pkg-config binary 'pkg-config' not found`:`sudo apt-get install pkg-config`

`ERROR: glib-2.48 gthread-2.0 is required to compile QEMU`:`sudo apt-get install libglib2.0-dev`

`ERROR: pixman >= 0.21.8 not present`: `sudo apt-get install libpixman-1-dev`

The above is a problem that may be encountered, if you encounter other problems, you only need to install the corresponding dependencies according to the prompt.

#### Rust environment configuration

We recommend using an official script to build the environment:

```shell
curl https://sh.rustup.rs -sSf | sh
```

Since Rust’s server is set up abroad, it’s likely to time out without a proxy, so we can build the environment using a UMPC image:

```shell
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
curl https://sh.rustup.rs -sSf | sh
```

After that, you can test whether the configuration of the rust environment is completed with some commands:

```shell
source $HOME/.cargo/env  
Rustc -version
```

After installing Rust, we recommend that you add something under the ".cargo/config" file (created without it), which will speed up your installation dependencies:

```shell
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```

In addition, you need to install some Rust tools to build our program:

```shell
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
rustup component add llvm-tools-preview
```

After that, you can clone our project to start your OS journey!

```shell
git clone https://github.com/Ko-ok-OS/xv6-rust.git
cd xv6-rust/kernel
make run
```

