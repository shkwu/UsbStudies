# EhcReset
```c
//
// Provides software reset for the USB host controller.
//
EFI_STATUS
EFIAPI
EhcReset (
  IN EFI_USB2_HC_PROTOCOL *This,
  IN UINT16               Attributes  // A bit mask of the reset operation to perform.
  )
{
    |-- Ehc = EHC_FROM_THIS (This);
    // Raise TPL to EHC_TPL, and return its previous TPL.
    |-- OldTpl  = gBS->RaiseTPL (EHC_TPL);
    |-- switch (Attributes)
        |-- case EFI_USB_HC_RESET_GLOBAL:  // 0x0001, same behavior as Host Controller Reset
        |-- case EFI_USB_HC_RESET_HOST_CONTROLLER: // 0x0002, Host Controller must be Halt when Reset it.
            |-- if (Ehc->DebugPortNum != 0)
            // If DebugPortNum != 0, and DbgCtrlStatus are IN_USE and OWNER.
                |-- DbgCtrlStatus = EhcReadDbgRegister(Ehc, 0);
                    if ((DbgCtrlStatus & (USB_DEBUG_PORT_IN_USE | USB_DEBUG_PORT_OWNER)) == 
                        (USB_DEBUG_PORT_IN_USE | USB_DEBUG_PORT_OWNER))
                        return EFI_SUCCESS;
            // Else DebugPortNum == 0, and EHC is running, then halt it.
            |-- if (!EhcIsHalt (Ehc))   // Check whether EHC_USBSTS[USBSTS_HALT] = 1
                // Clear EHC_USBCMD[USBCMD_RUN] to 0, and wait EHC_USBSTS[USBSTS_HALT] = 1.
                |-- Status = EhcHaltHC (Ehc, EHC_GENERIC_TIMEOUT); 
            
            // Remove all the asynchronous interrutp transfers.
            |-- EhciDelAllAsyncIntTransfers (Ehc); // Cannot understand well in detail.
            // Clear all the interrutp status bits in EHC_USBSTS.
            |-- EhcAckAllInterrupt (Ehc); // EhcWriteOpReg (Ehc, EHC_USBSTS_OFFSET, 0x003F);
            // Free the schedule data.
            |-- EhcFreeSched (Ehc);

            // Reset the host controller.
            // Host can only be reset when it is halt.
            |-- Status = EhcResetHC (Ehc, EHC_RESET_TIMEOUT);

            // Initialize the HC hardware.
            |-- Status = EhcInitHC (Ehc);
        |-- case EFI_USB_HC_RESET_GLOBAL_WITH_DEBUG:  // 0x0004
        |-- case EFI_USB_HC_RESET_HOST_WITH_DEBUG:    // 0x0008
            |-- Status = EFI_UNSUPPORTED;   // Not support debug level reset.
        |-- default: Status = EFI_INVALID_PARAMETER;
    
    |-- ON_EXIT:
        DEBUG ((EFI_D_INFO, "EhcReset: exit status %r\n", Status));
        gBS->RestoreTPL (OldTpl);
        return Status;
}

[Comments]
EhcReset focus on `EFI_USB_HC_RESET_HOST_CONTROLLER`:
1. Check whether exists debug ports or not. 
   If exist, and both DbgCtrlStatus[OWNER] =1 and DbgCtrlStatus[IN_USE] = 1, goto  ON_EXIT.
2. Check whether EHC_USBSTS[USBSTS_HALT] = 1. 
   If false, halt Ehc by clearing EHC_USBCMD[USBCMD_RUN] and wait EHC_USBSTS[USBSTS_HALT] = 1.
3. Clean up the asynchronous transfers.
4. Reset the host controller by setting EHC_USBCMD[USBCMD_RESET] = 1.
5. Initialize the EHC.
```

## EhcHaltHC
```c
//
// Halt the host controller.
//
EFI_STATUS
EhcHaltHC (
  IN USB2_HC_DEV         *Ehc,
  IN UINT32              Timeout
  )
{
    // Clear USBCMD_RUN bit in EHC_USBCMD register.
    EhcClearOpRegBit (Ehc, EHC_USBCMD_OFFSET, USBCMD_RUN);

    // Atfer chear EHC_USBCMD[USBCMD_RUN], wait EHC_USBSTS[USBSTS_HALT] = 1.
    Status = EhcWaitOpRegBit (Ehc, EHC_USBSTS_OFFSET, USBSTS_HALT, TRUE, Timeout);
    return Status;
}

[Comments]
1. Only EHC_USBCMD[USBCMD_RUN] = 0 and EHC_USBSTS[USBSTS_HALT] = 1, the EHC is halted.
2. Cannot directly write 1 to EHC_USBSTS[USBSTS_HALT].
```

## EhciDelAllAsyncIntTransfers
```c
//
// Remove all the asynchronous interrutp transfers.
//
VOID
EhciDelAllAsyncIntTransfers (
  IN USB2_HC_DEV          *Ehc
  )
{
    LIST_ENTRY              *Entry;
    LIST_ENTRY              *Next;
    URB                     *Urb;

    // for loop, when Entry == &Ehc->AsyncIntTransfers, end loop.
    EFI_LIST_FOR_EACH_SAFE (Entry, Next, &Ehc->AsyncIntTransfers) {
        Urb = EFI_LIST_CONTAINER (Entry, URB, UrbList); // BASE_CR

        // Unlink an interrupt queue head from the periodic schedule frame list.
        EhcUnlinkQhFromPeriod (Ehc, Urb->Qh);    // cannot understand in detail
        // Removes a node from a doubly-linked list.
        RemoveEntryList (&Urb->UrbList);

        gBS->FreePool (Urb->Data);
        EhcFreeUrb (Ehc, Urb);  // Free an allocated URB.
  }
}

[Comments]
1. Cannot understand well in detal.

```
### EFI_LIST_FOR_EACH_SAFE
```c
#define EFI_LIST_FOR_EACH_SAFE(Entry, NextEntry, ListHead)            \
  for(Entry = (ListHead)->ForwardLink, NextEntry = Entry->ForwardLink; \
      Entry != (ListHead);  \
      Entry = NextEntry, NextEntry = Entry->ForwardLink)
```

## EhcFreeSched
```c
// 
// Free the schedule data. It may be partially initialized.
//
VOID
EhcFreeSched (
  IN USB2_HC_DEV          *Ehc
  )
{
    // EHC_FRAME_BASE register and EHC_ASYNC_HEAD register write to 0x00.
    EhcWriteOpReg (Ehc, EHC_FRAME_BASE_OFFSET, 0); // 0x14, Frame list base address offset
    EhcWriteOpReg (Ehc, EHC_ASYNC_HEAD_OFFSET, 0); // 0x18, Next asynchronous list address offset
    
    // Check if Ehc->PeriodOne, ReclaimHead, ShortReadStop, MemPool, PeriodFrame, PeriodFrameHost != NULL,
    UsbHcFreeMem() or FreePool()
} 

```

## EhcResetHC
```c
//
// Reset the host controller.
//
EFI_STATUS
EhcResetHC (
  IN USB2_HC_DEV          *Ehc,
  IN UINT32               Timeout
  )
{
    // Host can only be reset when it is halt. If not so, halt it
    if (!EHC_REG_BIT_IS_SET (Ehc, EHC_USBSTS_OFFSET, USBSTS_HALT))  // Check EHC_USBSTS[USBSTS_HALT]
        // Clear EHC_USBCMD[USBCMD_RUN] to 0, and wait EHC_USBSTS[USBSTS_HALT] = 1.
        Status = EhcHaltHC (Ehc, Timeout); 

    EhcSetOpRegBit (Ehc, EHC_USBCMD_OFFSET, USBCMD_RESET);  // Write 1 to EHC_USBCMD[USBCMD_RESET]
    // After EHC_USBCMD[USBCMD_RESET] = 1, wait EHC_USBCMD[USBCMD_RESET] = 0.
    Status = EhcWaitOpRegBit (Ehc, EHC_USBCMD_OFFSET, USBCMD_RESET, FALSE, Timeout);
    return Status;
}

[Comments]
1. Host can only be reset when it is halt. If not so, halt it.
2. Halt host by EHC_USBCMD[USBCMD_RUN] = 0, then wait EHC_USBSTS[USBSTS_HALT] = 1.
3. When host is halt, reset host by write 1 to EHC_USBCMD[USBCMD_RESET]. 
4. After setting EHC_USBCMD[USBCMD_RESET] = 1, it should wait EHC_USBCMD[USBCMD_RESET] = 0 to 
   indicate that the reset operation has completed.

```