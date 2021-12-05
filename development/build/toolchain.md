<!-- # Build Clang toolchain

Fuchsia is using Clang as the official compiler. -->
# 构建 Clang 工具链

Fuchsia 采用 Clang 为官方的编译器。
<!-- 
## Prerequisites

You need [CMake](https://cmake.org/download/) version 3.13.4 or newer to
execute these commands. This is the [minimum required version](https://reviews.llvm.org/rGafa1afd4108)
to build LLVM. -->
## 前提要求

要执行这些命令，您需要安装 3.13.4 或更高版本的 [CMake](https://cmake.org/download/)。这里可以找到编译 LLVM 的[最低版本要求](https://reviews.llvm.org/rGafa1afd4108)

<!-- While CMake supports different build systems, it is recommended to use
[Ninja](https://github.com/ninja-build/ninja/releases). -->
尽管 CMake 支持多种不同的构建系统，但推荐使用[Ninja](https://github.com/ninja-build/ninja/releases)。

<!-- Both should be present in your Fuchsia checkout as prebuilts. The commands below
assume that `cmake` and `ninja` are in your `PATH`: -->
在检出的 Fuchsia 预构建系统中，CMake 和 Ninja 必须存在。以下的命令假定在环境变量 `PATH` 相关的路径里能找到 `cmake` 和 `ninja`。

```
export PATH=${FUCHSIA}/prebuilt/third_party/cmake/${platform}/bin:${PATH}
export PATH=${FUCHSIA}/prebuilt/third_party/ninja/${platform}/bin:${PATH}
```

<!-- ### Getting Source

The example commands below use `${LLVM_SRCDIR}` to refer to the root of
your LLVM source tree checkout. You can use the official monorepo
[https://github.com/llvm/llvm-project](https://github.com/llvm/llvm-project)
maintained by the LLVM community: -->
### 获取源代码

以下示例代码使用 `${LLVM_SRCDIR}` 指代您下载的 LLVM 源代码树的根目录。您可以从 LLVM 社区维护的官方仓库[https://github.com/llvm/llvm-project](https://github.com/llvm/llvm-project)中下载。

```bash
LLVM_SRCDIR=${HOME}/llvm/llvm-project
git clone https://github.com/llvm/llvm-project ${LLVM_SRCDIR}
```

<!-- Note: It is recommended checking out to the revision that's currently used for
Fuchsia.
The latest upstream revision may be broken or fail to build Fuchsia, whereas it is
guaranteed that the prebuilt revision can always build Fuchsia. This
revision can be found in `[//integration/prebuilts]`. Search for the package
`fuchsia/third_party/clang/${platform}`, and checkout the `git_revision`
associated with it. -->
注意：建议下载当前为 Fuchsia 所用的版本。采用最新上游版本构建 Fuchsia 有可能失败。但预构建版本能保证总是能够
成功构建 Fuchsia。 这个版本可以在 `[//integration/prebuilts]` 里找到。查找 `fuchsia/third_party/clang/${platform}` 包，并下载与之关联的 `git_revision`。

```bash
cd ${LLVM_SRCDIR}
git checkout ${REVISON_NUMBER}
```

<!-- ### Fuchsia IDK

Before building the runtime libraries that are built along with the
toolchain, you need a Fuchsia [IDK](/docs/development/idk)
(formerly known as the SDK).
The IDK must be located in the directory pointed to by the `${IDK_DIR}`
variable: -->
### Fuchsia IDK

在构建随工具链同时构建的运行时库之前，需要 Fuchsia 的[IDK](/docs/development/idk)（也就是之前说的SDK）。
IDK 必须在 `${IDK_DIR}` 变量指向的目录中存在。

```bash
IDK_DIR=${HOME}/fuchsia-idk
```

<!-- To download the latest IDK, you can use the following: -->
可以使用以下命令下载最新 IDK：

```bash
# For Linux
cipd install fuchsia/sdk/core/linux-amd64 latest -root ${IDK_DIR}

# For macOS
cipd install fuchsia/sdk/core/mac-amd64 latest -root ${IDK_DIR}
```

<!-- ### Sysroot for Linux

To include compiler runtimes and C++ library for Linux, download the sysroot
for both `arm64` and `x64`. Both sysroots must be located in
the directory pointed by the `${SYSROOT_DIR}` variable. -->
### Linux 的 Sysroot 工具

下载 `arm64` 和 `x64` 版本的 sysroot 以使 Linux 包含编译器和 C++ 库。这些需要在变量
`${SYSROOT_DIR}` 指向的目录中存在。


```bash
SYSROOT_DIR=${HOME}/fuchsia-sysroot/
```

<!-- To download the latest sysroots, you can use the following: -->
可以使用如下命令下载最新的 sysroot 工具。

```bash
cipd install fuchsia/sysroot/linux-arm64 latest -root ${SYSROOT_DIR}/linux-arm64
cipd install fuchsia/sysroot/linux-amd64 latest -root ${SYSROOT_DIR}/linux-x64
```

{% dynamic if user.is_googler %}

<!-- ### [Googlers only] Goma -->
### [谷歌用户专享] Goma

<!-- Goma is a service for accelerating builds by distributing compilations across
many machines. Googlers should ensure Goma is installed on your machine for faster
builds. If you have Goma installed in `${GOMA_DIR}` (which should be provided in
`//prebuilt/third_party/goma/${platform}`),
you can enable Goma by adding these extra CMake flags to your CMake invocation: -->
Goma 是一种通过在多台机器上进行分布式编译以加速构建过程的服务。要加速构建，谷歌用户需要确保
Goma 已经在您的机器上安装好。假如您将 Goma 安装在 `${GOMA_DIR}` （应该在`//prebuilt/third_party/goma/${platform}`里提供了），您可以通过使用以下额外标记调用 CMake 以启用 Goma：

```bash
  -DCMAKE_C_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DCMAKE_CXX_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
```

<!-- Then you can take advantage of Goma by allowing multiple `ninja` jobs to run in
parallel: -->
之后您就可以利用 Goma 提供的便利允许多个 `ninja` 任务并行执行：

```bash
ninja -j1000
```

<!-- Use `-j100` for Goma on macOS and `-j1000` for Goma on Linux. You may
need to tune the job count to suit your particular machine and workload. -->
在 macOS 系统上使用 `-j100` 及在 Linux 系统上使用 `-j1000` ，您需要视设备及负载
将任务数量调整到合适的值。


<!-- Warning: The examples below assume you can use Goma. If you cannot use Goma, do not
add the provided CMake flags or use an absurdly high number of jobs. -->
警告：以下示例假定您可以使用 Goma。如果还不能使用 Goma, 请勿添加相应的 CMake 标记或者随意一个极高的
任务数。

<!-- Note: In order to use Goma, you need a host compiler that is
supported by Goma such as the Fuchsia Clang installation.
To verify your compiler is available on Goma, you can set
`GOMA_USE_LOCAL=0 GOMA_FALLBACK=0` environment variables. If the
compiler is not available, you will see an error. -->
注意：您需要有一台 Goma 支持的编译主机，如安装了 Fuchsia Clang 的主机，
才能使用 Goma。 可以通过设置 `GOMA_USE_LOCAL=0 GOMA_FALLBACK=0` 环境变量
来验证编译器是否支持 Goma，如果不支持，您将看到一条错误信息。

{% dynamic endif %}

<!-- ## Building a Clang Toolchain for Fuchsia -->
## 为 Fuchsia 构建 Clang 工具链

<!-- The Clang CMake build system supports bootstrap (aka multi-stage)
builds. Fuchsia uses [two-stage bootstrap build](#two-stage-build) for the
Clang compiler.
However, for toolchain related development it is recommended to use
the [single-stage build](#single-stage-build). -->
Clang 的 CMake 构建系统支持自举（多阶段）构建。Fuchsia 利用 Clang 编译器的
[两阶段自举构建](#two-stage-build)。尽管如此，还是推荐使用[单阶段构建](#single-stage-build)
进行工具链相关的开发，

<!-- If your goal is to experiment with clang, the single-stage build is likely what you are looking for.
The first stage compiler is a host-only compiler with some options set
needed for the second stage. The second stage compiler is the fully
optimized compiler intended to ship to users. -->
如果您只是想试试 clang, 单阶段构建极可能就是您要找的东西。第一阶段的编译器是一个
具有某些第二阶段所需的选项集的单机编译器。第二阶段编译器是完全优化的旨在提供给用户的编译器。

<!-- Setting up these compilers requires a lot of options. To simplify the
configuration the Fuchsia Clang build settings are contained in CMake
cache files, which are part of the Clang codebase (`Fuchsia.cmake` and
`Fuchsia-stage2.cmake`). -->
配置这些编译器需要大量选项。为简化这些配置，CMake 缓存文件包含了 Fuchsia Clang 构建配置，
CMake 缓存文件是 Clang 代码库（`Fuchsia.cmake` 和 `Fuchsia-stage2.cmake`）的一部分。

<!-- In the following CMake invocations, `${CLANG_TOOLCHAIN_PREFIX}` refers to the directory
of binaries from a previous Clang toolchain. Normally, this refers to the
current toolchain shipped with Fuchsia, but any references to binaries
from this directory could theoretically be replaced with one's own binaries. -->
在以下的 CMake 调用中，`${CLANG_TOOLCHAIN_PREFIX}` 代表前一个 Clang 工具链产生的二进制文件的目录。
通常，这指当前随 Fuchsia 一起提供的工具链。但从此目录引用的任何二进制文件，
理论上都可以被替换为各自的二进制文件。

```bash
# FUCHSIA_SRCDIR refers to the root directory of your Fuchsia source tree
CLANG_TOOLCHAIN_PREFIX=${FUCHSIA_SRCDIR}/prebuilt/third_party/clang/linux-x64/bin/
```

<!-- Note: Clang must be built in a separate build directory. The directory itself
can be a subdirectory or in a whole other path. -->
注意：Clang 须在单独的目录构建。可以是一个子目录或者另一个路径。

```bash
mkdir llvm-build
mkdir llvm-install  # For placing stripped binaries here
INSTALL_DIR=${pwd}/llvm-install
cd llvm-build
```

<!-- ### Single Stage Build Fuchsia Configuration {#single-stage-build} -->
### 单阶段构建 Fuchsia 的配置 {#single-stage-build}

<!-- When developing Clang for Fuchsia, you can use the cache file to
test the Fuchsia configuration, but run only the second stage, with LTO
disabled, which gives you a faster build time suitable even for
incremental development, without having to manually specify all options: -->
在为 Fuchsia 开发 Clang 时，您可以使用缓存文件检查 Fuchsia 的配置，
但只运行禁用了 LTO 特性的第二阶段，无需手动设置所有选项，能使您更快地构建系统。这同样适用于增量开发。

```bash
cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang \
  -DCMAKE_CXX_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang++ \
  -DCMAKE_ASM_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang \
  -DCMAKE_C_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DCMAKE_CXX_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DCMAKE_ASM_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DLLVM_ENABLE_LTO=OFF \
  -DLINUX_x86_64-unknown-linux-gnu_SYSROOT=${SYSROOT_DIR}/linux-x64 \
  -DLINUX_aarch64-unknown-linux-gnu_SYSROOT=${SYSROOT_DIR}/linux-arm64 \
  -DFUCHSIA_SDK=${IDK_DIR} \
  -DCMAKE_INSTALL_PREFIX= \
  -C ${LLVM_SRCDIR}/clang/cmake/caches/Fuchsia-stage2.cmake \
  ${LLVM_SRCDIR}/llvm
ninja distribution  -j1000  # Build the distribution
```

<!-- If the above fails with an error related to Ninja, then you may need to add
`ninja` to your PATH. You can find the prebuilt executable at
`//prebuilt/third_party/ninja/${platform}/bin`. -->
如果以上命令出现一个 Ninja 相关的错误，那需要将 `ninja` 添加入 PATH 环境变量。预编译好的
可执行文件可以在 `//prebuilt/third_party/ninja/${platform}/bin` 找到。

<!-- `ninja distribution` should be enough for building all binaries, but the Fuchsia
build assumes some libraries are stripped so `ninja
install-distribution-stripped` is necessary. -->
编译所有的二进制文件用 `ninja distribution` 命令应该足够了，但 Fuchsia 构建假定一些库没有安装，
因此有必要执行 `ninja install-distribution-stripped`。

<!-- Caution: Due to a [bug in Clang](https://bugs.llvm.org/show_bug.cgi?id=44097),
builds with assertions enabled might crash while building Fuchsia. As a
workaround, you can disable Clang assertions by setting
`-DLLVM_ENABLE_ASSERTIONS=OFF` or using a release build
(`-DCMAKE_BUILD_TYPE=Release`). -->
警告：由于[Clang 的缺陷](https://bugs.llvm.org/show_bug.cgi?id=44097)，构建 Fuchsia 时，如果开启
断言，可能会导致构建失败。可以通过设置 `-DLLVM_ENABLE_ASSERTIONS=OFF` 以关闭 Clang 的断言或者使用
发布版本（`-DCMAKE_BUILD_TYPE=Release`）以避免这个错误。

<!-- ### Two-Stage Build Fuchsia Configuration {#two-stage-build} -->
### Fuchsia 的两阶段构建配置 {#two-stage-build} 

<!-- This is roughly equivalent to what is run on the prod builders and used to build
a toolchain that Fuchsia ships to users. -->
这大致等同于在产品级构建上运行的内容，也是用于构建 Fuchsia 发布给用户的工具链。

```bash
cmake -GNinja \
  -DCMAKE_C_COMPILER=${CLANG_TOOLCHAIN_PREFIX}/clang \
  -DCMAKE_CXX_COMPILER=${CLANG_TOOLCHAIN_PREFIX}/clang++ \
  -DCMAKE_ASM_COMPILER=${CLANG_TOOLCHAIN_PREFIX}/clang \
  -DCMAKE_C_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DCMAKE_CXX_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DCMAKE_ASM_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DCMAKE_INSTALL_PREFIX= \
  -DSTAGE2_LINUX_aarch64-unknown-linux-gnu_SYSROOT=${SYSROOT_DIR}/linux-arm64 \
  -DSTAGE2_LINUX_x86_64-unknown-linux-gnu_SYSROOT=${SYSROOT_DIR}/linux-amd64 \
  -DSTAGE2_FUCHSIA_SDK=${IDK_DIR} \
  -C ${LLVM_SRCDIR}/clang/cmake/caches/Fuchsia.cmake \
  ${LLVM_SRCDIR}/llvm
ninja stage2-distribution -j1000
DESTDIR=${INSTALL_DIR} ninja stage2-install-distribution-stripped -j1000
```

<!-- Note: The second stage build uses LTO (Link Time Optimization) to
achieve better runtime performance of the final compiler. LTO often
requires a large amount of memory and is very slow. Therefore it may not
be very practical for day-to-day development. -->
注意：第二阶段构建用到了 LTO（链接时优化）以使最终编译器达到更优的运行时性能。
LTO 往往需要大量的内存，并且十分缓慢。因此它可能并不适用于日常开发。

### runtime.json

<!-- If the Fuchsia build fails due to a missing `runtime.json` file, you can copy
them over from the prebuilt toolchain. -->
如果因为找不到 `runtime.json` 而导致 Fuchsia 构建失败，您可以尝试从预构建的工具链中复制一份。

```
cp ${FUCHSIA_SRCDIR}/prebuilt/third_party/clang/linux-x64/lib/runtime.json ${INSTALL_DIR}/lib/
```

<!-- This file contains relative paths used by the Fuchsia build to know where
various libraries from the toolchain are located. -->
该文件包含 Fuchsia 需要的工具链提供的各种库位置的相对路径。

<!-- ### Putting it All Together -->
### 整合到一起

<!-- Copy-paste code for building a single-stage toolchain. This code can be run
from inside your LLVM build directory and assumes a linux environment. -->
复制并粘贴工具链但阶段构建代码。该代码可以在 linux 环境下的 LLVM 构建目录中执行。

```
cd ${LLVM_BUILD_DIR}  # The directory your toolchain will be installed in

# Environment setup
FUCHSIA_SRCDIR=${HOME}/fuchsia/  # Replace with wherever Fuchsia lives
LLVM_SRCDIR=${HOME}/llvm/llvm-project  # Replace with wherever llvm-project lives
IDK_DIR=${HOME}/fuchsia-idk/
SYSROOT_DIR=${HOME}/fuchsia-sysroot/
CLANG_TOOLCHAIN_PREFIX=${FUCHSIA_SRCDIR}/prebuilt/third_party/clang/linux-x64/bin/
GOMA_DIR=${FUCHSIA_SRCDIR}/prebuilt/third_party/goma/linux-x64/

# Download necessary dependencies
cipd install fuchsia/sdk/core/linux-amd64 latest -root ${IDK_DIR}
cipd install fuchsia/sysroot/linux-arm64 latest -root ${SYSROOT_DIR}/linux-arm64
cipd install fuchsia/sysroot/linux-amd64 latest -root ${SYSROOT_DIR}/linux-x64

# CMake invocation
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang \
  -DCMAKE_CXX_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang++ \
  -DCMAKE_C_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DCMAKE_CXX_COMPILER_LAUNCHER=${GOMA_DIR}/gomacc \
  -DLLVM_ENABLE_LTO=OFF \
  -DLINUX_x86_64-unknown-linux-gnu_SYSROOT=${SYSROOT_DIR}/linux-x64 \
  -DLINUX_aarch64-unknown-linux-gnu_SYSROOT=${SYSROOT_DIR}/linux-arm64 \
  -DFUCHSIA_SDK=${IDK_DIR} \
  -DCMAKE_INSTALL_PREFIX= \
  -C ${LLVM_SRCDIR}/clang/cmake/caches/Fuchsia-stage2.cmake \
  ${LLVM_SRCDIR}/llvm

# Build and strip binaries and place them in the install directory
ninja distribution -j1000
DESTDIR=${INSTALL_DIR} ninja install-distribution-stripped -j1000

# Get runtimes.json
cp ${FUCHSIA_SRCDIR}/prebuilt/third_party/clang/linux-x64/lib/runtime.json ${INSTALL_DIR}/lib/
```

<!-- ### Building Fuchsia with a Custom Clang -->
### 用定制化的 Clang 构建 Fuchsia

<!-- To specify a custom clang toolchain for building Fuchsia, pass
`--args clang_prefix=\"${LLVM_BUILD_DIR}/bin\" --no-goma`
to `fx set` command and run `fx build`. -->
通过传递 `--args clang_prefix=\"${LLVM_BUILD_DIR}/bin\" --no-goma` 参数给
`fx set` 和 `fx build` 命令以指定一个定制化的 clang 工具链来构建 Fuchsia。

```bash
fx set core.x64 --args=clang_prefix=\"${LLVM_BUILD_DIR}/bin\" --no-goma
fx build
```

<!-- This file contains relative paths used by the Fuchsia build to know where
various libraries from the toolchain are located. -->
本文件包含 Fuchsia 构建需要的由工具链提供的各种库的相对路径。

<!-- Note: If you make another change to Clang after building Fuchsia with a previous
version of Clang, re-running `fx build` may not always guarantee that all necessary
targets will be built with the new Clang. For this case, should instead run `fx
clean-build`, which will rebuild everything but definitely use the new Clang. -->
注意：在构建 Fuchsia 之后您又改动了 Clang，只重新运行 `fx build` 可能不足以保证所有必要的目标
都能用新的 Clang 构建。这种情况下，需要运行 `fx clean-build`，这可以确保一切都用新的 Clang 构建。

<!-- ## Developing Clang -->
### 开发 Clang

<!-- When developing Clang, you may want to use a setup that is more suitable for
incremental development and fast turnaround time. -->
在开发 Clang 时， 您可能更希望有一个更适合于增量开发和具有更快周转时间的配置。

<!-- The simplest way to build LLVM is to use the following commands: -->
用以下命令构建 LLVM 是最简单的方式：

```bash
cmake -GNinja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld" \
  ${LLVM_SRCDIR}/llvm
ninja
```

<!-- You can enable additional projects using the `LLVM_ENABLE_PROJECTS`
variable. To enable all common projects, you would use: -->
您可以用 `LLVM_ENABLE_PROJECTS` 来激活附加的项目。要激活所有常用项目，需要：

```bash
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld;compiler-rt;libcxx;libcxxabi;libunwind"
```

<!-- Similarly, you can also enable some projects to be built as runtimes
which means these projects will be built using the just-built rather
than the host compiler: -->
类似地，您可以激活部分项目作为运行时库，这意味着这些项目将用现构建的编译器而不是主机编译器构建：

```bash
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld" \
  -DLLVM_ENABLE_RUNTIMES="compiler-rt;libcxx;libcxxabi;libunwind" \
```

<!-- Both `LLVM_ENABLE_PROJECTS` and `LLVM_ENABLE_RUNTIMES` are already set in the
CMake cache files, so you normally don't need to set these unless you would
like to explicitly add more projects or runtimes. -->
`LLVM_ENABLE_PROJECTS` 和 `LLVM_ENABLE_RUNTIMES` 均已在 CMake 的缓存文件中已经设置好，
因此通常您无需配置，除非您想更明确地添加更多项目或者运行时。

<!-- Clang is a large project and compiler performance is absolutely critical. To
reduce the build time, it is recommended to use Clang as a host compiler, and if
possible, LLD as a host linker. These should be ideally built using LTO and
for best possible performance also using Profile-Guided Optimizations (PGO). -->
Clang 是一个大项目，编译器性能显然严苛。建议将 Clang 作为主机编译器，以减少编译时间，
并且，如果有可能，将 LLD 设置为主机链接器。理想情况下，这些应当采用 LTO 构建，而且，
应用配置文件导向的优化（PGO）以获得最佳性能。

<!-- To set the host compiler to Clang and the host linker to LLD, you can
use the following extra flags: -->
要将主机编译器设置为 Clang，主机链接器设置为 LLD，您可以使用如下的额外标记：

```bash
  -DCMAKE_C_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang \
  -DCMAKE_CXX_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang++ \
  -DLLVM_ENABLE_LLD=ON
```

<!-- This assumes that `${CLANG_TOOLCHAIN_PREFIX}` points to the `bin` directory
of a Clang installation, with a trailing slash (as this Make variable is used
in the Zircon build). For example, to use the compiler from your Fuchsia
checkout (on Linux): -->
这假定 `${CLANG_TOOLCHAIN_PREFIX}` 指向 Clang 安装路径中的 `bin` 目录，包括结尾的
斜杠（因为 Make 变量会被 Zircon 构建使用）。例如，在Linux上从检出的 Fuchsia 中使用编译器： 

```bash
CLANG_TOOLCHAIN_PREFIX=${FUCHSIA}/prebuilt/third_party/clang/linux-x64/bin/
```

<!-- Note: To build Fuchsia, you need a stripped version of the toolchain runtime
binaries. Use `DESTDIR=${INSTALL_DIR} ninja install-distribution-stripped`
to get a stripped install and then point your build configuration to
`${INSTALL_DIR}/bin` as your toolchain. -->
注意：要构建 Fuchsia， 您需要裁剪版本的工具链运行时。使用 `DESTDIR=${INSTALL_DIR} ninja install-distribution-stripped` 来获得裁剪安装并作为工具链，然后将构建配置指向 `${INSTALL_DIR}/bin`。

<!-- ### Sanitizers -->
### 清理器

<!-- Most sanitizers can be used on LLVM tools by adding
`LLVM_USE_SANITIZER=<sanitizer name>` to your cmake invocation. MSan is
special however because some LLVM tools trigger false positives. To
build with MSan support you first need to build libc++ with MSan
support. You can do this in the same build. To set up a build with MSan
support first run CMake with `LLVM_USE_SANITIZER=Memory` and
`LLVM_ENABLE_LIBCXX=ON`. -->
可以通过在调用 cmake 时增加 `LLVM_USE_SANITIZER=<sanitizer name>`参数以在 LLVM
工具上使用大多数的清理器。然而 MSan 清理器是特殊的一个，因为其他一些 LLVM 工具会触发误报。
要构建含有 MSan 支持的工具，得首先构建有 MSan 支持的 libc++ 构建。这可以在同一个构件中完成。
设定构建含有 MSan 支持的构建，首先运行带有 `LLVM_USE_SANITIZER=Memory` 和 `LLVM_ENABLE_LIBCXX=ON`
的 CMake。

```bash
cmake -GNinja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang \
  -DCMAKE_CXX_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang++ \
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld;libcxx;libcxxabi;libunwind" \
  -DLLVM_USE_SANITIZER=Memory \
  -DLLVM_ENABLE_LIBCXX=ON \
  -DLLVM_ENABLE_LLD=ON \
  ${LLVM_SRCDIR}/llvm
```

<!-- Normally you would run Ninja at this point but we want to build
everything using a sanitized version of libc++ but if we build now it
will use libc++ from `${CLANG_TOOLCHAIN_PREFIX}`, which isn't sanitized.
So first we build just the cxx and cxxabi targets. These will be used in
place of the ones from `${CLANG_TOOLCHAIN_PREFIX}` when tools
dynamically link against libcxx -->
通常此时您可以运行 Ninja 了，但我们想用清理过的 libc++ 版本构建所有内容，而此时构建
则会用 `${CLANG_TOOLCHAIN_PREFIX}` 目录未经过清理的 libc++。因此我们得首先构建 cxx
和 cxxabi。二者会在工具动态链接 libcxx 时被位于 `${CLANG_TOOLCHAIN_PREFIX}` 下的libc++
所使用。
 
```bash
ninja cxx cxxabi
```

<!-- Now that you have a sanitized version of libc++ you can set your build to use
it instead of the one from `${CLANG_TOOLCHAIN_PREFIX}` and then build
everything. -->
此时您拥有了清理后的 libc++ 版本，您可以配置您的构建，用它替代 `${CLANG_TOOLCHAIN_PREFIX}`
里的 libc++ 构建其他一切了。

```bash
ninja
```

<!-- Putting that all together: -->
整合到一起：

```bash
cmake -GNinja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang \
  -DCMAKE_CXX_COMPILER=${CLANG_TOOLCHAIN_PREFIX}clang++ \
  -DLLVM_USE_SANITIZER=Address \
  -DLLVM_ENABLE_LIBCXX=ON \
  -DLLVM_ENABLE_LLD=ON \
  ${LLVM_SRCDIR}/llvm
ninja libcxx libcxxabi
ninja
```

<!-- ## Testing Clang -->
## 测试 Clang

<!-- To run Clang tests, you can use the `check-<component>` target: -->
可以用 `check-<component>` 命令运行 Clang 的测试：

```bash
ninja check-llvm check-clang
```

<!-- You can all use `check-all` to run all tests, but keep in mind that this
can take significant amount of time depending on the number of projects
you have enabled in your build. -->
您可以用 `check-all` 命令运行全部测试，但记住这可能耗费大量的时间，时间取决于您在构件中启用的项目的数量。

<!-- To test only one specific test, you can use the environment variable
`LIT_FILTER`. If the path to the test is `clang/test/subpath/testname.cpp`, you
can use: -->
要只运行特定测试，您可以使用 `LIT_FILTER` 环境变量。假如测试路径为 `clang/test/subpath/testname.cpp`，则
您可以这样做：

```bash
LIT_FILTER=testname.cpp ninja check-clang
```

<!-- The same trick can be applied for running tests in other sub-projects by
specifying different a different `check-<component>`. -->
同样的诀窍可以应用到其他子项目中，只要指定不同的 `check-<component>`。

{% dynamic if user.is_googler %}

<!-- ## [Googlers only] Building Fuchsia with custom Clang on bots -->
## [谷歌用户专享] 采用定制化的 Clang 自动构建 Fuchsia

<!-- Fuchsia's infrastructure has support for using a non-default version of Clang
to build. Only Clang instances that have been uploaded to CIPD or Isolate are
available for this type of build, and so any local changes must land in
upstream and be built by the CI or production toolchain bots. -->
Fuchsia 的基础架构支持非默认版本的 Clang 来构建。对于这种构建类型，只有导入到 CIPD 或者 Isolate 
的 Clang 实例才可用，因此任何在本地的更改须传到上游并且由 CI 或者产品工具链自动构建。

<!-- You will need the infra codebase and prebuilts. Directions for checkout are on
the infra page. -->
您需要基础架构代码库和预构建，检出的指令在基础架构页面上可以找到。

<!-- To trigger a bot build with a specific revision of Clang, you will need the Git
revision of the Clang with which you want to build. This is on the [CIPD page](https://chrome-infra-packages.appspot.com/p/fuchsia/clang),
or can be retrieved using the CIPD CLI. You can then run the following command: -->
要触发机器人用指定版本的 Clang，您需要通过 Git 获取需要编译的 Clang 版本。在[CIPD页面](https://chrome-infra-packages.appspot.com/p/fuchsia/clang)可以找到，或者可以通过 CIPD 命令行获取。
之后运行如下命令：

```bash
export FUCHSIA_SOURCE=<path_to_fuchsia>
export BUILDER=<builder_name>
export REVISION=<clang_revision>

export INFRA_PREBUILTS=${FUCHSIA_SOURCE}/fuchsia-infra/prebuilt/tools

cd ${FUCHSIA_SOURCE}/fuchsia-infra/recipes

${INFRA_PREBUILTS}/led get-builder 'luci.fuchsia.ci:${BUILDER}' | \
${INFRA_PREBUILTS}/led edit-recipe-bundle -O | \
jq '.userland.recipe_properties."$infra/fuchsia".clang_toolchain.type="cipd"' | \
jq '.userland.recipe_properties."$infra/fuchsia".clang_toolchain.instance="git_revision:${REVISION}"' | \
${INFRA_PREBUILTS}/led launch
```

<!-- It will provide you with a link to the BuildBucket page to track your build. -->
它将提供一个链接到 BuildBucket 页面的链接以跟踪您的构建。

<!-- You will need to run `led auth-login` prior to triggering any builds, and may need to
file an infra ticket to request access to run led jobs. -->
在触发任何构建之前，您需要先运行 `led auth-login`，并可能需要发起一个基础架构票据以申请运行 led 任务的权限。

{% dynamic endif %}

<!-- ## Useful CMake Flags -->
### 有用 CMake 标记

<!-- There are many other [CMake flags](https://llvm.org/docs/CMake.html#id11) that
are useful for building, but these are some that may be useful for toolchain
building. -->
有其他许多有利于构建的[CMake 标记](https://llvm.org/docs/CMake.html#id11)，但以下是部分对工具链的构建
有用的标记。

### `-DLLVM_PARALLEL_LINK_JOBS`

<!-- Increase the number of link jobs that can be run in parallel (locally). The number of
link jobs is dependent on RAM size. For LTO build you will
need at least 10GB for each job. -->
增加可并行执行（本地）的链接任务数。链接任务数量取决于内存大小。对于 LTO 构建，每个任务至少需要 10GB。

<!-- ## Additional Resources -->
## 附加资源

<!-- Documentation: -->
文档：

<!-- * [Getting Started with the LLVM System](http://llvm.org/docs/GettingStarted.html)
* [Building LLVM with CMake](http://llvm.org/docs/CMake.html)
* [Advanced Build Configurations](http://llvm.org/docs/AdvancedBuilds.html) -->
* [LLVM 系统入门](http://llvm.org/docs/GettingStarted.html)
* [用 CMake 构建 LLVM](http://llvm.org/docs/CMake.html)
* [进阶构建配置](http://llvm.org/docs/AdvancedBuilds.html) 

<!-- Talks: -->
谈话内容：

<!-- * [2016 LLVM Developers’ Meeting: C. Bieneman "Developing and Shipping LLVM and Clang with CMake"](https://www.youtube.com/watch?v=StF77Cx7pz8)
* [2017 LLVM Developers’ Meeting: Petr Hosek "Compiling cross-toolchains with CMake and runtimes build"](https://www.youtube.com/watch?v=OCQGpUzXDsY) -->
* [2016 LLVM 开发者大会：C. Bieneman “用 CMake 开发和发放 LLVM 及 Clang” ](https://www.youtube.com/watch?v=StF77Cx7pz8)
* [2017 LLVM 开发者大会：Petr Hosek “使用 CMake 编译交叉工具链和运行时的构建”](https://www.youtube.com/watch?v=OCQGpUzXDsY)

[//prebuilt/integration]: https://fuchsia.googlesource.com/integration/+/HEAD/prebuilts
