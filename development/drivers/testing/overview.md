<!-- # Driver testing -->
# 驱动测试

<!-- ## Manual hardware unit tests -->
## 人工硬件单元测试

<!-- A driver may choose to implement the `run_unit_tests()` driver op, which
provides the driver a hook in which it may run unit tests at system
initialization with access to the parent device. This means the driver may test
its bind and unbind hooks, as well as any interactions with real hardware. If
the tests pass (the driver returns `true` from the hook) then operation will
continue as normal and `bind()` will execute. If the tests fail then the device
manager will assume that the driver is invalid and never attempt to bind it. -->
一个驱动可能选择实现 `run_unit_tests()`驱动操作，这提供了一个启动勾子，用于在系统初始化
过程中运行单元测试并访问父级设备。这意味着驱动可以测试其绑定和解绑勾子，以及和真实硬件交互。
如果测试通过（驱动从勾子中返回 `true`）之后操作正常继续，并且执行`bind()`。如果测试失败，
设备管理器会认为驱动无效并且不再尝试去绑定它。

<!-- Since these tests must run at system initialization (in order to not interfere
with the usual operation of the driver) they are activated with a
[kernel command line flag](/docs/reference/kernel/kernel_cmdline.md). To enable
the hook for a specific driver, use `driver.<name>.tests.enable`. Or for all
drivers: `driver.tests.enable`. If a driver doesn't implement `run_unit_tests()`
then these flags will have no effect. -->
由于这些测试须在系统初始化过程中运行（避免影响驱动的常规操作），他们带有一个[内核命令行标记](/docs/reference/kernel/kernel_cmdline.md)。要为一个特定驱动开启勾子，可以使用`driver.<name>.tests.enable`。
或者为所有驱动开启勾子：`driver.tests.enable`。如果一个驱动并未实现`run_unit_tests()`，这些标记
则没有任何作用。

<!-- `run_unit_tests()` passes the driver a channel for it to write test output to.
Test output should be in the form of `fuchsia.driver.test.Logger` FIDL messages.
The driver-unit-test library contains a [helper class] that integrates with
zxtest and handles logging for you. -->
`run_unit_tests()`传递一个通道给驱动以供驱动输出测试结果信息。测试结果应当遵循`fuchsia.driver.test.Logger`
的FIDL消息格式。驱动单元测试库包含一个[帮助类]其集成了zxtest和日志处理器。

<!-- [helper class]: /zircon/system/ulib/driver-unit-test/include/lib/driver-unit-test/logger.h -->
[帮助类]: /zircon/system/ulib/driver-unit-test/include/lib/driver-unit-test/logger.h


<!-- ## Integration tests -->
## 集成测试

<!-- Driver authors can use several means for writing integration tests. For simple
cases, the [fake-ddk](/src/devices/testing/fake_ddk) library is recommended. For
more complicated ones,
 [isolated-devmgr](/src/lib/isolated_devmgr) is
recommended. -->
驱动开发者可以借助多种方式编写集成测试。对于一些简单例子，推荐使用[fake-ddk](/src/devices/testing/fake_ddk)
库。对于一些更复杂的，推荐使用 [isolated-devmgr](/src/lib/isolated_devmgr)库。

<!-- TODO(fxbug.dev/51320): Fill out more detail here. -->
TODO(fxbug.dev/51320)：在此填写更多细节。
