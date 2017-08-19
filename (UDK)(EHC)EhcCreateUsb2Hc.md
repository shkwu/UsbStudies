# EhcCreateUsb2Hc
```c
//
// Create and initialize a USB2_HC_DEV.
//
USB2_HC_DEV *
EhcCreateUsb2Hc (
  IN EFI_PCI_IO_PROTOCOL       *PciIo,                // The PciIo on this device.
  IN EFI_DEVICE_PATH_PROTOCOL  *DevicePath,           // The device path of host controller.
  IN UINT64                    OriginalPciAttributes  // Original PCI attributes.
  )
{   
    // USB2_HC_DEV  *Ehc;
    |-- Ehc = AllocateZeroPool (sizeof (USB2_HC_DEV));  // Allocate memory for Ehc.
    
    // Init EFI_USB2_HC_PROTOCOL interface and private data structure
    |-- Ehc->Signature                        = USB2_HC_DEV_SIGNATURE;
        
        // Get the controller supported `MaxSpeed`, `PortNumber` and `Is64BitCapable`.
        Ehc->Usb2Hc.GetCapability             = EhcGetCapability; 
        Ehc->Usb2Hc.Reset                     = EhcReset;
        Ehc->Usb2Hc.GetState                  = EhcGetState;
        Ehc->Usb2Hc.SetState                  = EhcSetState;
        Ehc->Usb2Hc.ControlTransfer           = EhcControlTransfer;
        Ehc->Usb2Hc.BulkTransfer              = EhcBulkTransfer;
        Ehc->Usb2Hc.AsyncInterruptTransfer    = EhcAsyncInterruptTransfer;
        Ehc->Usb2Hc.SyncInterruptTransfer     = EhcSyncInterruptTransfer;
        Ehc->Usb2Hc.IsochronousTransfer       = EhcIsochronousTransfer;
        Ehc->Usb2Hc.AsyncIsochronousTransfer  = EhcAsyncIsochronousTransfer;
        Ehc->Usb2Hc.GetRootHubPortStatus      = EhcGetRootHubPortStatus;
        Ehc->Usb2Hc.SetRootHubPortFeature     = EhcSetRootHubPortFeature;
        Ehc->Usb2Hc.ClearRootHubPortFeature   = EhcClearRootHubPortFeature;
        Ehc->Usb2Hc.MajorRevision             = 0x2;
        Ehc->Usb2Hc.MinorRevision             = 0x0;

        Ehc->PciIo                 = PciIo;
        Ehc->DevicePath            = DevicePath;
        Ehc->OriginalPciAttributes = OriginalPciAttributes;  // Used in EhcDriverBindingStop()

        // Initializes the head node of a doubly-linked list for AsyncIntTransfers.
        InitializeListHead (&Ehc->AsyncIntTransfers);

        Ehc->HcStructParams = EhcReadCapRegister (Ehc, EHC_HCSPARAMS_OFFSET);
        Ehc->HcCapParams    = EhcReadCapRegister (Ehc, EHC_HCCPARAMS_OFFSET);
        // #define EHC_CAPLENGTH_OFFSET    0
        Ehc->CapLen         = EhcReadCapRegister (Ehc, EHC_CAPLENGTH_OFFSET) & 0x0FF;
    
    |-- if (Ehc->CapLen == 0) -> gBS->FreePool (Ehc); -> return NULL;
    |-- EhcGetUsbDebugPortInfo (Ehc);   // Get the usb debug port related information.
    
    // Create an event (Ehc->PollTimer) for AsyncRequest Polling Timer
    |-- Status = gBS->CreateEvent (
                  EVT_TIMER | EVT_NOTIFY_SIGNAL,   // The type of event to create.
                  TPL_NOTIFY,                      // The task priority level of event notifications.
                  EhcMonitorAsyncRequests,         // Pointer to the event’s notification function.
                  Ehc,                             // Pointer to the notification function’s context.
                  &Ehc->PollTimer                  // Pointer to the newly created event.
                  );
    // If create event success,
    return Ehc;
}

[Comments]
1. Allocate memory pool with sizeof (USB2_HC_DEV).
2. Init EFI_USB2_HC_PROTOCOL interface and private data structure.
3. Check Ehc->CapLen, if Ehc->CapLen == 0, free Ehc and return NULL.
4. Get the usb debug port related information.
5. Create Ehc->PollTimer event for AsyncRequest Polling Timer.
6. Return Ehc.
```

## USB2_HC_DEV Structure
```c
struct _USB2_HC_DEV {
    UINTN                     Signature;
    EFI_USB2_HC_PROTOCOL      Usb2Hc;

    EFI_PCI_IO_PROTOCOL       *PciIo;
    EFI_DEVICE_PATH_PROTOCOL  *DevicePath;
    UINT64                    OriginalPciAttributes;
    USBHC_MEM_POOL            *MemPool;

    // Schedule data shared between asynchronous and periodic transfers:
    // ShortReadStop, as its name indicates, is used to terminate the short read except the control transfer. 
    // EHCI follows the alternative next QTD point when a short read happens.
    // For control transfer, even the short read happens, try the status stage.
    EHC_QTD                  *ShortReadStop;
    EFI_EVENT                 PollTimer;

    // ExitBootServicesEvent is used to stop the EHC DMA operation after exit boot service.
    EFI_EVENT                 ExitBootServiceEvent;

    // Asynchronous(bulk and control) transfer schedule data:
    // ReclaimHead is used as the head of the asynchronous transfer list. It acts as the reclamation header.
    EHC_QH                   *ReclaimHead;

    // Periodic (interrupt) transfer schedule data:
    // The buffer pointed by this pointer is used to store pci bus address of the QH descriptor.
    VOID                      *PeriodFrame;  
    // The buffer pointed by this pointer is used to store host memory address of the QH descriptor.   
    VOID                      *PeriodFrameHost; 
    VOID                      *PeriodFrameMap;

    EHC_QH                    *PeriodOne;
    LIST_ENTRY                AsyncIntTransfers;

    // EHCI configuration data
    UINT32                    HcStructParams; // Cache of HC structure parameter, EHC_HCSPARAMS_OFFSET
    UINT32                    HcCapParams;    // Cache of HC capability parameter, HCCPARAMS
    UINT32                    CapLen;         // Capability length

    // Misc
    EFI_UNICODE_STRING_TABLE  *ControllerNameTable;

    // EHCI debug port info
    UINT16                    DebugPortOffset; // The offset of debug port mmio register
    UINT8                     DebugPortBarNum; // The bar number of debug port mmio register
    UINT8                     DebugPortNum;    // The port number of usb debug port

    BOOLEAN                   Support64BitDma; // Whether 64 bit DMA may be used with this device
};
```

## EFI_USB2_HC_PROTOCOL Structure
```c
struct _EFI_USB2_HC_PROTOCOL {
    EFI_USB2_HC_PROTOCOL_GET_CAPABILITY              GetCapability;
    EFI_USB2_HC_PROTOCOL_RESET                       Reset;
    EFI_USB2_HC_PROTOCOL_GET_STATE                   GetState;
    EFI_USB2_HC_PROTOCOL_SET_STATE                   SetState;
    EFI_USB2_HC_PROTOCOL_CONTROL_TRANSFER            ControlTransfer;
    EFI_USB2_HC_PROTOCOL_BULK_TRANSFER               BulkTransfer;
    EFI_USB2_HC_PROTOCOL_ASYNC_INTERRUPT_TRANSFER    AsyncInterruptTransfer;
    EFI_USB2_HC_PROTOCOL_SYNC_INTERRUPT_TRANSFER     SyncInterruptTransfer;
    EFI_USB2_HC_PROTOCOL_ISOCHRONOUS_TRANSFER        IsochronousTransfer;
    EFI_USB2_HC_PROTOCOL_ASYNC_ISOCHRONOUS_TRANSFER  AsyncIsochronousTransfer;
    EFI_USB2_HC_PROTOCOL_GET_ROOTHUB_PORT_STATUS     GetRootHubPortStatus;
    EFI_USB2_HC_PROTOCOL_SET_ROOTHUB_PORT_FEATURE    SetRootHubPortFeature;
    EFI_USB2_HC_PROTOCOL_CLEAR_ROOTHUB_PORT_FEATURE  ClearRootHubPortFeature;
    
    // The major revision number of the USB host controller. The revision information 
    // indicates the release of the Universal Serial Bus Specification with which the 
    // host controller is compliant.
    UINT16                                           MajorRevision;

    // The minor revision number of the USB host controller. The revision information 
    // indicates the release of the Universal Serial Bus Specification with which the 
    // host controller is compliant.  
    UINT16                                           MinorRevision;
};
```
## EhcGetUsbDebugPortInfo(Ehc)
```c
//
// Get the usb debug port related information.
// 
EFI_STATUS
EhcGetUsbDebugPortInfo (
  IN  USB2_HC_DEV     *Ehc
 )
{
    |-- PciIo = Ehc->PciIo;

    // Detect if the EHCI host controller support Capaility Pointer.
    |-- Status = PciIo->Pci.Read(PCI_PRIMARY_STATUS_OFFSET, &PciStatus) // PCI_PRIMARY_STATUS_OFFSET = 0x06
        // The Pci Device Doesn't Support Capability Pointer.
        // Bit 3 -> Interrupt Status; Bit 4 -> Capability List, when this bit = 1, Capability Pointer is valid.
        if ((PciStatus & EFI_PCI_STATUS_CAPABILITY) == 0) return UNSUPPORTED; // EFI_PCI_STATUS_CAPABILITY = BIT4
    |-- Status = PciIo->Pci.Read (PCI_CAPBILITY_POINTER_OFFSET, &CapabilityPtr) // 0x34, Read Capability Pointer

    // Find Capability ID 0xA, Which Is For Debug Port
    |-- while (CapabilityPtr != 0)  // Traverse Capability List to find Debug Port Capability, then break while.
        |-- Status = PciIo->Pci.Read (CapabilityPtr, &CapabilityId)
        |-- if (CapabilityId == EHC_DEBUG_PORT_CAP_ID) -> break  // EHC_DEBUG_PORT_CAP_ID = 0xA
        |-- Status = PciIo->Pci.Read (CapabilityPtr + 1, &CapabilityPtr) // Read next Capability Pointer
    |-- if (CapabilityPtr == 0) -> return EFI_UNSUPPORTED; // No Debug Port Capability in Capability List.
    // else: Get The Base Address Of Debug Port Register In Debug Port Capability Register. 
    |-- Status = PciIo->Pci.Read (CapabilityPtr + 2, &DebugPort)
        Ehc->DebugPortOffset = DebugPort & 0x1FFF;
        Ehc->DebugPortBarNum = (UINT8)((DebugPort >> 13) - 1);
        Ehc->DebugPortNum    = (UINT8)((Ehc->HcStructParams & 0x00F00000) >> 20);
    
    return EFI_SUCCESS;
}

[Register]
31    29              16          8          0
 |BAR# |Offset         |NXT_PTR   | CAP_ID   |
BAR#: A 3-bit field, which indicates which one of the possible 6 Base
      Address Register offsets contains the Debug Port registers.
Offset: OFFSET of the Debug Port Capability register is used as 
        an offset into the address space assigned by the BAR.
NXT_PTR: Pointer to the next item in the capabilities list. Must be NULL for
         the final item in the list.
CAP_ID: The value of 0Ah in this field identifies that the function supports a Debug Port.

[Comment]
1. PciIo = Ehc->PciIo
2. Read PciStatus from PCI CFG Sapce, then check if PciStatus[Capability_List] = 0x1 or not.
   If ture, support Capability_List, Capability Pointer register (0x34) in PCI.CFG is valid, goto step 3; 
   if false, no Capability_List exists, return EFI_UNSUPPORTED.
3. Read PCI_CAPBILITY_POINTER -> CapabilityPtr.
4. Traverse Capability List to find Debug Port Capability (ID = 0xA), then break while:s
   PciIo->Pci.Read (CapabilityPtr, &CapabilityId) -> check if (CapabilityId == EHC_DEBUG_PORT_CAP_ID)
   If true, break while and goto step 5; else go on traversal by reading next CapabilityPtr.
5. Read |BAR# |Offset         |, then update Ehc debug port related information.
```
