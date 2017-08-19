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

    // Use PciIo to read PCI Class Code, PCI_CLASSCODE_OFFSET = 0x09
    // Please refer to `(EFI) EFI_PCI_IO_PROTOCOL.md`.
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
  |-- Status = gBS->OpenProtocol(&gEfiDevicePathProtocolGuid)  // Retrieve &HcDevicePath

  // Save original PCI attributes, restore OriginalPciAttributes to PCI device when `CLOSE_PCIIO`.
  |-- Status = PciIo->Attributes(EfiPciIoAttributeOperationGet) // Retrieve &OriginalPciAttributes, see 3.1
  |-- Status = PciIo->Attributes(EfiPciIoAttributeOperationSupported)//Retrieve all supported attributes &Supports

  // EFI_PCI_DEVICE_ENABLE = EFI_PCI_IO_ATTRIBUTE_IO|EFI_PCI_IO_ATTRIBUTE_MEMORY|EFI_PCI_IO_ATTRIBUTE_BUS_MASTER
  |-- Supports &= (UINT64)EFI_PCI_DEVICE_ENABLE;
  // Enable the PCI device attributes for further access.
  |-- Status = PciIo->Attributes(EfiPciIoAttributeOperationEnable, Supports)

  // Get the Pci device class code.
  |-- Status = PciIo->Pci.Read(USB_CLASSC)  // Retrieve &UsbClassCReg
  // Determine if the device is UHCI or OHCI host controller or not.
  // If False, it means the device is EHCI host controller, goto line 142
  |-- if ((UsbClassCReg.ProgInterface == PCI_IF_UHCI || 
           UsbClassCReg.ProgInterface == PCI_IF_OHCI) &&
           (UsbClassCReg.BaseCode == PCI_CLASS_SERIAL) && 
           (UsbClassCReg.SubClassCode == PCI_CLASS_SERIAL_USB))
      // If True, find out the companion usb ehci host controller
      |-- Status = PciIo->GetLocation(&CompanionSeg, &CompanionBus, &CompanionDev, &CompanionFunc)  // See 3.2
  
      // Returns a buffer contains an array of handles that support the requested protocol. See 3.3
      |-- Status = gBS->LocateHandleBuffer(ByProtocol,&gEfiPciIoProtocolGuid,NULL,&NumberOfHandles,&HandleBuffer)
      // Find out the EHCI host controller
      |-- for (Index = 0; Index < NumberOfHandles; Index++)
          // Queries a handle to determine if it supports a specified protocol. See 3.4 
          |-- Status = gBS->HandleProtocol(HandleBuffer[Index], &gEfiPciIoProtocolGuid, (VOID **)&Instance)
          |-- Status = Instance->Pci.Read(USB_CLASSC)   // Retrieve &UsbClassCReg
          // // Determine if the device is EHCI host controller or not.
          |-- if ((UsbClassCReg.ProgInterface == PCI_IF_EHCI) &&
                  (UsbClassCReg.BaseCode == PCI_CLASS_SERIAL) && 
                  (UsbClassCReg.SubClassCode == PCI_CLASS_SERIAL_USB))
              // If True, retrieves EHCI controller’s seg#, bus#, device#, and function#.
              |-- Status = Instance->GetLocation()
              |-- if (EhciBusNumber == CompanionBusNumber)  // If True, start EHCI controller
                  |-- gBS->CloseProtocol(&gEfiPciIoProtocolGuid) // Close PCI_IO_PROTOCOL
                  |-- EhcDriverBindingStart(This, HandleBuffer[Index], NULL); // Start EHCI first.
      
      // If executing here, it is out of 'for' cycle, means no found EHCI host controller
      |-- Status = EFI_NOT_FOUND; and goto CLOSE_PCIIO;  // Exit
  
  // If executing here, it means EHCI host controller be found out.
  // Create then install USB2_HC_PROTOCOL. 
  |-- Ehc = EhcCreateUsb2Hc (PciIo, HcDevicePath, OriginalPciAttributes); // Refer to 'EhcCreateUsb2Hc.md'.
  
  // #define EHC_BIT_IS_SET(Data, Bit) ((BOOLEAN)(((Data) & (Bit)) == (Bit)))
  |-- if (EHC_BIT_IS_SET (Ehc->HcCapParams, HCCP_64BIT))  // True, the controller supports 64-bit addressing.
      // #define EFI_PCI_IO_ATTRIBUTE_DUAL_ADDRESS_CYCLE  0x8000
      // Enable 64-bit DMA support 'in the PCI layer' by setting EFI_PCI_IO_ATTRIBUTE_DUAL_ADDRESS_CYCLE bit.
      |-- Status = PciIo->Attributes(EfiPciIoAttributeOperationEnable, EFI_PCI_IO_ATTRIBUTE_DUAL_ADDRESS_CYCLE)
  
  // Install a protocol interface (gEfiUsb2HcProtocolGuid, Ehc->Usb2Hc) on the device handle (Controller).
  |-- Status = gBS->InstallProtocolInterface(      // See 3.5
                  &Controller,                     // device handle
                  &gEfiUsb2HcProtocolGuid,         // protocol guid
                  EFI_NATIVE_INTERFACE,            // interface type
                  &Ehc->Usb2Hc                     // protocol interface
                  );
  
  // PcdValue set in .dsc and consume in .inf
  |-- if (FeaturePcdGet (PcdTurnOffUsbLegacySupport))  // If PcdTurnOffUsbLegacySupport = True
      // Stop the legacy USB SMI support by setting HC OS Owned Semaphore bit ???
      // When set HC OS Owned Semaphore bit to 1, Bios has no control of EHCI controller.
      |-- EhcClearLegacySupport (Ehc);   // See 'EhcClearLegacySupport.md'.
  
  // Read EHCI debug port register. See EhcGetUsbDebugPortInfo() in 'EhcCreateUsb2Hc.md'.
  |-- if (Ehc->DebugPortNum != 0) -> State = EhcReadDbgRegister(Ehc, 0);
      // Debug Port Control Register, in use and owned by EHCI controller.
      |-- if ((State & (USB_DEBUG_PORT_IN_USE | USB_DEBUG_PORT_OWNER)) != 
              (USB_DEBUG_PORT_IN_USE | USB_DEBUG_PORT_OWNER))
          |-- EhcResetHC (Ehc, EHC_RESET_TIMEOUT);  // Refer to 'EhcResetHC.md'.
  
  // Initialize the HC hardware. Refer to `EhcInitHC.md`.
  |-- Status = EhcInitHC (Ehc); 

  // Start the asynchronous interrupt monitor. See 3.6 
  |-- Status = gBS->SetTimer (Ehc->PollTimer, TimerPeriodic, EHC_ASYNC_POLL_INTERVAL);
  // If EFI_ERROR (Status), then halt HC.
      |-- EhcHaltHC (Ehc, EHC_GENERIC_TIMEOUT);  // Refer to 'EhcHaltHC.md'.
  
  // Creates an event (&Ehc->ExitBootServiceEvent) in a group (&gEfiEventExitBootServicesGuid).
  // In a EventGroup, once one event is signaled, all other events are signaled.
  |-- Status = gBS->CreateEventEx ( // Refer to 'Events & TPL.md'.
          EVT_NOTIFY_SIGNAL,  // Type of event to create
          TPL_NOTIFY,         // Task priority level of events
          EhcExitBootService, // One notified function to reset the HC when gBS->ExitBootServices() called.
          Ehc,                // Parameter Context pass to EhcExitBootService
          &gEfiEventExitBootServicesGuid,  // EventGroup
          &Ehc->ExitBootServiceEvent       // Return the newly created event.
          );
  
  // Install the component name protocol
  |-- AddUnicodeString2(L"Enhanced Host Controller (USB 2.0)");
  
  // EhcDriverBindingStart: EHCI started success.
  return EFI_SUCCESS;
}
```
*Comment:*
1. Open `gEfiPciIoProtocolGuid` and `gEfiDevicePathProtocolGuid`
2. Set `EFI_PCI_DEVICE_ENABLE` by `PciIo->Attributes` to enable PCI Device (Host Controller)
3. Get the Pci device class code by `PciIo->Pci.Read(USB_CLASSC)` to determine if the device is UHCI or OHCI host controller or not.
    + True, find out EHCI host controller, and call `EhcDriverBindingStart` or return `EFI_NOT_FOUND`.
    + False, goto Setp 4.
4. Call `EhcCreateUsb2Hc` to create `EFI_USB2_HC_PROTOCOL`.
5. If `HCCP_64BIT` set in `Ehc->HcCapParams`, enable 64-bit DMA support in the PCI layer by `PciIo->Attributes()`.
6. Install `EFI_USB2_HC_PROTOCOL` instance `Ehc->Usb2Hc`.
7. If `PcdTurnOffUsbLegacySupport = True` (default: Turn On), call `EhcClearLegacySupport(Ehc)` to set `USBLEGSUP[HC OS Owned Semaphore] = 1` and wait `USBLEGSUP[HC BIOS Owned Semaphore] = 0`.
8. Check Ehc Debug Port register, if in use and owned by EHCI controller or not.
9. Initialize the HC hardware by `EhcInitHC (Ehc)`.
10. Set periodic poll timer for `Async Interrupt Transfer Request` by `gBS->SetTimer (Ehc->PollTimer, TimerPeriodic, EHC_ASYNC_POLL_INTERVAL)`.
11. Create an event `Ehc->ExitBootServiceEvent` which is used to stop HC when `gBS->ExitBootServices()` called.
12. Install the component name protocol.

Note: Force EHCI driver get attached to EHCI host controller before UHCI or OHCI driver attaches to UHCI or OHCI host controller.

## 3.1 EFI_PCI_IO_PROTOCOL_ATTRIBUTE_OPERATION
```c
//
// EFI_PCI_IO_PROTOCOL_ATTRIBUTE_OPERATION 
//
typedef enum {
  // Retrieve the PCI controller's current attributes, and return them in Result.
  EfiPciIoAttributeOperationGet,

  // Set the PCI controller's current attributes to Attributes.
  EfiPciIoAttributeOperationSet,

  // Enable the attributes specified by the bits that are set in Attributes for this PCI controller.
  EfiPciIoAttributeOperationEnable,

  // Disable the attributes specified by the bits that are set in Attributes for this PCI controller.
  EfiPciIoAttributeOperationDisable,

  // Retrieve the PCI controller's supported attributes, and return them in Result.
  EfiPciIoAttributeOperationSupported,
  EfiPciIoAttributeOperationMaximum
} EFI_PCI_IO_PROTOCOL_ATTRIBUTE_OPERATION;


// Performs an operation on the attributes that this PCI controller supports. 
// The operations includegetting the set of supported attributes, retrieving 
// the current attributes, setting the current attributes, enabling attributes,
// and disabling attributes.
typedef
EFI_STATUS
(EFIAPI *EFI_PCI_IO_PROTOCOL_ATTRIBUTES)(
  IN EFI_PCI_IO_PROTOCOL                       *This,
  IN  EFI_PCI_IO_PROTOCOL_ATTRIBUTE_OPERATION  Operation,
  IN  UINT64                                   Attributes, // The mask of attributes that are used for 
                                                           // Set, Enable, and Disable operations.
  OUT UINT64                                   *Result OPTIONAL
  );
```
## 3.2 EFI_PCI_IO_PROTOCOL.GetLocation()
```c
//
// Retrieves this PCI controller’s current PCI bus number, device number, and function number.
//
typedef
EFI_STATUS
(EFIAPI *EFI_PCI_IO_PROTOCOL_GET_LOCATION)(
  IN EFI_PCI_IO_PROTOCOL          *This,
  OUT UINTN                       *SegmentNumber,   // PCI segment number
  OUT UINTN                       *BusNumber,
  OUT UINTN                       *DeviceNumber,
  OUT UINTN                       *FunctionNumber
  );
```

## 3.3 EFI_BOOT_SERVICES.LocateHandleBuffer()
```c
//
// Returns an array of handles that support the requested protocol in a buffer allocated from pool.
//
typedef
EFI_STATUS
(EFIAPI *EFI_LOCATE_HANDLE_BUFFER) (
  IN EFI_LOCATE_SEARCH_TYPE  SearchType, // Specifies which handle(s) are to be returned.
  IN EFI_GUID  *Protocol OPTIONAL,       // Provides the protocol to search by.
  IN VOID  *SearchKey OPTIONAL,          // Supplies the search key depending on the SearchType.
  IN OUT UINTN  *NoHandles,              // The number of handles returned in Buffer.
  OUT EFI_HANDLE  **Buffer   // A pointer to the buffer contains requested array of handles that support Protocol.
);
```
Note: The `LocateHandleBuffer()` function returns one or more handles that match the 
`SearchType` request. `Buffer` is allocated from pool, and the number of entries in `Buffer` is returned in `NoHandles`.

### 3.3.1 EFI_LOCATE_SEARCH_TYPE
1. `AllHandles`: `Protocol` and `SearchKey` are ignored and the function returns an array of every handle in the system.
2. `ByRegisterNotify`: `Protocol` is ignored for this search type.
3. `ByProtocol`: All handles that support `Protocol` are returned. `SearchKey` is ignored for this search type. 

## 3.4 EFI_BOOT_SERVICES.HandleProtocol()
```c
// 
// Queries a handle to determine if it supports a specified protocol. 
// 
typedef
EFI_STATUS
(EFIAPI *EFI_HANDLE_PROTOCOL) (
  IN EFI_HANDLE Handle,    // The handle being queried.
  IN EFI_GUID  *Protocol,
  OUT VOID  **Interface    // Supplies a pointer to the corresponding Protocol Interface is returned.
);
```
Note: The `HandleProtocol()` function queries `Handle` to determine if it supports `Protocol`. If it does, then on return `Interface` points to a pointer to the corresponding Protocol Interface. `Interface` can then be passed to any protocol service to identify the context of the request. 

## 3.5 EFI_BOOT_SERVICES.InstallProtocolInterface()
```c
//
// Installs a protocol interface on a device handle.
//
typedef
EFI_STATUS
(EFIAPI *EFI_INSTALL_PROTOCOL_INTERFACE) (
  IN OUT EFI_HANDLE      *Handle,      // A pointer to the EFI_HANDLE on which the interface is to be installed.
  IN EFI_GUID            *Protocol,
  IN EFI_INTERFACE_TYPE  InterfaceType, // Indicates whether Interfaceis supplied in native form.
  IN VOID                *Interface     // A pointer to the protocol interface.
);

// Enumeration of EFI Interface Types
typedef enum {
  // Indicates that the supplied protocol interface is supplied in native form.
  EFI_NATIVE_INTERFACE
} EFI_INTERFACE_TYPE;
```
Note: The `InstallProtocolInterface()` function installs a protocol interface (a `GUID/Protocol Interface` structure pair) on a device handle.

## 3.6 EFI_BOOT_SERVICES.SetTimer()
```c
//
// Sets the type of timer and the trigger time for a timer event.
// 
typedef
EFI_STATUS
(EFIAPI *EFI_SET_TIMER) (
  IN EFI_EVENT       Event,      // The timer event that is to be signaled at the specified time. 
  IN EFI_TIMER_DELAY Type,       // The type of time that is specified in TriggerTime.
  IN UINT64          TriggerTime // The number of 100ns units until the timer expires.
);

// EFI_TIMER_DELAY
typedef enum {
  TimerCancel,      // The event’s timer setting is to be cancelled and no timer trigger is to be set. 
  TimerPeriodic,    // The event is to be signaled periodically at TriggerTime intervals from the current time.
  TimerRelative     // The event is to be signaled in TriggerTime 100ns units.
} EFI_TIMER_DELAY;
```

## 3.7 EFI_BOOT_SERVICES.CreateEventEx()
```c
// 
// Creates an event in a group.
// 
typedef
EFI_STATUS
(EFIAPI *EFI_CREATE_EVENT_EX) (
  // The type of event to create and its mode and attributes.
  IN UINT32  Type, 
  // The task priority level of event notifications, if needed.  
  IN EFI_TPL  NotifyTpl,  
  // Pointer to the event’s notification function, if any.
  IN EFI_EVENT_NOTIFY  NotifyFunction OPTIONAL, 
  // Pointer to the notification function’s context; corresponds to parameter Contextin the notification function.
  IN CONST VOID  *NotifyContext OPTIONAL,
  // Pointer to the unique identifier of the group to which this event belongs.
  IN CONST EFI_GUID  *EventGroup OPTIONAL,
  // Pointer to the newly created event if the call succeeds; undefined otherwise.
  OUT EFI_EVENT  *Event
);
```
Note: The `CreateEventExfunction` creates a new event of type `Type` and returns it in the specified 
location indicated by `Event`. The event’s notification function, context and task priority are 
specified by `NotifyFunction`, `NotifyContext`, and `NotifyTpl`, respectively. The event 
will be added to the group of events identified by `EventGroup`.

# 4 EhcDriverBindingStop()
```c
//
// Stop this driver on ControllerHandle. Support stopping any child handles created by this driver.
//
EFI_STATUS
EFIAPI
EhcDriverBindingStop (
  IN EFI_DRIVER_BINDING_PROTOCOL *This,
  IN EFI_HANDLE                  Controller,        // Handle of device to stop driver on.
  IN UINTN                       NumberOfChildren,  // Number of Children in the ChildHandleBuffer.
  IN EFI_HANDLE                  *ChildHandleBuffer // List of handles for the children we need to stop.
  )
{ 
  // Test whether the Controller in argv[] is a valid Usb controller handle or not.
  // In Start(), EFI_USB2_HC_PROTOCOL has been installed on the Controller.
  |-- Status = gBS->OpenProtocol (
                  Controller,                     // DeviceHandle contains protocol `gEfiUsb2HcProtocolGuid`.
                  &gEfiUsb2HcProtocolGuid,
                  (VOID **) &Usb2Hc,
                  This->DriverBindingHandle,      // AgentHandle which actually openes the instance `&Usb2Hc`.
                  Controller,                     // A ControllerHandle requires the protocol instance `&Usb2Hc`.
                  EFI_OPEN_PROTOCOL_GET_PROTOCOL  // Used by a driver to get a protocol interface from a handle.
                  );
  
  // #define EHC_FROM_THIS(a)  CR(a, USB2_HC_DEV, Usb2Hc, USB2_HC_DEV_SIGNATURE)
  |-- Ehc   = EHC_FROM_THIS (Usb2Hc);
      PciIo = Ehc->PciIo;
  
  // Removes a protocol interface (Usb2Hc) from a device handle (Controller) on which it was previously installed.
  |-- Status = gBS->UninstallProtocolInterface (
                  Controller,
                  &gEfiUsb2HcProtocolGuid,
                  Usb2Hc
                  );
  
  // Stop AsyncRequest Polling timer
  |-- gBS->SetTimer (Ehc->PollTimer, TimerCancel, EHC_ASYNC_POLL_INTERVAL);
  // Stop the EHCI driver
  |-- EhcHaltHC (Ehc, EHC_GENERIC_TIMEOUT);
  // If Ehc->PollTimer != NULL and Ehc->ExitBootServiceEvent != NULL
  |-- gBS->CloseEvent (Ehc->PollTimer);
      gBS->CloseEvent (Ehc->ExitBootServiceEvent);
  
  |-- EhcFreeSched (Ehc);                                // Free the schedule data.
      FreeUnicodeStringTable (Ehc->ControllerNameTable); // Free the Unicode strings table in ControllerNameTable
  
  // Disable routing of all ports to EHCI controller, so all ports are routed back to the UHCI or OHCI controller.
  |-- EhcClearOpRegBit (Ehc, EHC_CONFIG_FLAG_OFFSET, CONFIGFLAG_ROUTE_EHC);
  
  // Restore original PCI attributes (OriginalPciAttributes). 
  |-- PciIo->Attributes (PciIo, EfiPciIoAttributeOperationSet, Ehc->OriginalPciAttributes, NULL);
  |-- gBS->CloseProtocol(PciIo)
  |-- FreePool (Ehc);

  return EFI_SUCCESS;
}
```


