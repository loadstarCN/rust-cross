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

本指南分为两个部分：“正文”和高级主题。 正文包括最简单的情况：将依赖于`std` crate 的Rust程序交叉编译为“受支持目标系统”（提供官方版本）。高级主题部分介绍了`no_std`程序，目标系统specifications文件，如何交叉编译“standard crates”和常见故障排除。

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
    - [安装交叉编译的standard crates](#installing-the-cross-compiled-standard-crates)
    - [目标系统描述文件](#target-specification-files)
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

一些triple省略了供应商或abi组件，因此它们实际上是真正的“三元组”。其中一个例子是 `x86_64-apple-darwin`, 这里:

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

 **NOTE** `arm-unknown-linux-gnueabihf` 和 `armv7-unknown-linux-gnueabihf` 之间的区别是： `arm` triple包括ARMv6和armv7处理器，而 `armv7` 只支持armv7处理器。因此，`armv7` triple支持仅适用于ARMv7处理器。如果你使用 `arm` triple，你就必须通过向 `rustc` 传递诸如 `-C target-feature=+neon` 这样的额外标志来选择这些优化。想要要获得更快的二进制文件，如果目标具有armv7处理器，请使用 `armv7`。

如果找不到与目标系统匹配的triple，则需要[创建目标描述文件]。

[创建目标描述文件]: #target-specification-files

From this point forwards, I'll use the term **$rustc_target** to refer to the triple you found in
this section. For example, if you found that your target is `arm-unknown-linux-gnueabihf`, then
whenever you see something like `--target=$rustc_target` mentally expand the `$rustc_target` bit so
you end with `--target=arm-unknown-linux-gnueaibhf`.

Similarly, I'll use the **$host** term to refer to the host triple. You can find this triple in the
`rustc -Vv` output under the host field. For example, my host system has triple
`x86_64-unknown-linux-gnu`.


从现在开始，我将使用术语 **$rustc_target** 来代指您在本节中找到的triple。例如，如果您发现您的目标系统是 `arm-unknown-linux-gnueabihf`，那么每当您看到诸如 `--target=$rustc_target` 之类的内容时，请自动脑补替换 `$rustc_target` 位，将其认为是 `--target=arm-unknown-linux-gnueaibhf`。

类似地，我将使用 **$host** 术语来指代本地系统triple。您可以在命令 `rustc -Vv` 的输出中找到这个triple。例如，我的主机系统的triple是 `x86_64-unknown-linux-gnu`。


### C 交叉编译工具链

从这里开始事情有点复杂起来。

`gcc` 的交叉编译器只针对单个triple。这个triple用于前缀所有toolchain命令：`ar`, `gcc`等。这有助于区分用于本机编译的工具（如 `gcc`）和交叉编译工具（如`arm-none-eabi-gcc`）。

The confusing part is that triples can be quite arbitrary, so your C cross compiler will most likely
be prefixed with a triple that's different from $rustc_target. For example, in Ubuntu the cross
compiler for ARM devices is packaged as `arm-linux-gnueabihf-gcc`, the same cross compiler is
prefixed as `armv7-unknown-linux-gnueabihf-gcc` in [Exherbo], and `rustc` uses the
`arm-unknown-linux-gnueabihf` triple for that target. None of these triples match, but they refer to
the same set of systems.

令人困惑的是，triples可能是非常任意的，所以C交叉编译器很可能会以不同于$rustc_target的三元组作为前缀。例如，在Ubuntu中，ARM设备的交叉编译器被打包为 `arm-linux-gnueabihf-gcc` ，相同的交叉编译器在[Exherbo]中前缀为 `armv7-unknown-linux-gnueabihf-gcc`，并且 `rustc` 对该目标使用 `arm-unknown-linux-gnueabihf` triple。这些triple都不匹配，但它们引用的是同一组系统。

[Exherbo]: http://exherbo.org/

确认目标系统具有正确的交叉编译工具链的最佳方法是交叉编译C程序（最好是一些不重要的程序），并在目标系统上测试执行它。

至于在哪里得到C交叉编译工具链，这将取决于您的系统。一些Linux发行版提供了打包的交叉编译器。在其他情况下，您需要自己编译交叉编译器。像[crosstool ng]这样的工具可以帮助您实现这一目标。对于Linux到OSX，请查看[osxcross]项目。

[crosstool-ng]: https://github.com/crosstool-ng/crosstool-ng
[osxcross]: https://github.com/tpoechtrager/osxcross

下面是打包好的交叉编译器的一些示例：

- 对于 `arm-unknown-linux-gnueabi`, Ubuntu和Debian提供了 `gcc-*-arm-linux-gnueabi` 包,
  其中 `*` 是gcc版本。例如： `gcc-4.9-arm-linux-gnueabi`
- 对于 `arm-unknown-linux-gnueabihf`, 同上，但将 `gnueabi` 替换为 `gnueabihf`
- 对于OpenWRT设备，即目标系统为 `mips-unknown-linux-uclibc` (15.05及更高版本) 和
    `mips-unknown-linux-musl` (15.05之后), 使用 [OpenWRT SDK]
- 对于Raspberry Pi（树莓派）, 使用 [Raspberry tools].

[OpenWRT SDK]: https://wiki.openwrt.org/doc/howto/obtain.firmware.sdk
[Raspberry tools]: https://github.com/raspberrypi/tools/tree/master/arm-bcm2708

请注意，C交叉编译工具链将为您的目标系统提供一个交叉编译的libc。确保：

- 工具链libc与目标系统libc匹配。例如，如果目标系统使用musl libc，则工具链必须使用musl libc。
- 工具链libc与目标系统libc兼容。这通常意味着工具链libc版本必须低于于目标系统libc。理想情况下，工具链libc和目标系统libc应该具有完全相同的版本。

从现在开始，我将使用术语 **$gcc_prefix** 来指代您在本节中安装的交叉编译工具（即交叉编译工具链）的前缀。

### 交叉编译 Rust crates

大多数Rust程序都链接到 `std` crate，因此至少需要交叉编译的 `std`
crate 来交叉编译程序。最简单的方法来自[官方版本]。

[官方版本]: http://static.rust-lang.org/dist/

如果您使用的是multirust，从2016-03-08开始，您可以使用单条命令安装这些crates：`multirust add-target nightly $rustc_target`。如果您使用的是rustup.rs，请使用命令：`rustup target add $rustc_target`。如果两者都不想使用，请按照下面的说明手动安装crates。

你想要的tarball是 `$date/rust-std-nightly-$rustc_target.tar.gz`。其中，`$date`通常与 `rustc -V` 中显示的 `rustc` 提交日期匹配，尽管有时日期可能相差一天或几天。

例如，对于 `arm-unknown-linux-gnueabihf` 目标系统，`rustc`具有版本（`rustc-V`） `rustc 1.8.0-nightly (3c9442fc5 2016-02-04)`。正确的tarbll是：

```
http://static.rust-lang.org/dist/2016-02-04/rust-std-beta-arm-unknown-linux-gnueabihf.tar.gz
```
要安装该tarball，请使用tarball中的 `install.sh` 脚本：

```
$ tar xzf rust-std-nightly-arm-unknown-linux-gnueabihf.tar.gz
$ cd rust-std-nightly-arm-unknown-linux-gnueabihf
$ ./install.sh --prefix=$(rustc --print sysroot)
```

**WARNING**上述命令将输出如下消息："creating uninstall script
at /some/path/lib/rustlib/uninstall.sh"。**不要** 运行该脚本，因为它将卸载交叉编译的standard crates 以及本地系统的standard crates；留下一个不可用的Rust安装，您将无法进行本机编译。

如果出于某种原因需要卸载刚刚安装的crates，只需删除以下目录：`$(rustc --print sysroot)/lib/rustlib/$rustc_target`。

**NOTE**如果您使用nightly channel，每次更新Rust安装时，都必须安装一组新的交叉编译standard crates。为此，只需下载一个新tarball并像以前一样使用 `install.sh` 脚本。脚本将自动负责删除旧的crates。


## 使用 `rustc` 交叉编译

下面部分开始变得容易！

使用 `rustc` 交叉编译只需要向其调用传递几个额外的flags：

- `--target=$rustc_target`, 告诉 `rustc` 我们正在为 `$rustc_target` 交叉编译。
- `-C linker=$gcc_prefix-gcc`,指示 `rustc` 使用交叉链接器而不是本机的（`cc`）。

接下来是一个测试交叉编译设置的示例：

- 在本地系统上创建hello world程序

```
$ cat hello.rs
fn main() {
    println!("Hello, world!");
}
```

- 在本地系统上交叉编译程序

```
$ rustc \
    --target=arm-unknown-linux-gnueabihf \
    -C linker=arm-linux-gnueabihf-gcc \
    hello.rs
```

- 在目标系统上运行程序

```
$ scp hello me@arm:~
$ ssh me@arm ./hello
Hello, world!
```

## 使用 `cargo` 交叉编译

要用cargo交叉编译，我们必须首先使用它的[配置系统]为目标系统设置正确的linker和archiver。一旦设置好，我们只需要将 `--target` 标志传递给cargo命令。

[配置系统]: http://doc.crates.io/config.html

Cargo配置保存在一个TOML文件中，我们感兴趣的配置关键字是`target.$rustc_target.linker`。存储在此关键字中的值与我们在上一节中传递给 `rustc` 的值相同。由您决定是将此配置设置为全局配置还是特定于项目。

让我们看一个例子：

- -创建新的二进制Cargo项目。

```
$ cargo new --bin foo
$ cd foo
```

- 向项目添加依赖项。

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

- 仅为此项目配置目标系统linker和archiver。

```
$ mkdir .cargo
$ cat >.cargo/config <<EOF
> [target.arm-unknown-linux-gnueabihf]
> linker = "arm-linux-gnueabihf-gcc"
> EOF
```

- 编写应用程序

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

- 为目标系统构建项目

```
$ cargo build --target=arm-unknown-linux-gnueabihf
```

- 将二进制文件部署到目标系统

```
$ scp target/arm-unknown-linux-gnueabihf/debug/foo me@arm:~
```

- 在目标系统上运行二进制文件。

```
$ ssh me@arm ./foo -h
foo 0.1.0

USAGE:
        foo [FLAGS]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information
```

## 高级主题

### 交叉编译standard crate

现在，如果目标系统受Rust构建系统（RBS）支持，可以只交叉编译standard crates。您可以在[`mk/cfg`]目录中找到所有受支持目标的列表（**NOTE** 下面的链接是**不是**最新版本）。截至`rustc 1.8.0-nightly (3c9442fc5 2016-02-04)`，我看到以下支持的目标：

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

**NOTE** 如果RBS不支持您的目标系统，则需要在其中添加对目标系统的支持。我不会详细介绍为新目标系统添加支持的细节，但您可以使用[this PR]作为参考。

[this PR]: https://github.com/rust-lang/rust/pull/31078

**NOTE** 如果您正在进行裸机编程、构建自己的内核，或者通常使用 `#![no_std]` 代码，那么您可能不想（也可能不能，因为没有操作系统）构建所有的standard crates，而只构建 `core` crate和其他独立crate。如果是这种情况，请阅读[交叉编译'no_std'代码]部分而不是本部分。

[交叉编译'no_std'代码]: #cross-compiling-no_std-code

交叉编译standard crates的步骤并不复杂，但是构建它们的过程确实需要很长时间，因为RBS将引导一个新的编译器，然后使用引导的编译器交叉编译crates。希望[即将推出的]cargo-based的构建系统能够让您使用已经安装的 `rustc` 和 `cargo` 交叉编译standard crates，从而更快地实现这一点。

[即将推出的]: https://github.com/rust-lang/rust/pull/31123

回到步骤说明中来，首先需要找出 `rustc` 的*commit hash*。这个显示在 `rustc -Vv`的输出下。例如：

```
$ rustc -Vv
rustc 1.8.0-nightly (3c9442fc5 2016-02-04)
binary: rustc
commit-hash: 3c9442fc503fe397b8d3495d5a7f9e599ad63cf6
commit-date: 2016-02-04
host: x86_64-unknown-linux-gnu
release: 1.8.0-nightly
```

commit hash是: `3c9442fc503fe397b8d3495d5a7f9e599ad63cf6`.

接下来，您需要获取Rust源并使用该commit hash来checkout。不要省略checkout步骤，否则最后编译器会返回无法使用的crates。

```
$ git clone https://github.com/rust-lang/rust
$ cd rust
$ git checkout $rustc_commit_hash
# Triple check the git checkout matches `rustc` commit hash
$ git rev-parse HEAD
$rustc_commit_hash
```

接下来，我们将创建一个build目录。

```
# Anywhere
$ mkdir build
$ cd build
$ /path/to/rust/configure --target=$rustc_target
```
`configure` 接受许多其他配置flag，查看 `configure --help`了解更多信息。请注意，默认情况下，即没有任何flag，`configure` 也将准备完全优化构建。


接下来我们开始构建：

```
$ make -j$(nproc)
```

如果在生成过程中遇到此错误：

```
make[1]: $rbs_prefix-gcc: Command not found
```

不要 `panic!`

这是因为RBS希望每个目标都有一个具有特定前缀的gcc，但是这个前缀可能与您安装的交叉编译器的前缀不匹配。例如，在我的系统中，安装的交叉编译器是 `armv7-unknown-linux-gnueabihf-gcc`，但是RBS在为 `arm-unknown-linux-gnueabihf` 标构建时，希望交叉编译器命名为`arm-none-gnueabihf-gcc`。

使用下面的方法可以很容易地解决这个问题：

```
# In the build directory
$ mkdir .shims
$ cd .shims
$ ln -s $(which $gcc_prefix-ar) $rbs_prefix-ar
$ ln -s $(which $gcc_prefix-gcc) $rbs_prefix-gcc
$ cd ..
$ export PATH=$(pwd)/.shims:$PATH
```
现在您应该可以同时调用 `$gcc_prefix-gcc` 和 `$rbs_prefix-gcc`。例如：

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

现在可以使用 `make -j$(nproc)` 继续生成。

祈祷构建将成功完成之后，交叉编译的crates将出现在 `$host/stage2/lib/rustlib/$rustc_target/lib` 目录中。


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

下一节将告诉您如何在Rust安装目录中安装这些crates。

### 安装交叉编译的standard crates

首先，我们需要仔细地查看Rust安装目录，使用 `rustc --print sysroot`可以找到它的路径：

```
# 我使用rustup.rs，如果使用rustup.sh或发行版包管理器安装Rust，您将得到一个不同的路径
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

看到 `lib/rustlib/$host` 目录了吗？那是你的本地crates存放的地方。交叉编译的crates必须安装在该目录的同级目录。按照上一节中的示例，下面的命令将在正确的位置复制RBS构建的standard crates。

```
# 在 'build' 目录中
$ cp -r \
    $host/stage2/lib/rustlib/$target
    $(rustc --print sysroot)/lib/rustlib
```

最后，我们检查crates是否在正确的位置。

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

这样你可以为你想要的目标安装crates。要卸载crates，请简单地删除$target目录。

### 目标系统描述文件

目标系统描述文件是一个[JSON]文件，它向Rust编译器提供有关目标系统的详细信息。此描述文件有五个必需字段和几个可选字段。它的所有关键字都是字符串，其值要么是字符串，要么是布尔值。Cortex M3微控制器的最小目标系统描述文件如下所示：

[JSON]: https://en.wikipedia.org/wiki/JSON

``` json
{
  "0": "NOTE: 我将使用这些'数字'字段作为注释，但它们不应出现在这些文件中",
  "1": "接下来的五个字段是 _必需的_",
  "arch": "arm",
  "llvm-target": "thumbv7m-none-eabi",
  "os": "none",
  "target-endian": "little",
  "target-pointer-width": "32",

  "2": "这些字段是可选的。但并非所有可能的可选字段都列在这里",
  "cpu": "cortex-m3",
  "morestack": false
}
```

所有可能的关键字及其对编译的影响的列表可以在[`src/librustc_back/target/mod.rs`]文件中找到（**NOTE**：该链接**不是**最新版本）。

[`src/librustc_back/target/mod.rs`]: https://github.com/rust-lang/rust/blob/3c9442fc503fe397b8d3495d5a7f9e599ad63cf6/src/librustc_back/target/mod.rs#L70-L207

有两种方法可以将这些目标系统描述文件传递给 `rustc`，第一种是通过 `--target` flag传递完整路径。

```
$ rustc --target path/to/thumbv7m-none-eabi.json (...)
```
另一种方法是简单地将文件的["file stem"]传递给 `--target`，但该文件必须位于工作目录或由 `RUST_TARGET_PATH` 变量指定的目录中。

["file stem"]: http://doc.rust-lang.org/std/path/struct.Path.html#method.file_stem

```
# Target specification file is in the working directory
$ ls thumbv7m-none-eabi.json
thumbv7m-none-eabi.json

# Passing just the "file stem" works
$ rustc --target thumbv7m-none-eabi (...)
```

### 交叉编译 `no_std` 代码

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
