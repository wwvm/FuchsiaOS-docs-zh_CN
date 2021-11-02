<!--
    (C) Copyright 2019 The Fuchsia Authors. All rights reserved.
    Use of this source code is governed by a BSD-style license that can be
    found in the LICENSE file.
-->

<!-- # Getting descriptors and endpoints from USB -->
# 从USB获取描述符和端点

<!-- The `usb` class contains several subclasses providing access to the interfaces, descriptors,
and endpoints of the usb device. The subclasses included are: -->
`usb`类包含几个供访问USB设备接口，描述符及端点的子类。包括：

*   [`InterfaceList`](/src/devices/usb/lib/usb/include/usb/usb.h#311)
*   [`Interface`](/src/devices/usb/lib/usb/include/usb/usb.h#290)
*   [`DescriptorList`](/src/devices/usb/lib/usb/include/usb/usb.h#166)
*   [`EndpointList`](/src/devices/usb/lib/usb/include/usb/usb.h#266)

<!-- USB descriptor report all of the device's attributes. An endpoint is a specific type of descriptor
that describes the terminus of a communication flow between the host and the device. -->
USB描述符提供了设备的所有信息。端点是描述符的特例，描述了宿主机和设备间通信流的终点。

<!-- The `InterfaceList` class iterates over each `Interface` within the usb device.
Each `Interface` then contains: -->
`InterfaceList`类依次迭代USB设备内的各个接口 `Interface`。
每个接口包含：

<!-- *   `DescriptorList` through `GetDescriptorList()`
*   `EndpointList` through `GetEndpointList()` -->
*  通过 `GetDescriptorList()` 获取 `DescriptorList` 
*  通过 `GetEndpointList()` 获取 `EndpointList`

<!-- These methods allow access to all the descriptors and endpoints of the interface. -->
这些方法允许访问接口的所有描述符和端点。

<!-- Note: Endpoints are still considered descriptors and therefore can also be
accessible through the `GetDescriptorList()` method. -->
注意：端点也是描述符，意味着也可以通过 `GetDescriptorList()` 方法访问。

<!-- The hierarchy of these subclasses can be seen in **Figure 1.** -->
这些子类的继承关系可以参看**Figure 1.** 

![drawing](images/usbstructure.jpg)

**Figure 1**

<!-- ## Examples -->
## 示例

<!-- ### Receiving Descriptors from USB -->
### 从USB接收描述符

<!-- These examples iterate through all the descriptors in a USB device. The example iterates through all
of the USB Interfaces and then iterates through all the descriptors in each interface. -->
这些示例遍历USB设备的所有描述符。实例遍历USB接口，并遍历每个接口的描述符。

<!-- Note: To iterate through the endpoints instead, replace `interface.getDescriptorList()` with `interface.getEndpointList()`. -->
注意：若要遍历端点，请将`interface.getDescriptorList()` 替换成 `interface.getEndpointList()`

<!-- #### Range-based for loop -->
### 基于范围的for循环

    std::optional<InterfaceList> interface_list;

    status = InterfaceList::Create(my_client, true, &interface_list);

    if (status != ZX_OK) {
        ...
    }

    for (auto& interface : *interface_list) {

        for (auto& descriptor : interface.GetDescriptorList()) {
            ...
        }
    }

<!-- #### Manual for loop -->
### 手动for循环

    std::optional<InterfaceList> interface_list;

    status = InterfaceList::Create(my_client, true, &interface_list);

    if (status != ZX_OK) {
        ...
    }

    for (auto& interface : *interface_list) {

        auto dList_itr = interface.GetDescriptorList().begin(); // or cbegin().

        do {
            ...
        } while (++dList_itr != interface.GetDescriptorList().end());
    }
