<!-- # Retrieve build information -->
# 收集构建信息

<!-- Metrics and error reports are collected from devices in several ways:
Cobalt, feedback reports, crash reports, manual reports from developers
and QA.  Interpreting these signals requires knowing where they are generated
from to varying levels of detail.  This document describes the places where
version information about the system are stored for use in these types of
reports.  Note that this information only applies to the base system -
dynamically or ephemerally added software will not be included here. -->
通过多种方式搜集度量和错误信息：Cobalt，反馈报告，奔溃报告，开发者的手动报告及问答。
解释这些信号要求了解他们产自哪里，以及不同程度的细节。本文档描述了供此类运用的
系统版本信息存放的位置。注意这些信息适用于基础系统。动态或临时假如的软件不包含在内。


<!-- To view this data via the commandline, you can use `fx shell`. For example: -->
可以通过命令行命令 `fx shell` 查看此部分信息。例如：

```sh
fx shell cat /config/build-info/latest-commit-date
```

<!-- To access this data at runtime, add the feature "build-info" to the
[component manifest][component-manifest] of the component that needs to
read these fields. -->
如运行时要访问此信息，可以在要访问的组件的[组件清单][component-manifest]里加入
"build-info" 特性。

<!-- 
## Product
### Location
`/config/build-info/product` -->
## 产品
### 位置
`/config/build-info/product`

<!-- ### Description -->
### 说明
<!-- String describing the product configuration used at build time.  Defaults to the value passed as PRODUCT in fx set.
Example: “products/core.gni”, “products/workstation.gni” -->
构建时指定的用于说明产品配置的字符串。默认为通过 PRODUCT 传给 fx set 命令的信息。例如：
“products/core.gni”， “products/workstation.gni”

<!-- ## Board
### Location -->
## 板子
### 位置
`/config/build-info/board`

<!-- ### Description -->
### 说明
<!-- String describing the board configuration used at build time to specify the target hardware.  Defaults to the value passed as BOARD in fx set.
Example: “boards/x64.gni” -->
构建时配置的用于说明目标硬件的配置信息的字符串。默认为通过 BOARD 传给 fx set 命令的信息。例如：“boards/x64.gni”

<!-- ## Version
### Location -->
## 版本
### 位置
`/config/build-info/version`

<!-- ### Description -->
### 说明
<!-- String describing the version of the build.  Defaults to the same string used currently in ‘latest-commit-date’.  Can be overridden by build infrastructure to provide a more semantically meaningful version, e.g. to include the release train the build was produced on. -->
构建版本说明的字符串。默认与 ‘latest-commit-date’ 中使用的字符串相同。可以被构建基础设施改写以提供语义更清晰的版本。例如，生成构建的发布序列信息。

<!-- ## Latest-commit-date
### Location -->
## 最后提交时间
### 位置
`/config/build-info/latest-commit-date`

<!-- ### Description -->
### 说明
<!-- String containing a timestamp of the most recent commit to the integration repository (specifically, the "CommitDate" field) formatted in strict ISO 8601 format in the UTC timezone.  Example: “2019-03-28T15:42:20+00:00”. -->
集成仓库中最近提交的时间戳的字符串（特别地，是"CommitDate"字段），格式化为采用 UTC 时区制的严格遵循 ISO 8601 规范格式的字符串。例如：“2019-03-28T15:42:20+00:00”。

<!-- ## Snapshot
### Location -->
## 快照
### 位置
`/config/build-info/snapshot`

<!-- ### Description
Jiri snapshot of the most recent ‘jiri update’ -->
### 说明
最近的 ‘jiri update’ 命令的 Jiri 快照

<!-- ## Kernel version -->
## 内核版本

<!-- ### Location -->
### 位置
<!-- Stored in vDSO.  Accessed through [`zx_system_get_version_string`]( /docs/reference/syscalls/system_get_version_string.md) -->
存于 vDSO 中，可以通过[`zx_system_get_version_string`]( /docs/reference/syscalls/system_get_version_string.md)访问。

<!-- ### Description -->
### 描述
<!-- Zircon revision computed during the kernel build process. -->
内核构建过程中计算得到的 Zircon 版本。 

<!-- [component-manifest]: /docs/concepts/components/v1/component_manifests.md -->
[组件清单]：/docs/concepts/components/v1/component_manifests.md
