[![travis-badge][]][travis]

[travis-badge]: https://img.shields.io/travis/japaric/rust-cross/master.svg?style=flat-square
[travis]: https://travis-ci.org/japaric/rust-cross

# `rust-cross`
陆续翻译过程中... 在此之前请直接查看[英文版原文](https://github.com/japaric/rust-cross)

> 您需要了解有关交叉编译Rust程序的所有知识！

如果您想将Rust工具链设置位交叉编译器，那么来对地方了！该文档记录了所有必要步骤，可能的陷阱和常见的问题。

> 如果您发现输入错误，无效链接或者文法错误，请创建一个issue指出该问题，我将对文本进行更新。

## Ubuntu 示例

在一个全新的Ubuntu环境，以下命令是将稳定的Rust工具链设置为ARMv7 (\*)设备交叉编译器所必须。
这个例子的目的是展示交叉编译的设置和执行是非常容易的。

（*）ARM **v7**，这些指令无法为Raspberry Pi（树莓派）（即ARM **v6**设备）进行交叉编译。

```
# 安装Rust，强烈建议使用rustup.rs。 查看 https://www.rustup.rs/ 获取详细信息
# 另外您也可以使用multirust. 查看 https://github.com/brson/multirust 获取详细信息
$ curl https://sh.rustup.rs -sSf | sh

# 步骤 0: 我们的目标是ARMv7设备，该目标的triple是`armv7-unknown-linux-gnueabihf`

# Step 1: 安装 C 交叉编译工具链
$ sudo apt-get install -qq gcc-arm-linux-gnueabihf

# Step 2: 安装交叉编译标准crates
$ rustup target add armv7-unknown-linux-gnueabihf

# Step 3: 配置cargo以进行交叉编译
$ mkdir -p ~/.cargo
$ cat >>~/.cargo/config <<EOF
> [target.armv7-unknown-linux-gnueabihf]
> linker = "arm-linux-gnueabihf-gcc"
> EOF

# 测试交叉编译Cargo项目
$ cargo new --bin hello
$ cd hello
$ cargo build --target=armv7-unknown-linux-gnueabihf
   Compiling hello v0.1.0 (file:///home/ubuntu/hello)
$ file target/armv7-unknown-linux-gnueabihf/debug/hello
hello: ELF 32-bit LSB  shared object, ARM, EABI5 version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=67b58f42db4842dafb8a15f8d47de87ca12cc7de, not stripped

# 测试编译好的二进制文件
$ scp target/armv7-unknown-linux-gnueabihf/debug/hello me@arm:~
$ ssh me@arm:~ ./hello
Hello, world!
```

1\. 2. 3. 您现在正在交叉编译！

更多示例，请查看 [Travis CI builds](https://travis-ci.org/japaric/rust-cross).

本指南的其余部分将解释和概括上一个示例中执行的每个步骤。

## 目录

本指南分为两个部分：“正文”和高级主题。 正文包括最简单的情况：将依赖于`std` crate 的Rust程序交叉编译为“受支持目标系统”（提供官方版本）。高级主题部分介绍了`no_std`程序，目标系统格式文件，如何交叉编译“standard crates”和常见故障排除。

高级主题部分以正文中解释的信息为基础。因此，即使您的用例与正文所述的用例不同，您仍应在进入“高级主题”部分之前阅读该部分内容。


- [术语](#terminology)
- [必要条件](#requirements)
    - [目标系统triple](#the-target-triple)
    - [C 交叉编译工具链](#c-cross-toolchain)
    - [交叉编译 Rust crates](#cross-compiled-rust-crates)
- [使用 `rustc` 交叉编译](#cross-compiling-with-rustc)
- [使用 `cargo` 交叉编译](#cross-compiling-with-cargo)
- [高级主题](#advanced-topics)
    - [交叉编译standard crate](#cross-compiling-the-standard-crates)
    - [安装交叉编译standard crates](#installing-the-cross-compiled-standard-crates)
    - [目标系统格式文件](#target-specification-files)
    - [交叉编译 `no_std` 代码](#cross-compiling-no_std-code)
    - [常见问题](#troubleshooting-common-problems)
        - [can't find crate](#cant-find-crate)
        - [crate incompatible with this version of rustc](#crate-incompatible-with-this-version-of-rustc)
        - [undefined reference](#undefined-reference)
        - [can't load library](#cant-load-library)
        - [`$symbol` not found](#symbol-not-found)
        - [illegal instruction](#illegal-instruction)
- [FAQ](#faq)
    - [我想为Linux，Mac和Windows构建二进制文件。 如何从Linux交叉编译到Mac？](#i-want-to-build-binaries-for-linux-mac-and-windows-how-do-i-cross-compile-from-linux-to-mac)
    - [如何编译完全静态链接的Rust二进制文件](#how-do-i-compile-a-fully-statically-linked-rust-binaries)
- [License](#license)
    - [贡献](#contribution)

## 术语

首先定义一些术语，以确保我们使用的是同一种语言！

交叉编译最基本的形式涉及两个不同的系统/计算机/设备。 编译程序的 **本地** 系统和执行编译后的程序的 **目标** 系统。

例如，如果您在笔记本电脑上交叉编译Rust程序以在Raspberry Pi 2上执行该程序（RPi2）。那么您的笔记本电脑就是本地系统，而RPi2是目标系统。

但是，（交叉）编译器不会生成仅在单个系统上运行的二进制文件（例如
RPi2）。生成的二进制文件也可以在其他共享某些特性的系统（例如 ODROIDs）上执行，比如它们的体系结构（例如 ARM）和操作系统（例如 Linux）。 为了引用具有共享特征的这套系统，我们使用一个称为 **triple** 的字符串。


Triples 通常有如下格式: `{arch}-{vendor}-{sys}-{abi}`。
24/5000
例如 triple
`arm-unknown-linux-gnueabihf` 指具有这些特征的系统:

- architecture: `arm`.
- vendor: `unknown`. 在这种情况下，未指定任何供应商 和/或 并不重要。
- system: `linux`.
- ABI: `gnueabihf`. `gnueabihf` 表示系统使用`glibc`作为其C标准库（libc）的实现，并具有硬件加速的浮点运算（即FPU）。

诸如RPi2，ODROIDs之类的系统以及几乎每个运行GNU / Linux的ARMv7开发板都属于这个triple。

一些三元组省略了供应商或abi组件，因此它们实际上是真正的“三元组”。其中一个例子是 `x86_64-apple-darwin`, 这里:

- architecture: `x86_64`.
- vendor: `apple`.
- system: `darwin`.

**NOTE** 从现在开始，我将重载术语 **target** 以表示单个目标系统，并且也指一组由某些triple指定的具有共享特征的系统。


## 必要条件

要编译Rust程序，我们需要做4件事：

- 找出目标系统的triple。
- 一个`gcc`交叉编译器，因为`rustc`使用`gcc`将程序[链接]在一起。
- C依赖关系（通常是“ libc”）针对目标系统交叉编译。
- Rust依赖关系，通常是`std` crate，是为目标系统交叉编译的。

["link"]: https://en.wikipedia.org/wiki/Linker_(computing)

### 目标系统triple

要找出目标系统的triple，您首先需要弄清楚目标系统四个信息：

- architecture: 在类Unix系统上，您可以使用 `uname -m`命令查看。
- vendor: Linux: 通常是 `unknown`. Windows: `pc`. OSX/iOS: `apple`
- system: 在类Unix系统上，您可以使用 `uname -s`命令查看。
- ABI: 在Linux上，这是指libc实现，您可以通过 `ldd --version`命令查看。
    Mac 和 \*BSD 系统不提供multiple ABIs, 因此此字段被忽略. 在Windows, AFAIK 只有两个 ABIs: gnu 和 msvc.

接下来，您需要将此信息与Rustc支持的目标系统进行比较，并检查是否存在匹配项。 如果您有nightly-2016-02-14、1.8.0-beta.1或更高版本的`rustc`，则可以使用 `rustc --print target-list` 命令获取受支持目标的完整列表。 以下是从1.8.0-beta.1开始支持的目标系统的列表：


```
$ rustc --print target-list | pr -tw100 --columns 3
aarch64-apple-ios                i686-pc-windows-gnu              x86_64-apple-darwin
aarch64-linux-android            i686-pc-windows-msvc             x86_64-apple-ios
aarch64-unknown-linux-gnu        i686-unknown-dragonfly           x86_64-pc-windows-gnu
arm-linux-androideabi            i686-unknown-freebsd             x86_64-pc-windows-msvc
arm-unknown-linux-gnueabi        i686-unknown-linux-gnu           x86_64-rumprun-netbsd
arm-unknown-linux-gnueabihf      i686-unknown-linux-musl          x86_64-sun-solaris
armv7-apple-ios                  le32-unknown-nacl                x86_64-unknown-bitrig
armv7-unknown-linux-gnueabihf    mips-unknown-linux-gnu           x86_64-unknown-dragonfly
armv7s-apple-ios                 mips-unknown-linux-musl          x86_64-unknown-freebsd
asmjs-unknown-emscripten         mipsel-unknown-linux-gnu         x86_64-unknown-linux-gnu
i386-apple-ios                   mipsel-unknown-linux-musl        x86_64-unknown-linux-musl
i586-unknown-linux-gnu           powerpc-unknown-linux-gnu        x86_64-unknown-netbsd
i686-apple-darwin                powerpc64-unknown-linux-gnu      x86_64-unknown-openbsd
i686-linux-android               powerpc64le-unknown-linux-gnu
```

**NOTE** If you are wondering what's the difference between `arm-unknown-linux-gnueabihf` and
`armv7-unknown-linux-gnueabihf`, the `arm` triple covers ARMv6 and ARMv7 processors whereas `armv7`
only supports ARMv7 processors. For this reason, the `armv7` triple enables optimizations that are
only possible on ARMv7 processors. OTOH, if you use the `arm` triple you would have to opt-in to
these optimizations by passing extra flags like `-C target-feature=+neon` to `rustc`. TL;DR For
faster binaries, use `armv7` if your target has an ARMv7 processor.

If you didn't find a triple that matches your target system, then you are going to need to
[create a target specification file].

[create a target specification file]: #target-specification-files

From this point forwards, I'll use the term **$rustc_target** to refer to the triple you found in
this section. For example, if you found that your target is `arm-unknown-linux-gnueabihf`, then
whenever you see something like `--target=$rustc_target` mentally expand the `$rustc_target` bit so
you end with `--target=arm-unknown-linux-gnueaibhf`.

Similarly, I'll use the **$host** term to refer to the host triple. You can find this triple in the
`rustc -Vv` output under the host field. For example, my host system has triple
`x86_64-unknown-linux-gnu`.

### C cross toolchain

Here things get a little confusing.

`gcc` cross compilers only target a single triple. And this triple is used to prefix all the
toolchain commands: `ar`, `gcc`, etc. This helps to distinguish a tool used for native compilation,
e.g. `gcc`, from a cross compilation tool, e.g. `arm-none-eabi-gcc`.

The confusing part is that triples can be quite arbitrary, so your C cross compiler will most likely
be prefixed with a triple that's different from $rustc_target. For example, in Ubuntu the cross
compiler for ARM devices is packaged as `arm-linux-gnueabihf-gcc`, the same cross compiler is
prefixed as `armv7-unknown-linux-gnueabihf-gcc` in [Exherbo], and `rustc` uses the
`arm-unknown-linux-gnueabihf` triple for that target. None of these triples match, but they refer to
the same set of systems.

[Exherbo]: http://exherbo.org/

The best way to confirm that you have the correct cross toolchain for your target system is to cross
compile a C program, preferably something not trivial, and test executing it on the target system.

As to where to get the C cross toolchain, that will depend on your system. Some Linux distributions
provide packaged cross compilers. In other cases, you'll need to compile the cross compiler
yourself. Tools like [crosstool-ng] can help with that endeavor. For Linux to OSX, check the
[osxcross] project.

[crosstool-ng]: https://github.com/crosstool-ng/crosstool-ng
[osxcross]: https://github.com/tpoechtrager/osxcross

Some examples of packaged cross compilers below:

- For `arm-unknown-linux-gnueabi`, Ubuntu and Debian provide the `gcc-*-arm-linux-gnueabi` packages,
  where `*` is gcc version. Example: `gcc-4.9-arm-linux-gnueabi`
- For `arm-unknown-linux-gnueabihf`, same as above but replace `gnueabi` with `gnueabihf`
- For OpenWRT devices, i.e. targets `mips-unknown-linux-uclibc` (15.05 and older) and
    `mips-unknown-linux-musl` (post 15.05), use the [OpenWRT SDK]
- For the Raspberry Pi, use the [Raspberry tools].

[OpenWRT SDK]: https://wiki.openwrt.org/doc/howto/obtain.firmware.sdk
[Raspberry tools]: https://github.com/raspberrypi/tools/tree/master/arm-bcm2708

Note that the C cross toolchain will ship with a cross compiled libc for your target. Make sure
that:

- The toolchain libc matches the target libc. Example, if your target uses the musl libc, then your
    toolchain must also use the musl libc.
- The toolchain libc is ABI compatible with the target libc. This usually means that the toolchain
    libc must be older than the target libc. Ideally, both the toolchain libc and the target libc
    should have the exact same version.

From this point forwards, I'll use the term **$gcc_prefix** to refer to the prefix of the cross
compilation tools (i.e. the cross toolchain) you installed in this section.

### Cross compiled Rust crates

Most Rust programs link to the `std` crate, so at the very least you'll need a cross compiled `std`
crate to cross compile your program. The easiest way to get it is from the [official builds].

[official builds]: http://static.rust-lang.org/dist/

If you are using multirust, as of 2016-03-08, you can install these crates with a single command:
`multirust add-target nightly $rustc_target`. If you are using rustup.rs, use the command:
`rustup target add $rustc_target`. And if you are using neither, follow the instructions below to
install the crates manually.

The tarball you want is `$date/rust-std-nightly-$rustc_target.tar.gz`. Where `$date` usually matches
with the `rustc` commit date shown in `rustc -V`, although on occasion the dates may differ by one
or a few days.

For example, for a `arm-unknown-linux-gnueabihf` target and a `rustc` with version (`rustc -V`)
`rustc 1.8.0-nightly (3c9442fc5 2016-02-04)` this is the correct tarball:

```
http://static.rust-lang.org/dist/2016-02-04/rust-std-beta-arm-unknown-linux-gnueabihf.tar.gz
```

To install the tarball use the `install.sh` script that's inside the tarball:

```
$ tar xzf rust-std-nightly-arm-unknown-linux-gnueabihf.tar.gz
$ cd rust-std-nightly-arm-unknown-linux-gnueabihf
$ ./install.sh --prefix=$(rustc --print sysroot)
```

**WARNING** The above command will output a message that looks like this: "creating uninstall script
at /some/path/lib/rustlib/uninstall.sh". Do **not** run that script because it will uninstall the
cross compiled standard crates **and** the native standard crates; leaving you with an unusable Rust
installation and you won't be able to compile natively.

If for some reason you need to uninstall the crates you just installed, simply remove the following
directory: `$(rustc --print sysroot)/lib/rustlib/$rustc_target`.

**NOTE** If you are using the nightly channel, every time you update your Rust install you'll have
to install a new set of cross compiled standard crates. To do so, simply download a new tarball and
use the `install.sh` script as before. AFAICT the script will also take care of removing the old set
of crates.

## Cross compiling with `rustc`

This is the easy part!

Cross compiling with `rustc` only requires passing a few extra flags to its invocation:

- `--target=$rustc_target`, tells `rustc` we are cross compiling for `$rustc_target`.
- `-C linker=$gcc_prefix-gcc`, instructs `rustc` to use a cross linker instead of the native one
    (`cc`).

Next, an example to test the cross compilation setup so far:

- Create a hello world program on the host

```
$ cat hello.rs
fn main() {
    println!("Hello, world!");
}
```

- Cross compile the program on the host

```
$ rustc \
    --target=arm-unknown-linux-gnueabihf \
    -C linker=arm-linux-gnueabihf-gcc \
    hello.rs
```

- Run the program on the target

```
$ scp hello me@arm:~
$ ssh me@arm ./hello
Hello, world!
```

## Cross compiling with `cargo`

To cross compile with cargo, we must first use its [configuration system] to set the proper linker
and archiver for the target. Once set, we only need to pass the `--target` flag to cargo commands.

[configuration system]: http://doc.crates.io/config.html

Cargo configuration is stored in a TOML file, the key we are interested in is
`target.$rustc_target.linker`. The value to store in this key is
the same we passed to `rustc` in the previous section. It's up to you to decide if you make this
configuration global or project specific.

Let's go over an example:

- Create a new binary Cargo project.

```
$ cargo new --bin foo
$ cd foo
```

- Add a dependency to the project.

```
$ echo 'clap = "2.0.4"' >> Cargo.toml
$ cat Cargo.toml
[package]
authors = ["me", "myself", "I"]
name = "foo"
version = "0.1.0"

[dependencies]
clap = "2.0.4"
```

- Configure the target linker and archiver only for this project.

```
$ mkdir .cargo
$ cat >.cargo/config <<EOF
> [target.arm-unknown-linux-gnueabihf]
> linker = "arm-linux-gnueabihf-gcc"
> EOF
```

- Write the application

```
$ cat >src/main.rs <<EOF
> extern crate clap;
>
> use clap::App;
>
> fn main() {
>     let _ = App::new("foo").version("0.1.0").get_matches();
> }
> EOF
```

- Build the project for the target

```
$ cargo build --target=arm-unknown-linux-gnueabihf
```

- Deploy the binary to the target

```
$ scp target/arm-unknown-linux-gnueabihf/debug/foo me@arm:~
```

- Run the binary on the target.

```
$ ssh me@arm ./foo -h
foo 0.1.0

USAGE:
        foo [FLAGS]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information
```

## Advanced topics

### Cross compiling the standard crates

Right now, you can only cross compile the standard crates if your target is supported by the Rust
build system (RBS). You can find a list of all the supported targets in the [`mk/cfg`] directory
(**NOTE** linked directory is **not** the latest revision). As of
`rustc 1.8.0-nightly (3c9442fc5 2016-02-04)`, I see the following supported targets:

[`mk/cfg`]: https://github.com/rust-lang/rust/tree/3c9442fc503fe397b8d3495d5a7f9e599ad63cf6/mk/cfg

```
$ ls mk/cfg
aarch64-apple-ios.mk              i686-pc-windows-msvc.mk           x86_64-pc-windows-gnu.mk
aarch64-linux-android.mk          i686-unknown-freebsd.mk           x86_64-pc-windows-msvc.mk
aarch64-unknown-linux-gnu.mk      i686-unknown-linux-gnu.mk         x86_64-rumprun-netbsd.mk
arm-linux-androideabi.mk          le32-unknown-nacl.mk              x86_64-sun-solaris.mk
arm-unknown-linux-gnueabihf.mk    mipsel-unknown-linux-gnu.mk       x86_64-unknown-bitrig.mk
arm-unknown-linux-gnueabi.mk      mipsel-unknown-linux-musl.mk      x86_64-unknown-dragonfly.mk
armv7-apple-ios.mk                mips-unknown-linux-gnu.mk         x86_64-unknown-freebsd.mk
armv7s-apple-ios.mk               mips-unknown-linux-musl.mk        x86_64-unknown-linux-gnu.mk
armv7-unknown-linux-gnueabihf.mk  powerpc64le-unknown-linux-gnu.mk  x86_64-unknown-linux-musl.mk
i386-apple-ios.mk                 powerpc64-unknown-linux-gnu.mk    x86_64-unknown-netbsd.mk
i686-apple-darwin.mk              powerpc-unknown-linux-gnu.mk      x86_64-unknown-openbsd.mk
i686-linux-android.mk             x86_64-apple-darwin.mk
i686-pc-windows-gnu.mk            x86_64-apple-ios.mk
```

**NOTE** If your target is not supported by the RBS, then you'll need to add support for your target
to it. I won't go over the details of adding support for a new target, but you can use [this PR] as
a reference.

[this PR]: https://github.com/rust-lang/rust/pull/31078

**NOTE** If you are doing bare metal programming, building your own kernel or, in general, working
with `#![no_std]` code, then you probably don't want to (and probably can't because there is no OS)
build all the standard crates, but just the `core` crate and other freestanding crates. If that's
your case, read the [Cross compiling `no_std` code] section instead of this one.

[Cross compiling `no_std` code]: #cross-compiling-no_std-code

The steps for cross compiling the standard crates are not complicated, but the process of building
them does take a very long time because the RBS will bootstrap a new compiler, and then use that
bootstrapped compiler to cross compile the crates. Hopefully, the [upcoming] cargo-based build
system will open the possibility of making this much faster by letting you use your already
installed `rustc` and `cargo` to cross compile the standard crates.

[upcoming]: https://github.com/rust-lang/rust/pull/31123

Back to the instructions, first you need to figure out the *commit hash* of your `rustc`. This
is listed under the output of `rustc -Vv`. For example, this `rustc`:

```
$ rustc -Vv
rustc 1.8.0-nightly (3c9442fc5 2016-02-04)
binary: rustc
commit-hash: 3c9442fc503fe397b8d3495d5a7f9e599ad63cf6
commit-date: 2016-02-04
host: x86_64-unknown-linux-gnu
release: 1.8.0-nightly
```

Has commit hash: `3c9442fc503fe397b8d3495d5a7f9e599ad63cf6`.

Next you need to fetch Rust source and check it out at that exact commit hash. Don't omit the
checkout or you'll end with crates that are unusable by your compiler.

```
$ git clone https://github.com/rust-lang/rust
$ cd rust
$ git checkout $rustc_commit_hash
# Triple check the git checkout matches `rustc` commit hash
$ git rev-parse HEAD
$rustc_commit_hash
```

Next we prepare a build directory for an out of source build.

```
# Anywhere
$ mkdir build
$ cd build
$ /path/to/rust/configure --target=$rustc_target
```

`configure` accepts many other configuration flags, check out `configure --help` for more
information. Do note that by default, i.e. without any flag, `configure` will prepare a fully
optimized build.

Next we kick off the build:

```
$ make -j$(nproc)
```

If you hit this error during the build:

```
make[1]: $rbs_prefix-gcc: Command not found
```

Don't `panic!`

This happens because the RBS expects a gcc with a certain prefix for each target, but this prefix
may not match the prefix of your installed cross compiler. For example, in my system, the installed
cross compiler is `armv7-unknown-linux-gnueabihf-gcc`, but the RBS, when building for the
`arm-unknown-linux-gnueabihf` target, expects the cross compiler to be named
`arm-none-gnueabihf-gcc`.

This can be easily fixed with some shim binaries:

```
# In the build directory
$ mkdir .shims
$ cd .shims
$ ln -s $(which $gcc_prefix-ar) $rbs_prefix-ar
$ ln -s $(which $gcc_prefix-gcc) $rbs_prefix-gcc
$ cd ..
$ export PATH=$(pwd)/.shims:$PATH
```

Now you should be able to call both `$gcc_prefix-gcc` and `$rbs_prefix-gcc`. For example:

```
# My installed cross compiler
$ armv7-unknown-linux-gnueabihf-gcc -v
Using built-in specs.
COLLECT_GCC=armv7-unknown-linux-gnueabihf-gcc
COLLECT_LTO_WRAPPER=/usr/x86_64-pc-linux-gnu/libexec/gcc/armv7-unknown-linux-gnueabihf/5.3.0/lto-wrapper
Target: armv7-unknown-linux-gnueabihf
Configured with: (...)
Thread model: posix
gcc version 5.3.0 (GCC)

# The cross compiler that the RBS expects, which is supplied by the .shims directory
$ arm-linux-gnueabihf-gcc -v
Using built-in specs.
COLLECT_GCC=armv7-unknown-linux-gnueabihf-gcc
COLLECT_LTO_WRAPPER=/usr/x86_64-pc-linux-gnu/libexec/gcc/armv7-unknown-linux-gnueabihf/5.3.0/lto-wrapper
Target: armv7-unknown-linux-gnueabihf
Configured with: (...)
Thread model: posix
gcc version 5.3.0 (GCC)
```

You can now resume the build with `make -j$(nproc)`.

Hopefully the build will complete successfully and your cross compiled crates will be available in
the `$host/stage2/lib/rustlib/$rustc_target/lib` directory.

```
# In the build directory
$ ls x86_64-unknown-linux-gnu/stage2/lib/rustlib/arm-unknown-linux-gnueabihf/lib
liballoc-db5a760f.rlib           librand-db5a760f.rlib            stamp.arena
liballoc_jemalloc-db5a760f.rlib  librbml-db5a760f.rlib            stamp.collections
liballoc_system-db5a760f.rlib    librbml-db5a760f.so              stamp.core
libarena-db5a760f.rlib           librustc_bitflags-db5a760f.rlib  stamp.flate
libarena-db5a760f.so             librustc_unicode-db5a760f.rlib   stamp.getopts
libcollections-db5a760f.rlib     libserialize-db5a760f.rlib       stamp.graphviz
libcompiler-rt.a                 libserialize-db5a760f.so         stamp.libc
libcore-db5a760f.rlib            libstd-db5a760f.rlib             stamp.log
libflate-db5a760f.rlib           libstd-db5a760f.so               stamp.rand
libflate-db5a760f.so             libterm-db5a760f.rlib            stamp.rbml
libgetopts-db5a760f.rlib         libterm-db5a760f.so              stamp.rustc_bitflags
libgetopts-db5a760f.so           libtest-db5a760f.rlib            stamp.rustc_unicode
libgraphviz-db5a760f.rlib        libtest-db5a760f.so              stamp.serialize
libgraphviz-db5a760f.so          rustlib                          stamp.std
liblibc-db5a760f.rlib            stamp.alloc                      stamp.term
liblog-db5a760f.rlib             stamp.alloc_jemalloc             stamp.test
liblog-db5a760f.so               stamp.alloc_system
```

The next section will tell you how to install these crates in your Rust installation directory.

### Installing the cross compiled standard crates

First, we need to take a closer look at your Rust installation directory, whose path you can get
with `rustc --print sysroot`:

```
# I'm using rustup.rs, you'll get a different path if you used rustup.sh or your distro package
# manager to install Rust
$ tree -d $(rustc --print sysroot)
~/.multirust/toolchains/nightly
├── bin
├── etc
│   └── bash_completion.d
├── lib
│   └── rustlib
│       ├── etc
│       └── $host
│           └── lib
└── share
    ├── doc
    │   └── (...)
    ├── man
    │   └── man1
    └── zsh
        └── site-functions
```

See that `lib/rustlib/$host` directory? That's where your native crates are stored. The cross
compiled crates must be installed right next to that directory. Following the example from the
previous section, the following command will copy the standard crates built by the RBS in the right
place.

```
# In the 'build' directory
$ cp -r \
    $host/stage2/lib/rustlib/$target
    $(rustc --print sysroot)/lib/rustlib
```

Finally, we check that the crates are in the right place.

```
$ tree $(rustc --print sysroot)/lib/rustlib
/home/japaric/.multirust/toolchains/nightly/lib/rustlib
├── (...)
├── uninstall.sh
├── $host
│  └── lib
│       ├── liballoc-fd663c41.rlib
│       ├── (...)
│       ├── libarena-fd663c41.so
│       └── (...)
└── $target
    └── lib
        ├── liballoc-fd663c41.rlib
        ├── (...)
        ├── libarena-fd663c41.so
        └── (...)
```

This way you can install crates for as many targets as you want. To "uninstall" the crates simply
remove the $target directory.

### Target specification files

A target specification file is a [JSON] file that provides detailed information about a target to
the Rust compiler. This specification file has five required fields and several optional ones. All
its keys are strings and its values are either strings or booleans. A minimal target spec file for
Cortex M3 microcontrollers is shown below:

[JSON]: https://en.wikipedia.org/wiki/JSON

``` json
{
  "0": "NOTE: I'll use these 'numeric' fields as comments, but they shouldn't appear in these files",
  "1": "The next five fields are _required_",
  "arch": "arm",
  "llvm-target": "thumbv7m-none-eabi",
  "os": "none",
  "target-endian": "little",
  "target-pointer-width": "32",

  "2": "These fields are optional. Not all the possible optional fields are listed here, though",
  "cpu": "cortex-m3",
  "morestack": false
}
```

A list of all the possible keys and their effect on compilation can be found in the
[`src/librustc_back/target/mod.rs`] file (**NOTE**: the linked file is **not** the latest revision).

[`src/librustc_back/target/mod.rs`]: https://github.com/rust-lang/rust/blob/3c9442fc503fe397b8d3495d5a7f9e599ad63cf6/src/librustc_back/target/mod.rs#L70-L207

There are two ways to pass these target specification files to `rustc`, the first is pass the full
path via the `--target` flag.

```
$ rustc --target path/to/thumbv7m-none-eabi.json (...)
```

The other is to simply pass the ["file stem"] of the file to `--target`, but then the file must be
in the working directory or in the directory specified by the `RUST_TARGET_PATH` variable.

["file stem"]: http://doc.rust-lang.org/std/path/struct.Path.html#method.file_stem

```
# Target specification file is in the working directory
$ ls thumbv7m-none-eabi.json
thumbv7m-none-eabi.json

# Passing just the "file stem" works
$ rustc --target thumbv7m-none-eabi (...)
```

### Cross compiling `no_std` code

When working with `no_std` code you only want a few freestanding crates like `core`, and you are
probably working with a custom target, e.g. a Cortex-M microcontroller, so there are no official
builds for your target nor can you build these crates using the RBS.

A simple solution to get a cross compiled `core` crate is to make your program/crate depend on the
[`rust-libcore`] crate. This will make Cargo build the `core` crate as part of the `cargo build`
process. However, this approach has two problems:

[`rust-libcore`]: https://crates.io/crates/rust-libcore

- Virality: You can't make your crate depend on another `no_std` crate unless that crate also
    depends on `rust-libcore`.

- If you want your crate to depend on another standard crate then a new `rust-lib$crate` crate would
    need to be created.

An alternative solution that doesn't have these problems is to use a "sysroot" that holds the cross
compiled crates. I'm implementing this approach in [`xargo`]. For more details check the repository.

[`xargo`]: https://github.com/japaric/xargo

### Troubleshooting common problems

> Anything that can go wrong, will go wrong -- Murphy's law

This section: What to do when things go wrong.

#### can't find crate

**Symptom**

```
$ cargo build --target $rustc_target
error: can't find crate for `$crate`
```

**Cause**

`rustc` can't find the cross compiled standard crate `$crate` in your Rust installation directory.

**Solution**

Check the [Installing the cross compiled standard crates] section and make sure the cross compiled
`$crate` crate is in the right place.

[Installing the cross compiled standard crates]: #installing-the-cross-compiled-standard-crates

#### crate incompatible with this version of rustc

**Symptom**

```
$ cargo build --target $rustc_target
error: the crate `$crate` has been compiled with rustc $version-$channel ($hash $date), which is incompatible with this version of rustc
```

**Cause**

The version of the cross compiled standard crates that you installed don't match your `rustc`
version.

**Solution**

If you are on the nightly channel and installed an official build, you probably got the date of the
tarball wrong. Try a different date.

[official build]: http://static.rust-lang.org/dist/

If you cross compiled the crates from source, then you checked out the wrong commit of the source.
You'll have the build the crates again, but making sure you check out the repository at the right
commit (it must match the commit-hash field of `rustc -Vv` output).

#### undefined reference

**Symptom**

```
$ cargo build --target $rustc_target
/path/to/some/file.c:$line: undefined reference to `$symbol`
```

**Cause**

The scenario goes like this:

- The standard crates were cross compiled using a C cross toolchain "A".
- Then you cross compile a Rust program using C cross toolchain "B", this program was also linked to
    the standard crates produced in the previous step.

The problem occurs when the libc component of toolchain "A" is newer than the libc component of
toolchain "B". In this case, the standard crates cross compiled with "A" may depend on libc symbols
that are not available in "B"'s libc.

This error will also occur if "A"'s libc is different from "B"'s libc. Example: toolchain "A" is
`mips-linux-gnu` and toolchain "B" is `mips-linux-musl`.

**Solution**

If you observe this with a [official build], that's a [bug]. It indicates that the Rust team must
downgrade the libc component of the C cross toolchain they are using to build the standard crates.

[bug]: https://github.com/rust-lang/rust/issues/30966

If you are cross compiling the standard crates yourself, then it would be ideal if you use the same
C cross toolchain to build the standard crates and to cross compile Rust programs.

#### can't load library

**Symptom**

```
# On target
$ ./hello
./hello: can't load library 'libpthread.so.0'
```

**Cause**

Your target system is missing a shared library. You can confirm this with `ldd`:

```
# Or `LD_TRACE_LOADED_OBJECTS=1 ./hello` on uClibc-based OpenWRT devices
$ ldd hello
        libdl.so.0 => /lib/libdl.so.0 (0x771ba000)
        libpthread.so.0 => not found
        libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x77196000)
        libc.so.0 => /lib/libc.so.0 (0x77129000)
        ld-uClibc.so.0 => /lib/ld-uClibc.so.0 (0x771ce000)
        libm.so.0 => /lib/libm.so.0 (0x77103000)
```

All the missing libraries are marked with "not found".

**Solution**

Install the missing shared libraries in your target system. Continuing the previous example:

```
# target system is an OpenWRT device
$ opkg install libpthread
$ ./hello
Hello, world!
```

#### `$symbol` not found

**Symptom**

```
# On target
$ ./hello
rustc: /path/to/$c_library.so: version `$symbol' not found (required by /path/to/$rust_library.so).
```

**Cause**

ABI mismatch between the library that was dynamically linked to the binary during cross compilation
and the library that's installed in the target.

**Solution**

Update/change the library on either the host or the target to make them both ABI compatible.
Ideally, the host and the target should have the same library version.

**NOTE** When I say the library on the host, I'm referring to *the cross compiled library* that
the `$prefix_gcc-gcc` is linking into your Rust program. I'm **not** referring to the **native**
library that may be installed in the host.

#### illegal instruction

**Symptom**

```
# on target
$ ./hello
Illegal instruction
```

**Causes**

**NOTE** You can also get an "illegal instruction" error if your program reaches an Out Of Memory
(OOM) condition. In some systems, you will additionally see an "fatal runtime error: out of memory"
message when you hit OOM. If you are sure that's not your case, then this is a cross compilation
problem.

This occurs because your program contains an [instruction] that's not supported by your target
system. Among the possible causes of this problem we have:

[instruction]: https://simple.wikipedia.org/wiki/Instruction_(computer_science)

- You are compiling for a hard float target, e.g. `arm-unknown-linux-gnueabihf`, but your target
    doesn't support hard float operations and it's actually a soft float target, e.g.
    `arm-unknown-linux-gnueabi`. **Solution**: Use the right triple, in this example:
    `arm-unknown-linux-gnueabi`.

- You are using the right soft float triple, e.g. `arm-unknown-linux-gnueabi`, for your target
    system. But your C cross toolchain was compiled with hard float support and is injecting hard
    float instructions into your binary. **Solution**: Get the correct toolchain, one that was built
    with soft float support. Hint: look for the flag `--with-float` in the output of
    `$gcc_prefix-gcc -v`.

## FAQ

### I want to build binaries for Linux, Mac and Windows. How do I cross compile from Linux to Mac?

Short answer: You don't.

It's hard to find a cross C toolchain (and cross compiled C libraries) between different OSes
(except perhaps from Linux to Windows). A much simpler and less error prone way is to build natively
for these targets because they are [tier 1] platforms. You may not have direct access to all these
OSes but that's not a problem because you can use CI services like [Travis CI] and [AppVeyor]. Check
my [rust-everywhere] project for instructions on how to do that.

[tier 1]: https://doc.rust-lang.org/book/getting-started.html#tier-1
[Travis CI]: https://travis-ci.org/
[AppVeyor]: https://www.appveyor.com/
[rust-everywhere]: https://github.com/japaric/rust-everywhere

### How do I compile a fully statically linked Rust binaries?

Short answer: `cargo build --target x86_64-unknown-linux-musl`

For targets of the form `*-*-linux-gnu*`, `rustc` always produces binaries dynamically linked to
`glibc` and other libraries:

```
$ cargo new --bin hello
$ cargo build --target x86_64-unknown-linux-gnu
$ file target/x86_64-unknown-linux-gnu/debug/hello
target/x86_64-unknown-linux-gnu/debug/hello: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /usr/x86_64-pc-linux-gnu/lib/ld-linux-x86-64.so.2, for GNU/Linux 2.6.34, BuildID[sha1]=a3fa7281e9ded30372b5131a2feb6f1e78a6f1cd, not stripped
$ ldd target/x86_64-unknown-linux-gnu/debug/hello
        linux-vdso.so.1 (0x00007fff58bf4000)
        libdl.so.2 => /usr/x86_64-pc-linux-gnu/lib/libdl.so.2 (0x00007fc4b2d3f000)
        libpthread.so.0 => /usr/x86_64-pc-linux-gnu/lib/libpthread.so.0 (0x00007fc4b2b22000)
        libgcc_s.so.1 => /usr/x86_64-pc-linux-gnu/lib/libgcc_s.so.1 (0x00007fc4b290c000)
        libc.so.6 => /usr/x86_64-pc-linux-gnu/lib/libc.so.6 (0x00007fc4b2568000)
        /usr/x86_64-pc-linux-gnu/lib/ld-linux-x86-64.so.2 (0x00007fc4b2f43000)
        libm.so.6 => /usr/x86_64-pc-linux-gnu/lib/libm.so.6 (0x00007fc4b2272000)
```

To produce statically linked binaries, Rust provides two targets:
`x86_64-unknown-linux-musl` and `i686-unknown-linux-musl`. The binaries produced for these targets
are statically linked to the MUSL C library. Example below:

```
$ cargo new --bin hello
$ cd hello
$ rustup target add x86_64-unknown-linux-musl
$ cargo build --target x86_64-unknown-linux-musl
$ file target/x86_64-unknown-linux-musl/debug/hello
target/x86_64-unknown-linux-musl/debug/hello: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, BuildID[sha1]=759d41b9a78d86bff9b6529d12c8fd6b934c0088, not stripped
$ ldd target/x86_64-unknown-linux-musl/debug/hello
        not a dynamic executable
```

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or
  http://www.apache.org/licenses/LICENSE-2.0)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the
work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any
additional terms or conditions.
