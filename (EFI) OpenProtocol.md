# OpenProtocol()

UEFI spec 2.7 - ch7.3

device handle 上都会安装一个 PCI IO Protocol
```c
//
// Test whether there is PCI IO Protocol attached on the controller handle.
// 1. Opens PCI I/O Protocol on Controller(handle).
// 2. The driver that is opening the protocol is identified 
//   by the Driver Binding Protocol instance This.
// 3. This->DriverBindingHandle identifies the agent that is opening the 
//   protocol instance, and it is opening this protocol on behalf of Controller(handle).
//
 Status = gBS->OpenProtocol (
                  Controller,                // Handle
                  &gEfiPciIoProtocolGuid,    // Protocol
                  (VOID **) &PciIo,          // Interface
                  This->DriverBindingHandle, // AgentHandle
                  Controller,                // ControllerHandle
                  EFI_OPEN_PROTOCOL_BY_DRIVER// Attributes
                  );
```
Note: Handle是Protocol的提供者，如果Handle的Protocols链表中有该Protocol，则Protocol对象的指针写到`*Interface`，并返回`EFI_SUCCESS`，否则返回`EFI_UNSUPPORTED`。

如果在driver中调用`OpenProtocol()`，则`ControllerHandle`是拥有该driver的controller，也就是请求使用这个Protocol的控制器；`AgentHandle`是拥有该`EFI_DRIVER_BINGING_PROTOCOL`对象的Handle。`EFI_DRIVER_BINGING_PROTOCOL`是UEF Driver开发中一定会用到的一个Protocol，它负责driver的安装与卸载。

如果调用`OpenProtocol()`的是Application，那么`AgentHandle`是该Application的Handle，也就是UefiMain函数的第一个参数，而`ControllerHandle`此时可以忽略。