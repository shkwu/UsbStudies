# EFI_USB2_HC_PROTOCOL Structure
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
    
    // Init EFI_USB2_HC_PROTOCOL interface in EhcCreateUsb2Hc()
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
```

## EhcGetCapability
```c
//
// Retrieves the capability of root hub ports.
// Contains MaxSpeed, PortNumber and Is64BitCapable.
//
EFI_STATUS
EFIAPI
EhcGetCapability ( 
  IN  EFI_USB2_HC_PROTOCOL  *This,           // This EFI_USB_HC_PROTOCOL instance.
  OUT UINT8                 *MaxSpeed,       // Max speed supported by the controller.
  OUT UINT8                 *PortNumber,     // Number of the root hub ports.
  OUT UINT8                 *Is64BitCapable  // Whether the controller supports 64-bit memory addressing.
  )
{
    USB2_HC_DEV             *Ehc;
    EFI_TPL                 OldTpl;   // Task priority level
    
    // Check if parameters valid or not.
    if ((MaxSpeed == NULL) || (PortNumber == NULL) || (Is64BitCapable == NULL)) {
        return EFI_INVALID_PARAMETER;
    }
    
    // Raises a  new task's priority level and returns its previous level.
    // The EHC_TPL is a new task priority level. What is the previous level???
    OldTpl          = gBS->RaiseTPL (EHC_TPL);  // EHC_TPL = TPL_NOTIFY = 16
    Ehc             = EHC_FROM_THIS (This);

    *MaxSpeed       = EFI_USB_SPEED_HIGH;
    *PortNumber     = (UINT8) (Ehc->HcStructParams & HCSP_NPORTS);
    *Is64BitCapable = (UINT8) Ehc->Support64BitDma;

    DEBUG ((EFI_D_INFO, "EhcGetCapability: %d ports, 64 bit %d\n", *PortNumber, *Is64BitCapable));
    
    // Restores a task's priority level to its previous value.
    gBS->RestoreTPL (OldTpl);  // Why??? Where to restore???
    return EFI_SUCCESS;
}
[Comments]
1. About TPL, 执行某操作之前，想提高该operation的优先级，就先OldTpl = gBS->RaiseTPL(NewTpl)，
   等operation执行结束后，再通过gBS->RestoreTPL (OldTpl)退回到原来的优先级上。是否应该这样来理解？？？

[Define]
                              CR(Record,  TYPE,         Field,   TestSignature)
1. #define EHC_FROM_THIS(a)   CR(a,       USB2_HC_DEV,  Usb2Hc,  USB2_HC_DEV_SIGNATURE)
```

