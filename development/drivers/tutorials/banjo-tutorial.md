<!-- # Banjo tutorial -->
# Banjo 教程

<!-- Banjo is a "transpiler" (like [FIDL's
`fidlc`](/docs/development/languages/fidl/README.md))
&mdash; a program that converts an interface definition language (**IDL**) into target language
specific files. -->
Banjo 是一个语言翻译器（类似[FIDL's
`fidlc`](/docs/development/languages/fidl/README.md))&mdash; 一个将接口定义语言(**IDL**)转
成目标语言特定文档的程序。

<!-- This tutorial is structured as follows: -->
本教程组织成下：

<!-- * brief overview of Banjo
* simple example (I2C)
* explanation of generated code from example -->
* Banjo 概览
* 简单示例（I2C）
* 示例中生成代码的解释

<!-- There's also a reference section that includes: -->
还有一个参考章节内容包含：

<!-- * a list of builtin keywords and primitive types. -->
* 内置关键字和基本数据类型

<!-- ## Overview -->
## 概览

<!-- Banjo generates C and C++ code that can be used by both the protocol implementer
and the protocol user. -->
Banjo 生成可以被协议开发者和协议用户均可以使用的C和C++代码。

<!-- ## A simple example -->
## 简单示例

<!-- As a first step, let's take a look at a relatively simple Banjo specification.
This is the file [`//sdk/banjo/fuchsia.hardware.i2c/i2c.fidl`](/sdk/banjo/fuchsia.hardware.i2c/i2c.fidl): -->
初次尝试，我们先看看 Banjo 相对简单的定义。文件在这里[`//sdk/banjo/fuchsia.hardware.i2c/i2c.fidl`](/sdk/banjo/fuchsia.hardware.i2c/i2c.fidl)：

<!-- > Note that the line numbers in the code samples throughout this tutorial are not part of the files. -->
 > 注意本教程示例中的行号并不是文件的一部分。

```banjo
[01] // Copyright 2018 The Fuchsia Authors. All rights reserved.
[02] // Use of this source code is governed by a BSD-style license that can be
[03] // found in the LICENSE file.
[04]
[05] library fuchsia.hardware.i2c;
[06]
[07] using zx;
[08]
[09] const uint32 I2C_10_BIT_ADDR_MASK = 0xF000;
[10] const uint32 I2C_MAX_RW_OPS = 8;
[11]
[12] /// See `Transact` below for usage.
[13] struct I2cOp {
[14]     vector<voidptr> data;
[15]     bool is_read;
[16]     bool stop;
[17] };
[18]
[19] [Transport = "Banjo", BanjoLayout = "ddk-protocol"]
[20] protocol I2c {
[21]     /// Writes and reads data on an i2c channel. Up to I2C_MAX_RW_OPS operations can be passed in.
[22]     /// For write ops, i2c_op_t.data points to data to write.  The data to write does not need to be
[23]     /// kept alive after this call.  For read ops, i2c_op_t.data is ignored.  Any combination of reads
[24]     /// and writes can be specified.  At least the last op must have the stop flag set.
[25]     /// The results of the operations are returned asynchronously through the transact_cb.
[26]     /// The cookie parameter can be used to pass your own private data to the transact_cb callback.
[27]     [Async]
[28]     Transact(vector<I2cOp> op) -> (zx.status status, vector<I2cOp> op);
[29]     /// Returns the maximum transfer size for read and write operations on the channel.
[30]     GetMaxTransferSize() -> (zx.status s, usize size);
[31]     GetInterrupt(uint32 flags) -> (zx.status s, handle<interrupt> irq);
[32] };
```

<!-- It defines an interface that allows an application to read and write data on an I2C bus.
In the I2C bus, data must first be written to the device in order to solicit
a response.
If a response is desired, the response can be read from the device.
(A response might not be required when setting a write-only register, for example.) -->
其定义了一个允许应用从I2C总线中读写数据的接口。在I2C总线中，数据应该首先被写入设备以等待一个应答。
如果应答正是期望的，那么他可以被设备读取。
（如果设定了止血的寄存器还没应答就不是必须的）。

<!-- Let's look at the individual components, line-by-line: -->
我们一起一行一行的看一看各个组件：

<!-- * `[05]` &mdash; the `library` directive tells the Banjo compiler what prefix it should
  use on the generated output; think of it as a namespace specifier.
* `[07]` &mdash; the `using` directive tells Banjo to include the `zx` library.
* `[09]` and `[10]` &mdash; these introduce two constants for use by the programmer.
* `[13` .. `17]` &mdash; these define a structure, called `I2cOp`, that the programmer
  will then use for transferring data to and from the bus.
* `[19` .. `32]` &mdash; these lines define the interface methods that are provided by
  this Banjo specification; we'll discuss this in greater detail [below](#the-interface). -->
* `[05]` &mdash; `library` 指令告诉Banjo编译器在生成的代码中应该使用什么样的前缀。你认为是一个命名空间的限定符。
* `[07]` &mdash; `using` 指令告诉 Banjo 将`zx`库包含进来。
* `[09]` and `[10]` &mdash; 引入2个可以被程序员使用的常量。
* `[13` .. `17]` &mdash; 定义了名为`I2cOp`的数据结构供程序员之后使用以与总线进行数据交互。
* `[19` .. `32]` &mdash; 这几行定义了 Banjo 规范提供的接口方法。我们会在后面做更详细的讨论[below](#the-interface)

<!-- > Don't be confused by the comments on `[21` .. `26]` (and elsewhere) &mdash; they're
> "flow through" comments that are intended to be emitted into the generated source.
> Any comment that starts with "`///`" (three! slashes) is a "flow through" comment.
> Ordinary comments (that is, "`//`") are intended for the current module.
> This will become clear when we look at the generated code. -->
> 不要被`[21` .. `26]`（及其他地方）的注释弄糊涂 &mdash;他们只是要被加到生成代码中的注释。
> 任何以“`///`”（三个斜线）开头的注释都是“过场”注释。
> 正常注释（也就是“`//`”）是专门为当前模块。
> 当我们看到生成的代码之后这就会变得很清晰。

<!-- ## The operation structure -->
## 运算结构

<!-- In our I2C sample, the `struct I2cOp` structure defines three elements: -->
在我们I2C例子里，`struct I2cOp`结构定义了3个元素：

<!-- Element   | Type              | Use
----------|-------------------|-----------------------------------------------------------------
`data`    | `vector<voidptr>` | contains the data sent to, and optionally received from, the bus
`is_read` | `bool`            | flag indicating read functionality desired
`stop`    | `bool`            | flag indicating a stop byte should be sent after the operation -->

元素      | 类型              | 用法
----------|-------------------|-----------------------------------------------------------------
`data`    | `vector<voidptr>` | 包含发往总线，或从总线中接收的数据
`is_read` | `bool`            | 标记期待读功能
`stop`    | `bool`            | 表明操作后需要发送停止位

<!-- The structure defines the communications area that will be used between the protocol
implementation (the driver) and the protocol user (the program that's using the bus). -->
本结构定义了协议实现方（驱动）和协议使用方（使用总线的程序）之间的通信区。

<!-- ## The interface -->
## 接口

<!-- The more interesting part is the `protocol` specification. -->
更有意思的部分是`protocol`（协议）规范

<!-- We'll skip the `[Transport = "Banjo", BanjoLayout]` (line `[19]`) and `[Async]` (line `[27]`) attributes for now, -->
我们先跳过`[Transport = "Banjo", BanjoLayout]` (line `[19]`) and `[Async]` (line `[27]`)

<!-- but will return to them below, in [Attributes](#attributes). -->
但在后边[Attributes](#attributes)会回过头来看此部分。

<!-- The `protocol` section defines three interface methods: -->
`protocol` 协议章节定义了3个接口方法：

* `Transact`
* `GetMaxTransferSize`
* `GetInterrupt`

<!-- Without going into details about their internal operations (this isn't a tutorial on
I2C, after all), let's see how they translate into the target language.
We'll look at the C and C++ implementations separately, using the C description
to include the structure definition that's common to the C++ version as well. -->
跳过内部操作的细节（这本不是I2C的教程），让我们看看他们如何翻译成目标语言。
我们将分开来看C和C++实现，C语言来描述包含的结构定义，这也常常见于C++的版本。

<!-- > Currently, generation of C and C++ code is supported, with Rust support planned
> in the future. -->
> 目前支持生成C和C++代码，对Rust的支持在计划之中。

## C

<!-- The C implementation is relatively straightforward: -->
C语言的实现相当明了：

<!-- * `struct`s and `union`s map almost directly into their C language counterparts.
* `enum`s and constants are generated as `#define` macros.
* `protocol`s are generated as two `struct`s:
    * a function table, and
    * a struct with pointers to the function table and a context.
* Some helper functions are also generated. -->
* `struct` 和 `union`几乎直接映射为C语言对应的部分。
* `enum` 和常量则以`#define`宏的形式生成。
* `protocol` 生成两个 `struct` ：
  * 函数表，和
  * 包含指向函数表的指针和上下文的结构
* 同时也生成一些辅助函数。

<!-- The C version is generated into
`$BUILD_DIR/banjoing/gen/fuchisia/hardware/i2c/c/banjo.h`,
where _TARGET_ is the target architecture, e.g., `arm64`. -->
C语言版本生成到
`$BUILD_DIR/banjoing/gen/fuchisia/hardware/i2c/c/banjo.h`，
此处 _TARGET_ 是目标系统架构，比如，`arm64`。

<!-- The file is relatively long, so we'll look at it in several parts. -->
该文件比较长，因此我们分几部分来考察。

<!-- ### Boilerplate -->
### 文件范例

<!-- The first part has some boilerplate, which we'll show without further comment: -->
第一部分有一些范例因此我们不做更深的解释：

```c
[01] // Copyright 2018 The Fuchsia Authors. All rights reserved.
[02] // Use of this source code is governed by a BSD-style license that can be
[03] // found in the LICENSE file.
[04]
[05] // WARNING: THIS FILE IS MACHINE GENERATED. DO NOT EDIT.
[06] //          MODIFY sdk/banjo/fuchsia.hardware.i2c/i2c.banjo INSTEAD.
[07]
[08] #pragma once
[09]
[10] #include <zircon/compiler.h>
[11] #include <zircon/types.h>
[12]
[13] __BEGIN_CDECLS
```

<!-- ### Forward declarations -->
### 前向声明

<!-- Next are forward declarations for our structures and functions: -->
接下来是对结构和函数的前向声明：

```c
[15] // Forward declarations
[16]
[17] typedef struct i2c_op i2c_op_t;
[18] typedef struct i2c_protocol i2c_protocol_t;
[19] typedef void (*i2c_transact_callback)(void* ctx, zx_status_t status, const i2c_op_t* op_list, size_t op_count);
[20]
[21] // Declarations
[22]
[23] // See `Transact` below for usage.
[24] struct i2c_op {
[25]     const void* data_buffer;
[26]     size_t data_size;
[27]     bool is_read;
[28]     bool stop;
[29] };
```

<!-- Note that lines `[17` .. `19]` only declare types, they don't actually define
structures or prototypes for functions. -->
注意 `[17` .. `19]` 行仅声明了类型，并未实际定义结构或者函数的原型。

<!-- Notice how the "flow through" comments (original `.banjo` file line `[12]`, for example)
got emitted into the generated code (line `[23]` above), with one slash stripped off to
make them look like normal comments. -->
留意“过场注释”（如，原始 `.banjo` 文件中的第 `[12]`行）是如何被注入到产生的代码中的（如，上面的第 `[23]` 行），
去除了一个斜线以使它们看起来像是一个通常的注释。

<!-- Lines `[24` .. `29`] are, as advertised, an almost direct mapping of the `struct I2cOp`
from the `.banjo` file above (lines `[13` .. `17`]). -->
如之前所言，第 `[24` .. `29]` 行几乎是上述 `.banjo` 文件中（`[13` .. `17]`行） `struct I2cOp` 类型的直接映射。

<!-- Astute C programmers will immediately see how the C++ style `vector<voidptr> data` (original
`.banjo` file line `[14]`) is handled in C: it gets converted to a pointer
("`data_buffer`") and a size ("`data_size`"). -->
敏锐的C程序员几乎会立刻发现C++风格的 `vector<voidptr> data`（原始 `.banjo` 文件第 `[14]` 行）
如何在C语言中实现：转换成一个指针（“`data_buffer`”）和一个宽度（“`data_size`”）。

<!-- > As far as the naming goes, the base name is `data` (as given in the `.banjo` file).
> For a vector of `voidptr`, the transpiler appends `_buffer` and `_size` to convert the
> `vector` into a C compatible structure.
> For all other vector types, the transpiler appends `_list` and `_count` instead (for
> code readability). -->
> 就命名而言，基本名称是 `data`（如 `.banjo` 文件所定）。
> 对于 `voidptr` 型的向量，编译器添加 `_buffer`和 `_size` 后缀以将向量 `vector` 转换成C语言兼容的结构。
> 对于所有其他向量类型, 编译器取而代之添加 `_list` 和 `_count` 后缀（增强代码可读性）。

<!-- ### Constants -->
### 常量

<!-- Next, we see our `const uint32` constants converted into `#define` statements: -->
接下来，我们看到常量 `const uint32` 被转成 `#define` 声明：

```c
[31] #define I2C_MAX_RW_OPS UINT32_C(8)
[32]
[33] #define I2C_10_BIT_ADDR_MASK UINT32_C(0xF000)
```

<!-- In the C version, We chose `#define` instead of "passing through" the `const uint32_t`
representation because: -->
在C版本里，我们选择用 `#define` 而不是 `const uint32_t` 表示，因为：

<!-- * `#define` statements only exist at compile time, and get inlined at every usage site, whereas
  a `const uint32_t` would get embedded in the binary, and
* `#define` allows for more compile time optimizations (e.g., doing math with the constant value). -->
* `#define` 语句时在编译时存在，并且在使用之处直接插入替换，而 `const uint32_t` 将嵌入二进制文件，并且
* `#define` 允许更多的编译时优化（如：使用常量进行数学计算）。

<!-- The downside is that we don't get type safety, which is why you see the helper macros (like
**UINT32_C()** above); they just cast the constant to the appropriate type. -->
缺点是我们无法保证类型安全，这就是为什么你常看到那些帮助宏（如上面的 **UINT32_C()**）；他们只是将常量转成适当的类型。

<!-- ### Protocol structures -->
### 协议结构

<!-- And now we get into the good parts. -->
现在我们进入漂亮的部分。

```c
[35] typedef struct i2c_protocol_ops {
[36]     void (*transact)(void* ctx, const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie);
[37]     zx_status_t (*get_max_transfer_size)(void* ctx, size_t* out_size);
[38]     zx_status_t (*get_interrupt)(void* ctx, uint32_t flags, zx_handle_t* out_irq);
[39] } i2c_protocol_ops_t;
```

<!-- This `typedef` creates a structure definition that contains the three `protocol` methods
that were defined in the original `.banjo` file at lines `[28]`, `[30]` and `[31]`. -->
`typedef` 创建了一个包括3个协议 `protocol` 方法的结构。该方法定义在原始的 `.banjo` 文件中的第 `[28]`, `[30]` 和 `[31]` 行。

Notice the name mangling that has occurred &mdash; this is how you can map the
`protocol` method names to the C function pointer names so that you know what
they're called:
注意命名变化&mdash;你如何将 `protocol` 方法名称与C语言函数指针相映射以便你知道他们的叫法：

<!-- Banjo                | C                       | Rule
---------------------|-------------------------|---------------------------------------------------------------
`Transact`           | `transact`              | Convert leading uppercase to lowercase
`GetMaxTransferSize` | `get_max_transfer_size` | As above, and convert camel-case to underscore-separated style
`GetInterrupt`       | `get_interrupt`         | Same as above -->
Banjo                | C                       | 规则
---------------------|-------------------------|---------------------------------------------------------------
`Transact`           | `transact`              | 首字母转小写
`GetMaxTransferSize` | `get_max_transfer_size` | 同上，并将驼峰命名改为以下划线区分的命名法
`GetInterrupt`       | `get_interrupt`         | 同上

<!-- Next, the interface definitions are wrapped in a context-bearing structure: -->
接下来，接口定义被封装到一个语境相关的结构中：

```c
[41] struct i2c_protocol {
[42]     i2c_protocol_ops_t* ops;
[43]     void* ctx;
[44] };
```

<!-- And now the "flow-through" comments (`.banjo` file, lines `[21` .. `26]`)
suddenly make way more sense! -->
现在“过场注释”（`.banjo`文件第 `[21` .. `26]`行）忽然使路途变得更有意义！

```c
[46] // Writes and reads data on an i2c channel. Up to I2C_MAX_RW_OPS operations can be passed in.
[47] // For write ops, i2c_op_t.data points to data to write.  The data to write does not need to be
[48] // kept alive after this call.  For read ops, i2c_op_t.data is ignored.  Any combination of reads
[49] // and writes can be specified.  At least the last op must have the stop flag set.
[50] // The results of the operations are returned asynchronously through the transact_cb.
[51] // The cookie parameter can be used to pass your own private data to the transact_cb callback.
```

<!-- Finally, we see the actual generated code for the three methods: -->
最后，我们看看这3个方法实际生成的代码：

```c
[52] static inline void i2c_transact(const i2c_protocol_t* proto, const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie) {
[53]     proto->ops->transact(proto->ctx, op_list, op_count, callback, cookie);
[54] }
[55] // Returns the maximum transfer size for read and write operations on the channel.
[56] static inline zx_status_t i2c_get_max_transfer_size(const i2c_protocol_t* proto, size_t* out_size) {
[57]     return proto->ops->get_max_transfer_size(proto->ctx, out_size);
[58] }
[59] static inline zx_status_t i2c_get_interrupt(const i2c_protocol_t* proto, uint32_t flags, zx_handle_t* out_irq) {
[60]     return proto->ops->get_interrupt(proto->ctx, flags, out_irq);
[61] }
```

<!-- ### Prefixes and paths -->
### 前缀和路径

<!-- Notice how the prefix `i2c_` (from the interface name, `.banjo` file line `[20]`)
got added to the method names; thus, `Transact` became `i2c_transact`, and so on.
This is part of the mapping between `.banjo` names and their C equivalents. -->
注意前缀 `i2c_` （ `.banjo` 文件第 `[20]` 行的接口名称）是如何被加到方法名上的，
也就是 `Transact` 变成 `i2c_transact` 等等。这是 `.banjo` 名称和相应的C语言的映射的一部分。

<!-- Also, the `library` name (line `[05]` in the `.banjo` file) is transformed into the
include path: so `library fuchsia.hardware.i2c` implies a path of `<fuchsia/hardware/i2c/c/banjo.h>`. -->
同样， `library` 名称（ `.banjo` 文件的第 `[05]`行）被转成包含路径：即 `library fuchsia.hardware.i2c`
意味着 `<fuchsia/hardware/i2c/c/banjo.h>` 这样的路径。

## C++

<!-- The C++ code is slightly more complex than the C version.
Let's take a look. -->
C++代码比C语言版本稍稍复杂，一起看看。

<!-- The Banjo transpiler generates three files:
the first is the C file discussed above, and the other two are under
`$BUILD_DIR/banjoing/gen/fuchsia/hardware/i2c/cpp/banjo.h`: -->
Banjo编译器生成三个文件：
第一个是上面讨论的C语言文件，其他2个在`$BUILD_DIR/banjoing/gen/fuchsia/hardware/i2c/cpp/banjo.h`: 

<!-- * `i2c.h` &mdash; the file your program should include, and
* `i2c-internal.h` &mdash; an internal file, included by `i2c.h` -->
* `i2c.h` &mdash; 你的程序中需要包含的文件
* `i2c-internal.h` &mdash; 被 `i2c.h` 包含的内部文件

<!-- As usual, _TARGET_ is the build target architecture (e.g., `x64`). -->
一如既往 _TARGET_ 是构建目标的架构（如`x64`）。

<!-- The "internal" file contains declarations and assertions, which we can safely skip. -->
内部文件包含定义和断言，我们可以安全地跳过。

<!-- The C++ version of `i2c.h` is fairly long, so we'll look at it in smaller pieces.
Here's an overview "map" of what we'll be looking at, showing the starting line
number of each piece: -->
C++ 版本的 `i2c.h` 文件极长，因此我们分成小段来看。这里是概要目录，显示了每一段的起始行号：

<!-- Line | Section
--------------|----------------------------
1    | [boilerplate](#a-simple-example-c-boilerplate-2)
20   | [auto generated usage comments](#auto_generated-comments)
55   | [class I2cProtocol](#the-i2cprotocol-mixin-class)
99   | [class I2cProtocolClient](#the-i2cprotocolclient-wrapper-class) -->
行号 | 章节
--------------|----------------------------
1    | [模板](#a-simple-example-c-boilerplate-2)
20   | [自动生成的使用注释](#auto_generated-comments)
55   | [类 I2cProtocol](#the-i2cprotocol-mixin-class)
99   | [类 I2cProtocolClient](#the-i2cprotocolclient-wrapper-class)

<!-- ### Boilerplate -->
### 模板

<!-- The boilerplate is pretty much what you'd expect: -->
模板差不多是你所期望的样子：

```c++
[001] // Copyright 2018 The Fuchsia Authors. All rights reserved.
[002] // Use of this source code is governed by a BSD-style license that can be
[003] // found in the LICENSE file.
[004]
[005] // WARNING: THIS FILE IS MACHINE GENERATED. DO NOT EDIT.
[006] //          MODIFY sdk/banjo/fuchsia.hardware.i2c/i2c.banjo INSTEAD.
[007]
[008] #pragma once
[009]
[010] #include <ddk/driver.h>
[011] #include <fuchsia/hardware/i2c/c/banjo.h>
[012] #include <ddktl/device-internal.h>
[013] #include <zircon/assert.h>
[014] #include <zircon/compiler.h>
[015] #include <zircon/types.h>
[016] #include <lib/zx/interrupt.h>
[017]
[018] #include "i2c-internal.h"
```

<!-- It `#include`s a bunch of DDK and OS headers, including: -->
他包含了大量的DDK和系统头文件，包括：

<!-- * the C version of the header (line `[011]`, which means that everything discussed
  [above in the C section](#a-simple-example-c-1) applies here as well), and
* the generated `i2c-internal.h` file (line `[018]`). -->
* C版本的头文件（ `[011]`行，这意味着上面[C语言章节](#a-simple-example-c-1) 讨论的也适用于此处），
* 生成的 `i2c-internal.h` 文件（`[018]`行）

<!-- Next is the "auto generated usage comments" section; we'll come back to that
[later](#auto_generated-comments) as it will make more sense once we've seen
the actual class declarations. -->
之后是“自动生成的使用说明”章节；我们将回过头来再看[稍后](#auto_generated-comments)，因为当我们看了实际生成的类定义之后一切会更清晰明了。

<!-- The two class declarations are wrapped in the DDK namespace: -->
2个类定义封装在 DDK 命名空间里：

```c++
[053] namespace ddk {
...
[150] } // namespace ddk
```

<!-- ### The I2cProtocolClient wrapper class -->
### I2cProtocolClient 包装类 

<!-- The `I2cProtocolClient` class is a simple wrapper around the `i2c_protocol_t`
structure (defined in the C include file, line `[41]`, which we discussed in
[Protocol structures](#protocol-structures), above). -->
`I2cProtocolClient`类是 `i2c_protocol_t` 结构（定义在上面讨论的[协议结构](#protocol-structures)
的C包含文件第  `[41]` 行里）的简单包装。

```c++
[099] class I2cProtocolClient {
[100] public:
[101]     I2cProtocolClient()
[102]         : ops_(nullptr), ctx_(nullptr) {}
[103]     I2cProtocolClient(const i2c_protocol_t* proto)
[104]         : ops_(proto->ops), ctx_(proto->ctx) {}
[105]
[106]     I2cProtocolClient(zx_device_t* parent) {
[107]         i2c_protocol_t proto;
[108]         if (device_get_protocol(parent, ZX_PROTOCOL_I2C, &proto) == ZX_OK) {
[109]             ops_ = proto.ops;
[110]             ctx_ = proto.ctx;
[111]         } else {
[112]             ops_ = nullptr;
[113]             ctx_ = nullptr;
[114]         }
[115]     }
[116]
[117]     void GetProto(i2c_protocol_t* proto) const {
[118]         proto->ctx = ctx_;
[119]         proto->ops = ops_;
[120]     }
[121]     bool is_valid() const {
[122]         return ops_ != nullptr;
[123]     }
[124]     void clear() {
[125]         ctx_ = nullptr;
[126]         ops_ = nullptr;
[127]     }
[128]     // Writes and reads data on an i2c channel. Up to I2C_MAX_RW_OPS operations can be passed in.
[129]     // For write ops, i2c_op_t.data points to data to write.  The data to write does not need to be
[130]     // kept alive after this call.  For read ops, i2c_op_t.data is ignored.  Any combination of reads
[131]     // and writes can be specified.  At least the last op must have the stop flag set.
[132]     // The results of the operations are returned asynchronously through the transact_cb.
[133]     // The cookie parameter can be used to pass your own private data to the transact_cb callback.
[134]     void Transact(const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie) const {
[135]         ops_->transact(ctx_, op_list, op_count, callback, cookie);
[136]     }
[137]     // Returns the maximum transfer size for read and write operations on the channel.
[138]     zx_status_t GetMaxTransferSize(size_t* out_size) const {
[139]         return ops_->get_max_transfer_size(ctx_, out_size);
[140]     }
[141]     zx_status_t GetInterrupt(uint32_t flags, zx::interrupt* out_irq) const {
[142]         return ops_->get_interrupt(ctx_, flags, out_irq->reset_and_get_address());
[143]     }
[144]
[145] private:
[146]     i2c_protocol_ops_t* ops_;
[147]     void* ctx_;
[148] };
```

<!-- There are three constructors: -->
有3个构造函数：

<!-- * the default one (`[101]`) that sets `ops_` and `ctx_` to `nullptr`,
* an initializer (`[103]`) that takes a pointer to an `i2c_protocol_t` structure and populates
  the `ops_` and `ctx`_ fields from their namesakes in the structure, and
* another initializer (`[106]`) that extracts the `ops`_ and `ctx_` information from
  a `zx_device_t`. -->
* 默认构造函数（第`[101]`行）将 `ops_` 和 `ctx_` 初始化为 `nullptr`
* 接收一个指向 `i2c_protocol_t` 结构的指针的构造函数（第 `[106]` 行），它将 `ops_` 和 `ctx_` 字段设定为结构中同名字段的值。
* 另一个构造函数（第 `[106]` 行）从 `zx_device_t` 中抽取 `ops_` 和 `ctx_` 信息。

<!-- The last constructor is the preferred one, and can be used like this: -->
首先使用最后一个构造函数，可以像下面这样：

```c++
ddk::I2cProtocolClient i2c(parent);
if (!i2c.is_valid()) {
  return ZX_ERR_*; // return an appropriate error
}
```

<!-- Three convenience member functions are provided: -->
同时也提供了3个便捷函数字：

<!-- * `[117]` **GetProto()** fetches the `ctx_` and `ops_` members into a protocol structure,
* `[121]` **is_valid()** returns a `bool` indicating if the class has been initialized with
   a protocol, and
* `[124]` **clear()** invalidates the `ctx_` and `ops_` pointers. -->
* 第 `[117]` 行 **GetProto()** 提取 `ctx_` 和 `ops_` 成员到一个协议结构中
* 第 `[121]` 行 **is_valid()** 返回一个 `bool` 标记该类是否已经被一个协议初始化
* 第 `[124]` 行 **clear()** 清除 `ctx_` 和 `ops_` 指针

<!-- Next we find the three member functions that were specified in the `.banjo` file: -->
接下来我们发现在 `.banjo` 文件中详细定义的3个成员函数：

<!-- * `[134]` **Transact()**,
* `[138]` **GetMaxTransferSize()**, and
* `[141]` **GetInterrupt()**. -->
* 第 `[134]` 行 **Transact()**
* 第 `[138]` 行 **GetMaxTransferSize()**
* 第 `[141]` 行 **GetInterrupt()**

<!-- These work just liked the three wrapper functions from the C version of the include file &mdash;
that is, they pass their arguments into a call through the respective function pointer. -->
这三个函数与包含文件中的C语言版本的工作原理类似 &mdash; 也就是，他们通过函数指针给被调用函数传递各自相应的参数。

<!-- In fact, compare **i2c_get_max_transfer_size()** from the C version: -->
事实上， 相较于 **i2c_get_max_transfer_size()** 的C语言版本：

```c
[56] static inline zx_status_t i2c_get_max_transfer_size(const i2c_protocol_t* proto, size_t* out_size) {
[57]     return proto->ops->get_max_transfer_size(proto->ctx, out_size);
[58] }
```

<!-- with the C++ version above: -->
上述的C++版本：

```c++
[138] zx_status_t GetMaxTransferSize(size_t* out_size) const {
[139]   return ops_->get_max_transfer_size(ctx_, out_size);
[140] }
```

<!-- As advertised, all that this class does is store the operations and context pointers for
later use, so that the call through the wrapper is more elegant. -->
正如宣称的，这个类所做的全部就是将操作和上下文指针存储起来以备后用，因此通过包装函数来调用更加优雅。

<!-- > You'll also notice that the C++ wrapper function doesn't have any name mangling &mdash;
> to use a tautology, **GetMaxTransferSize()** is **GetMaxTransferSize()**. -->
> 你应当注意到C++版本的包装函数并没有对名称进行修正  &mdash;
> **GetMaxTransferSize()** 就是 **GetMaxTransferSize()**

<!-- ### The I2cProtocol mixin class -->
### I2cProtocol 杂混类

<!-- Ok, that was the easy part.
For this next part, we're going to talk about [mixins](https://en.wikipedia.org/wiki/Mixin)
and [CRTPs &mdash; or Curiously Recurring Template
Patterns](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern). -->
好了，这是容易的部分。
对于接下来程序部分，我们打算讲讲[mixins](https://en.wikipedia.org/wiki/Mixin)
和[CRTPs &mdash; 或，神奇再生样板模式](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)。

<!-- Let's understand the "shape" of the class first (comment lines deleted for outlining
purposes): -->
我们先了解下类的概况（清晰起见 ，删掉了注释行）。

```c++
[055] template <typename D, typename Base = internal::base_mixin>
[056] class I2cProtocol : public Base {
[057] public:
[058]     I2cProtocol() {
[059]         internal::CheckI2cProtocolSubclass<D>();
[060]         i2c_protocol_ops_.transact = I2cTransact;
[061]         i2c_protocol_ops_.get_max_transfer_size = I2cGetMaxTransferSize;
[062]         i2c_protocol_ops_.get_interrupt = I2cGetInterrupt;
[063]
[064]         if constexpr (internal::is_base_proto<Base>::value) {
[065]             auto dev = static_cast<D*>(this);
[066]             // Can only inherit from one base_protocol implementation.
[067]             ZX_ASSERT(dev->ddk_proto_id_ == 0);
[068]             dev->ddk_proto_id_ = ZX_PROTOCOL_I2C;
[069]             dev->ddk_proto_ops_ = &i2c_protocol_ops_;
[070]         }
[071]     }
[072]
[073] protected:
[074]     i2c_protocol_ops_t i2c_protocol_ops_ = {};
[075]
[076] private:
...
[083]     static void I2cTransact(void* ctx, const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie) {
[084]         static_cast<D*>(ctx)->I2cTransact(op_list, op_count, callback, cookie);
[085]     }
...
[087]     static zx_status_t I2cGetMaxTransferSize(void* ctx, size_t* out_size) {
[088]         auto ret = static_cast<D*>(ctx)->I2cGetMaxTransferSize(out_size);
[089]         return ret;
[090]     }
[091]     static zx_status_t I2cGetInterrupt(void* ctx, uint32_t flags, zx_handle_t* out_irq) {
[092]         zx::interrupt out_irq2;
[093]         auto ret = static_cast<D*>(ctx)->I2cGetInterrupt(flags, &out_irq2);
[094]         *out_irq = out_irq2.release();
[095]         return ret;
[096]     }
[097] };
```

<!-- The `I2CProtocol` class inherits from a base class, specified by the second template parameter.
If it's left unspecified, it defaults to `internal::base_mixin`, and no special magic happens.
If, however, the base class is explicitly specified, it should be `ddk::base_protocol`,
in which case additional asserts are added (to double check that only one mixin is the base protocol).
In addition, special DDKTL fields are set to automatically register this protocol as the
base protocol when the driver triggers **DdkAdd()**. -->
`I2CProtocol` 类继承自第2个模板参数指定的基类。该模板参数如果不指定，则默认为 `internal::base_mixin`，
并且什么都不会发生。但如果要确切指定，必须得是 `ddk::base_protocol` 。这种情况下，将加入额外的断言（双重确认只有一个混合类型是基础协议）。另外，特殊的 DDKTL 字段将被设置为当驱动触发 **DdkAdd()** 时自动将此协议注册为基础协议。

<!-- The constructor calls an internal validation function, **CheckI2cProtocolSubclass()** `[059]`
(defined in the generated `i2c-internal.h` file), which has several **static_assert()** calls.
The class `D` is expected to implement the three member functions (**I2cTransact()**,
**I2cGetMaxTransferSize()**, and **I2cGetInterrupt()**) in order for the static methods to work.
If they're not provided by `D`, then the compiler would (in the absence of the static
asserts) produce gnarly templating errors.
The static asserts serve to produce diagnostic errors that are understandable by mere humans. -->
构造函数调用包含数个 **static_assert()** 调用的内部验证函数 **CheckI2cProtocolSubclass()** 第 `[059]` 行（定义在生成的 `i2c-internal.h` 文件中）。为使静态方法可用，类 `D` 需要实现三个成员函数（**I2cTransact()**,
**I2cGetMaxTransferSize()** 和 **I2cGetInterrupt()**）。如果未实现，编译器会产生粗略的模板错误提示（不存在静态断言的时候）。

<!-- Next, the three pointer-to-function operations members (`transact`,
`get_max_transfer_size`, and `get_interrupt`) are bound (lines `[060` .. `062]`). -->
接下来，绑定了3个（第 `[060` 到 `062]` 行）成员操作的函数指针（`transact`,`get_max_transfer_size` 和 `get_interrupt`）

<!-- Finally, the `constexpr` expression provides a default initialization if required. -->
最后，如有需要，`constexpr` 表达式提供了默认的初始化。

<!-- ### Using the mixin class -->
### 使用混合类

<!-- The `I2cProtocol` class can be used as follows (from
[`//src/devices/bus/drivers/platform/platform-proxy.h`](/src/devices/bus/drivers/platform/platform-proxy.h)): -->
 `I2cProtocol` 类可以象下面的方式使用（取自[`//src/devices/bus/drivers/platform/platform-proxy.h`](/src/devices/bus/drivers/platform/platform-proxy.h)):

```c++
[01] class ProxyI2c : public ddk::I2cProtocol<ProxyI2c> {
[02] public:
[03]     explicit ProxyI2c(uint32_t device_id, uint32_t index, fbl::RefPtr<PlatformProxy> proxy)
[04]         : device_id_(device_id), index_(index), proxy_(proxy) {}
[05]
[06]     // I2C protocol implementation.
[07]     void I2cTransact(const i2c_op_t* ops, size_t cnt, i2c_transact_callback transact_cb,
[08]                      void* cookie);
[09]     zx_status_t I2cGetMaxTransferSize(size_t* out_size);
[10]     zx_status_t I2cGetInterrupt(uint32_t flags, zx::interrupt* out_irq);
[11]
[12]     void GetProtocol(i2c_protocol_t* proto) {
[13]         proto->ops = &i2c_protocol_ops_;
[14]         proto->ctx = this;
[15]     }
[16]
[17] private:
[18]     uint32_t device_id_;
[19]     uint32_t index_;
[20]     fbl::RefPtr<PlatformProxy> proxy_;
[21] };
```

<!-- Here we see that `class ProxyI2c` inherits from the DDK's `I2cProtocol` and provides
itself as the argument to the template &mdash; this is the "mixin" concept.
This causes the `ProxyI2c` type to be substituted for `D` in the template definition
of the class (from the `i2c.h` header file above, lines `[084]`, `[088]`, and `[093]`). -->
我们看到 `class ProxyI2c` 从 DDK 的 `I2cProtocol` 类继承，并使自己可作为模板的参数 &mdash; 这就是
“混合类”的概念。这导致 `ProxyI2c` 类型可以取代类模板定义中的 `D` （来自上面的头文件 `i2c.h` 的第  `[084]`， `[088]` 和 `[093]` 行）。

<!-- Taking a look at just the **I2cGetMaxTransferSize()** function as an example, it's
effectively as if the source code read: -->
以 **I2cGetMaxTransferSize()** 函数为例，如源码读来这般高效：

```c++
[087] static zx_status_t I2cGetMaxTransferSize(void* ctx, size_t* out_size) {
[088]     auto ret = static_cast<ProxyI2c*>(ctx)->I2cGetMaxTransferSize(out_size);
[089]     return ret;
[090] }
```

<!-- This ends up eliminating the cast-to-self boilerplate in your code.
This casting is necessary because the type information is erased at the DDK boundary &mdash;
recall that the context `ctx` is a `void *` pointer. -->
这消除了代码中转换为自己的步骤。这个转换有必要，因为类型信息在 DDK 层面就被消除了 &mdash; 想想看，上下文
`ctx` 是一个 `void *` 指针。

<!-- ### Auto-generated comments -->
### 自动生成的注释

<!-- Banjo automatically generates comments in the include file that basically summarize what we
talked about above: -->
Banjo 在包含文件中自动生成了注释，概括了上面提到的内容：

```c++
[020] // DDK i2c-protocol support
[021] //
[022] // :: Proxies ::
[023] //
[024] // ddk::I2cProtocolClient is a simple wrapper around
[025] // i2c_protocol_t. It does not own the pointers passed to it
[026] //
[027] // :: Mixins ::
[028] //
[029] // ddk::I2cProtocol is a mixin class that simplifies writing DDK drivers
[030] // that implement the i2c protocol. It doesn't set the base protocol.
[031] //
[032] // :: Examples ::
[033] //
[034] // // A driver that implements a ZX_PROTOCOL_I2C device.
[035] // class I2cDevice;
[036] // using I2cDeviceType = ddk::Device<I2cDevice, /* ddk mixins */>;
[037] //
[038] // class I2cDevice : public I2cDeviceType,
[039] //                   public ddk::I2cProtocol<I2cDevice> {
[040] //   public:
[041] //     I2cDevice(zx_device_t* parent)
[042] //         : I2cDeviceType(parent) {}
[043] //
[044] //     void I2cTransact(const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie);
[045] //
[046] //     zx_status_t I2cGetMaxTransferSize(size_t* out_size);
[047] //
[048] //     zx_status_t I2cGetInterrupt(uint32_t flags, zx::interrupt* out_irq);
[049] //
[050] //     ...
[051] // };
```

<!-- # Using Banjo -->
# 使用代码生成器 Banjo

<!--
> Suraj says:
>> We also need something in-between a FIDL tutorial and a driver writing tutorial,
>> in order to describe banjo usage.
>> Basically, writing a simple protocol, and then describing a driver that emits
>> it, and another driver that binds on top of it and makes use of that protocol.
>> If it makes sense, the existing driver writing tutorial could just be modified
>> to have more fleshed out details on banjo usage.
>> I think the current driver tutorial is focused on C usage as well, and getting
>> a C++ version (using ddktl) would probably bring the most value [this is
>> already on my work queue, "Tutorial on using ddktl (C++ DDK wrappers)" -RK].
-->
<!-- Now that we've seen the generated code for the I2C driver, let's take a look
at how we would use it. -->
现在我们已经看到了生成的 I2C 驱动的代码，再来看看我们如何使用他。

<!-- > @@@ to be completed -->
> @@@ 待完成

<!-- # Reference -->
# 参考

<!-- > @@@ This is where we should list all builtin keywords and primitive types -->
> @@@ 这里我们应当列出内置的关键字和基础类型

<!-- ## Attributes -->
## 属性

<!-- Recall from the example above that the `protocol` section had two attributes;
a `[Transport = "Banjo", BanjoLayout]` and an `[Async]` attribute. -->
回顾上面的例子中 `protocol` 章节有两个属性，`[Transport = "Banjo", BanjoLayout]` 和 `[Async]` 。

<!-- ### The BanjoLayout attribute -->
### 布局

<!-- The line just before the `protocol` is the `[Transport = "Banjo", BanjoLayout]` attribute: -->
`protocol`的上一行是 `[Transport = "Banjo", BanjoLayout]` 属性

```banjo
[19] [Transport = "Banjo", BanjoLayout = "ddk-protocol"]
[20] protocol I2c {
```

<!-- The attribute applies to the next item; so in this case, the entire `protocol`.
Only one layout is allowed per interface. -->
该属性应用到接下来的项目，也就是整个 `protocol`。每个接口只能有一个布局。

<!-- There are in fact 3 `BanjoLayout` attribute types currently supported: -->
事实上，`BanjoLayout` 目前支持三种类型：

* `ddk-protocol`
* `ddk-interface`
* `ddk-callback`

<!-- In order to understand how these layout types work, let's assume we have two drivers,
`A` and `B`.
Driver `A` spawns a device, which `B` then attaches to, (making `B` a child of `A`). -->
为便于理解这些布局类型如何起作用，我们假定有两个驱动`A` 和 `B`。
驱动  `A` 先生成一个设备，然后 `B` 驱动附着其上（使 `B` 成为 `A` 的子级）

<!-- If `B` then queries the DDK for its parent's "protocol" through **device_get_protocol()**,
it'll get a `ddk-protocol`.
A `ddk-protocol` is a set of callbacks that a parent provides to its child. -->
若之后 `B` 通过 **device_get_protocol()** 向DDK查询其父级的 "protocol" ，将得到一个 `ddk-protocol`。
 `ddk-protocol`是一组父级提供给其子级的回调函数。

<!-- One of the protocol functions can be to register a "reverse-protocol", whereby
the child provides a set of callbacks for the parent to trigger instead.
This is a `ddk-interface`. -->
其中之一的协议函数可以被注册为“反向协议”，由此，子级向父级提供了一组回调函数。这就是一个 `ddk-interface`。


<!-- From a code generation perspective, these two (`ddk-protocol` and `ddk-interface`)
look almost identical, except for some slight naming differences (`ddk-protocol`
automatically appends the word "protocol" to the end of generated structs / classes,
whereas `ddk-interface` doesn't). -->
从代码生成的角度看，这两个（`ddk-protocol` 和 `ddk-interface`）除了有一些细微的名称区别之外（`ddk-protocol` 在生成的代码中的结构/类型的名称后自动加了"protocol"后缀，而 `ddk-interface` 没有），看起来完全一样。

<!-- `ddk-callback` is a slight optimization over `ddk-interface`, and is used when an
interface has just one single function.
Instead of generating two structures, like: -->
`ddk-callback` 在 `ddk-interface` 上做了轻微优化，且仅在接口只有一个函数的时候使用。而不是生成两个结构，如：

```c
struct interface {
   void* ctx;
   inteface_function_ptr_table* callbacks;
};

struct interface_function_ptr_table {
   void (*one_function)(...);
}
```

<!-- a `ddk-callback` will generate a single structure with the function pointer inlined: -->
 `ddk-callback` 将只生成一个含有内联函数指针的结构。

```c
struct callback {
  void* ctx;
  void (*one_function)(...);
};
```

<!-- ### The Async attribute -->
### 异步

Within the `protocl` section, we see another attribute: the `[Async]` attribute:
`protocl` 章节里我们看到另一个属性 `[Async]`：

```banjo
[20] protocl I2c {
...      /// comments (removed)
[27]     [Async]
```

<!-- The `[Async]` attribute is a way to make protocol messages not be synchronous.
It autogenerates a callback type in which the output arguments are inputs to the callback.
The original method will not have any of the output parameters specified in its signatures. -->
 `[Async]` 属性是一种使协议消息不同步的方式。其自动生成一个回调类型。并将输出参数传给回调函数。
 原始的方法在函数签名中不包含任何输出参数。

<!-- Recall from the example above that we had a `Transact` method: -->
回顾上面含有 `Transact` 的例子：

```banjo
[27] [Async]
[28] Transact(vector<I2cOp> op) -> (zx.status status, vector<I2cOp> op);
```

<!-- When used (as above) in conjunction with the `[Async]` attribute, it means that we want Banjo
to invoke a callback function, so that we can handle the output data (the second
`vector<I2cOp>` above, representing the data from the I2C bus). -->
当与 `[Async]` 属性连用时（如上述），意味着Banjo需要调用回调函数，以便我们处理输出数据（上面的第二个 `vector<I2cOp>` 代表从I2C总线出来的数据）。

<!-- Here's how it works.
We send data to the I2C bus through the first `vector<I2cOp>` argument.
Some time later, the I2C bus may generate data in response to our request.
Because we specified `[Async]`, Banjo generates the functions to take a callback function
as input. -->
这是其工作原理：
数据通过第一个 `vector<I2cOp>` 参数传给 I2C 总线。一会之后，I2C 将产生响应数据。
由于指定了 `[Async]`， Banjo 生成一个函数以接收回调函数作为输入参数。

<!-- In C, these two lines (from the `i2c.h` file) are important: -->
C语言里，这两行（在 `i2c.h` 文件中）非常重要：

```c
[19] typedef void (*i2c_transact_callback)(void* ctx, zx_status_t status, const i2c_op_t* op_list, size_t op_count);
...
[36] void (*transact)(void* ctx, const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie);
```

<!-- In C++, we have two place where the callback is referenced: -->
C++语言中，有两处引用了回调函数：

```c++
[083] static void I2cTransact(void* ctx, const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie) {
[084]     static_cast<D*>(ctx)->I2cTransact(op_list, op_count, callback, cookie);
[085] }
```

<!-- and -->
和

```c++
[134] void Transact(const i2c_op_t* op_list, size_t op_count, i2c_transact_callback callback, void* cookie) const {
[135]     ops_->transact(ctx_, op_list, op_count, callback, cookie);
[136] }
```

<!-- Notice how the C++ is similar to the C: that's because the generated code includes the
C header file as part of the C++ header file. -->
注意C++比C简洁：那是因为生成的代码中C头文件作为C++的头文件的一部分被包含了。

<!-- The transaction callback has the following arguments: -->
事务处理回调函数有如下参数：

<!-- Argument   | Meaning
-----------|----------------------------------------
`ctx`      | the cookie
`status`   | status of the asynchronous response (provided by callee)
`op_list`  | the data from the transfer
`op_count` | the number of elements in the transfer -->
参数   | 含义
-----------|----------------------------------------
`ctx`      | 上下文环境
`status`   | 异步相应状态 (被调用者提供)
`op_list`  | 传入的数据
`op_count` | 传递的数量

<!-- How is this different than just using the `ddk-callback` `[Transport = "Banjo", BanjoLayout]` attribute we
discussed above? -->
这与上面刚讨论的使用 `ddk-callback` `[Transport = "Banjo", BanjoLayout]`属性有什么区别？

<!-- First, there's no `struct` with the callback and cookie value in it, they're inlined
as arguments instead. -->
首先，回调中没有 `struct`和上下文信息，他们取而代之，成为内联参数了。

<!-- Second, the callback provided is a "one time use" function.
That is to say, it should be called once, and only once, for each invocation of the
protocol method it was supplied to.
For contrast, a method provided by a `ddk-callback` is a "register once, call
many times" type of function (similar to `ddk-interface` and `ddk-protocol`).
For this reason, `ddk-callback` and `ddk-interface` structures usually have
paired **register()** and **unregister()** calls in order to tell the parent device
when it should stop calling those callbacks. -->
其次，回调提供的是“一次性”的函数。也就是说，对每个协议只能被调用一次。
相较而言， `ddk-callback` 则是 “一次注册，多次调用”的函数（与 `ddk-interface` 和 `ddk-protocol` 类似）。
由此原因，`ddk-callback` 和 `ddk-interface` 结构常有成对的 **register()** 和 **unregister()** 调用以
通告父级设备应当何时不再调用那些回调函数。

<!-- > One more caveat with `[Async]` is that its callback *MUST* be called for each
> protocol method invocation, and the accompanying cookie must be provided.
> Failure to do so will result in undefined behavior (likely a leak, deadlock,
> timeout, or crash). -->
> 再次提醒 `[Async]` 的回调 *应当* 对每个协议方法都调用，且提供相应的上下文信息。
> 如果没做此步骤将导致未定义的行为（或许是内存泄漏、死锁、超时或崩溃）

<!-- Although not the case currently, C++ and future language bindings (like Rust)
will provide "future" / "promise" style based APIs in the generated code, built on top of
these callbacks in order to prevent mistakes. -->
尽管目前不是如此，C++和之后的语言绑定（如Rust）将在生成的代码中提供 "future" / "promise" 风格的 API，
基于此类的回调构建以防止错误。

<!-- > Ok, one more caveat with `[Async]` &mdash; the `[Async]` attribute applies *only*
> to the immediately following method; not any other methods. -->
> 好了，再次提醒 `[Async]` &mdash; `[Async]` 属性 *仅适用* 于立即方法，而不是其他的方法。  
>

<!-- ### The Buffer attribute -->
### 缓冲区

<!-- This attribute applies to protocol method parameters of the `vector` type to convey that they are
used as buffers. In practice, it only affects the names of the generated parameters. -->
此属性适用于有 `vector` 类型参数的协议方法，以用缓冲区。实践中，只影响生成的参数的名称。

<!-- ### The CalleeAllocated attribute -->
### 被调函数分配的属性

<!-- When applied to a protocol method output parameter of type `vector`, the attribute conveys the fact
that the contents of the vector should be allocated by the receiver of the method call. -->
当 `vector`类型的输出参数应用到一个协议方法上时，其属性传达了这样一个信息，那就是向量的内容是由被调用函数负责分配内存。

<!-- ### The InnerPointer attribute -->
### 内部指针属性

<!-- In the context of a protocol input parameter of type `vector`, this attribute turns the contents of
the vector into pointers to objects instead of objects themselves. -->
在协议输入参数为 `vector`类型的情况下，本属性将向量的内容从对象本身转成了指向对象的指针。

<!-- ### The InOut attribute -->
### 输入输出属性

<!-- Adding this attribute to a protocol method input parameter makes the parameter mutable, effectively
turning it into an "in-out" parameter. -->
一个协议方法的输入参数上加上本属性后，会使该参数可被修改，有效地使其变成一个“输入输出”参数。

<!-- ### The Mutable attribute -->
### 可变属性

<!-- This attribute should be used to make `struct`/`union` fields of type `vector` or `string` mutable. -->
本属性应该用于使拥有`vector` 或 `string` 类型的 `struct`/`union` 字段可修改。

<!-- ### The OutOfLineContents attribute -->
### OutOfLineContents属性

<!-- This attribute allows the contents of a `vector` field in a `struct`/`union` to be stored outside
of the container. -->
本属性允许 `struct`/`union`类型的`vector`字段在容器外存储。

<!-- ### The PreserveCNames attribute -->
### PreserveCNames 属性

<!-- This attribute applies to `struct` declarations and makes it so that their fields' names remain
unchanged when run through the C backend. -->
本属性适用于 `struct` 声明和构造，以便在底层使用C时保持字段名称不变。

<!-- # Banjo Mocks -->
# Banjo 模拟

<!-- Banjo generates a C++ mock class for each protocol. This mock can be passed to protocol users in
tests. -->
Banjo 对每个协议生成一个C++模拟类。测试中，这个模拟类可以被传递给协议用户。

<!-- ## Building -->
## 构建

<!-- Tests in Zircon get the mock headers automatically. Tests outsize of Zircon must depend on the
protocol target with a `_mock` suffix, e.g. -->
Zircon中的测试自动获取模拟类的头文件。Zircon之外的测试须依赖有`_mock`后缀的协议对象，如：
`//zircon/public/banjo/fuchsia.hardware.gpio:fuchsia.hardware.gpio_banjo_cpp_mock`.

<!-- ## Using the mocks -->
## 使用模拟器

<!-- Test code must include the protocol header with a `mock/` prefix, e.g. -->
测试代码须包含有`mock/`为前缀的协议头文件，如：
`#include <fuchsia/hardware/gpio/cpp/banjo-mock.h>`.

<!-- Consider the following Banjo protocol snippet: -->
思考如下Banjo协议代码片段：

```banjo
[021] [Transport = "Banjo", BanjoLayout = "ddk-protocol"]
[022] protocol Gpio {
 ...
[034]     /// Gets an interrupt object pertaining to a particular GPIO pin.
[035]     GetInterrupt(uint32 flags) -> (zx.status s, handle<interrupt> irq);
 ...
[040] };
```

<!-- Here are the corresponding bits of the mock class generated by Banjo: -->
下面是Banjo生成的模拟类对应的代码：

```c++
[034] class MockGpio : ddk::GpioProtocol<MockGpio> {
[035] public:
[036]     MockGpio() : proto_{&gpio_protocol_ops_, this} {}
[037]
[038]     const gpio_protocol_t* GetProto() const { return &proto_; }
 ...
[065]     virtual MockGpio& ExpectGetInterrupt(zx_status_t out_s, uint32_t flags, zx::interrupt out_irq) {
[066]         mock_get_interrupt_.ExpectCall({out_s, std::move(out_irq)}, flags);
[067]         return *this;
[068]     }
 ...
[080]     void VerifyAndClear() {
 ...
[086]         mock_get_interrupt_.VerifyAndClear();
 ...
[089]     }
 ...
[117]     virtual zx_status_t GpioGetInterrupt(uint32_t flags, zx::interrupt* out_irq) {
[118]         std::tuple<zx_status_t, zx::interrupt> ret = mock_get_interrupt_.Call(flags);
[119]         *out_irq = std::move(std::get<1>(ret));
[120]         return std::get<0>(ret);
[121]     }
```

<!-- The MockGpio class implements the GPIO protocol. `ExpectGetInterrupt`
is used to set expectations on how `GpioGetInterrupt` is called. `GetProto` is used to get the
`gpio_protocol_t` that can be passed to the code under test. This code will call `GpioGetInterrupt`
which will ensure that it got called with the correct arguments and will return the value specified
by `ExpectGetInterrupt`. Finally, the test can call `VerifyAndClear` to verify that all expectations
were satisfied. Here is an example test using this mock: -->
MockGpio类实现了GPIO协议。`ExpectGetInterrupt` 用于设定期望`GpioGetInterrupt`如何被调用。 `GetProto`用于
获取`gpio_protocol_t`的值，该值可以传给测试代码。测试代码将调用 `GpioGetInterrupt`，其确保被正确的参数调用，
且返回`ExpectGetInterrupt`指明的值。最后，测试调用 `VerifyAndClear` 以校验满足了所有预期。下面时一个使用这个
模拟器的测试样例：

```c++
TEST(SomeTest, SomeTestCase) {
    ddk::MockGpio gpio;

    zx::interrupt interrupt;
    gpio.ExpectGetInterrupt(ZX_OK, 0, zx::move(interrupt))
        .ExpectGetInterrupt(ZX_ERR_INTERNAL, 100, zx::interrupt());

    CodeUnderTest dut(gpio.GetProto());
    EXPECT_OK(dut.DoSomething());

    ASSERT_NO_FATAL_FAILURES(gpio.VerifyAndClear());
}
```

<!-- ### Equality operator overrides -->
### 相等操作符重载

<!-- Tests using Banjo mocks with structure types will have to define equality operator overrides. For
example, for a struct type `some_struct_type` the test will have to define a function with the
signature -->
使用含有结构类型数据的 Banjo 模拟器的测试，需要重载相等操作符。例如，对于一个`some_struct_type`类型的结构，测试须定义一个如下形式的函数：

```c++
bool operator==(const some_struct_type& lhs, const some_struct_type& rhs);
```

<!-- in the top-level namespace. -->
在顶层命名空间中。

<!-- ### Custom mocks -->
### 自定义模拟器

<!-- It is expected that some tests may need to alter the default mock behavior. To help with this, all
expectation and protocol methods are `virtual`, and all `MockFunction` members are `protected`. -->
如期望的，有些测试可能需要修改模拟器的默认行为。为达此目的，所有的期望值和协议方法都是`virtual`的，并且所有 `MockFunction`成员都是`protected`的。

<!-- ### Async methods -->
### 异步方法

<!-- The Banjo mocks issue callbacks from all async methods by default. -->
Banjo 模拟器默认情况下会在异步方法中调用回调函数。
