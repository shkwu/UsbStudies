```c
/** @file  
  The Ehci controller driver.

  EhciDxe driver is responsible for managing the behavior of EHCI controller. 
  It implements the interfaces of monitoring the status of all ports and transferring 
  Control, Bulk, Interrupt and Isochronous requests to Usb2.0 device.

  Note that EhciDxe driver is enhanced to guarantee that the EHCI controller get attached
  to the EHCI controller before a UHCI or OHCI driver attaches to the companion UHCI or 
  OHCI controller.  This way avoids the control transfer on a shared port between EHCI 
  and companion host controller when UHCI or OHCI gets attached earlier than EHCI and a 
  USB 2.0 device inserts.

**/
```
# 1. Driver Binding Protocol
```c
//
// This protocol provides the services required to determine if a driver supports a given controller.
// If a controller is supported, then it also provides routines to start and stop the controller.
//
EFI_DRIVER_BINDING_PROTOCOL
gEhciDriverBinding = {
  EhcDriverBindingSupported,
  EhcDriverBindingStart,
  EhcDriverBindingStop,
  0x30,                 // Version number
  NULL,                 // ImageHandle
  NULL                  // DriverBindingHandle
};
```
## 1.1 Version Number
关于 version number of the UEFI driver, 当一个 controller 需要被 started 时，EFI boot service - ConnectController() 使用这个 version number 来决定该 UEFI driver 的 Support() 服务被调用的 order。

## 1.2 Image Handle
> The image handle of the UEFI driver that produced this instance of the EFI_DRIVER_BINDING_PROTOCOL.

## 1.3 Driver Binding Handle
> The handle on which this instance of the EFI_DRIVER_BINDING_PROTOCOL is installed. In most cases, this is the same handle as ImageHandle. However, for UEFI drivers that produce more than one instance of the EFI_DRIVER_BINDING_PROTOCOL, this value may not be the same as ImageHandle. 


# 2. EhcDriverBindingSupported()

```c
//
// Test to see if this driver supports ControllerHandle. 
// Any ControllerHandle that has Usb2HcProtocol installed will be supported.
//
EFI_STATUS
EFIAPI
EhcDriverBindingSupported (
  IN EFI_DRIVER_BINDING_PROTOCOL *This,
  IN EFI_HANDLE                  Controller, // Handle of device to test.
  IN EFI_DEVICE_PATH_PROTOCOL    *RemainingDevicePath
  )
{

    // Test whether there is PCI IO Protocol attached on the controller handle.
    |-- Status = gBS->OpenProtocol(&gEfiPciIoProtocolGuid);  // return &PciIo

    // Use PciIo to read PCI Class Code
    // PCI_CLASSCODE_OFFSET = 0x09
    |-- Status = PciIo->Pci.Read(USB_CLASSC)  // return &UsbClassCReg
  
    // Test whether the controller belongs to Ehci type
    |-- if ((UsbClassCReg.BaseCode != PCI_CLASS_SERIAL) ||
         (UsbClassCReg.SubClassCode != PCI_CLASS_SERIAL_USB) || 
         ((UsbClassCReg.ProgInterface != PCI_IF_EHCI) && 
         (UsbClassCReg.ProgInterface != PCI_IF_UHCI) && 
         (UsbClassCReg.ProgInterface != PCI_IF_OHCI)))  // If True, return EFI_UNSUPPORTED
  
    // If EFI_ERROR (Status) == True, goto ON_EXIT;
    |-- ON_EXIT: gBS->CloseProtocol(&gEfiPciIoProtocolGuid)
    |-- return Status;
}
```

## 2.1 EFI_PCI_IO_PROTOCOL
Please refer to `(EFI) EFI_PCI_IO_PROTOCOL.md`.

# 3. EhcDriverBindingStart()
```c
//
// Starting the Usb EHCI Driver.
// 
EFI_STATUS
EFIAPI
EhcDriverBindingStart (
  IN EFI_DRIVER_BINDING_PROTOCOL *This,
  IN EFI_HANDLE                  Controller,
  IN EFI_DEVICE_PATH_PROTOCOL    *RemainingDevicePath
  )
{
  // Open the PciIo Protocol, then enable the USB host controller
  |-- Status = gBS->OpenProtocol(&gEfiPciIoProtocolGuid)
  
  // Open Device Path Protocol for on USB host controller
  // HcDevicePath = NULL;
  |-- Status = gBS->OpenProtocol(&gEfiDevicePathProtocolGuid)  // Retrieve (VOID **) &HcDevicePath

  // Save original PCI attributes
  |-- Status = PciIo->Attributes(EfiPciIoAttributeOperationGet) // Retrieve &OriginalPciAttributes
}
```
## 3.1 EFI_PCI_IO_PROTOCOL_ATTRIBUTE_OPERATION


