# EhcCreateHelpQ

```c
// Path: \MdeModulePkg\Bus\Pci\EhciDxe\EhciSched.c
//
// Create helper QTD/QH for the EHCI device.
// 
EFI_STATUS
EhcCreateHelpQ (
  IN USB2_HC_DEV          *Ehc
  )
{
    |-- USB_ENDPOINT            Ep;
        EHC_QH                  *Qh;
        QH_HW                   *QhHw;
        EHC_QTD                 *Qtd;
        EFI_PHYSICAL_ADDRESS    PciAddr;

    // Create an inactive Qtd to terminate the short packet read.
    |-- Qtd = EhcCreateQtd (Ehc, NULL, NULL, 0, QTD_PID_INPUT, 0, 64);
        // Any time the Halted bit being set to a one, the Active bit (Bit7) is also set to 0.
        Qtd->QtdHw.Status   = QTD_STAT_HALTED;   // Halted bit (Bit6)
        Ehc->ShortReadStop  = Qtd;  // ShortReadStop is used to terminate the short read.

    // Before create a QH to act as the EHC reclamation header, config an endpoint at first.
    |-- Ep.DevAddr    = 0;
        Ep.EpAddr     = 1;
        Ep.Direction  = EfiUsbDataIn;
        Ep.DevSpeed   = EFI_USB_SPEED_HIGH;
        Ep.MaxPacket  = 64;
        Ep.HubAddr    = 0;
        Ep.HubPort    = 0;
        Ep.Toggle     = 0;
        Ep.Type       = EHC_BULK_TRANSFER;
        Ep.PollRate   = 1;
        
        Qh            = EhcCreateQh (Ehc, &Ep);

    // Set the header to loopback to itself.
    |-- PciAddr           = UsbHcGetPciAddressForHostMem (Ehc->MemPool, Qh, sizeof (EHC_QH));
        QhHw              = &Qh->QhHw;
        QhHw->HorizonLink = QH_LINK (PciAddr + OFFSET_OF(EHC_QH, QhHw), EHC_TYPE_QH, FALSE);
        QhHw->Status      = QTD_STAT_HALTED;
        QhHw->ReclaimHead = 1;
        Qh->NextQh        = Qh;
        Ehc->ReclaimHead  = Qh;

    // Create a dummy QH to act as the terminator for periodical schedule
    |-- Ep.EpAddr   = 2;
        Ep.Type     = EHC_INT_TRANSFER_SYNC;

        Qh          = EhcCreateQh (Ehc, &Ep);
        Qh->QhHw.Status = QTD_STAT_HALTED;
        Ehc->PeriodOne  = Qh;

    return EFI_SUCCESS;
}

[Comments]
1. EhcCreateQtd, create an inactive Qtd as ShortReadStop, which is used to terminate the short read.
   In this Qtd, its QtdHw.Status = QTD_STAT_HALTED.
2. Config an endpoint at first, and then create a QH to act as the EHC reclamation header.
   In this Qh, it should set its QhHw->Status[QTD_STAT_HALTED], ReclaimHead = 1, and header to loopback to itself.
3. Create an another QH to act as the terminator for periodical schedule.

[Related]
// Endpoint address and its capabilities
typedef struct _USB_ENDPOINT {
    UINT8                   DevAddr;
    UINT8                   EpAddr;     // Endpoint address, no direction encoded in
    EFI_USB_DATA_DIRECTION  Direction;
    UINT8                   DevSpeed;
    UINTN                   MaxPacket;
    UINT8                   HubAddr;
    UINT8                   HubPort;
    UINT8                   Toggle;     // Data toggle, not used for control transfer
    UINTN                   Type;
    UINTN                   PollRate;   // Polling interval used by EHCI
} USB_ENDPOINT;

// Software QH structure.
// The control/bulk/interrupt transfers all use the QH and queue token strcuture.
struct _EHC_QH {
    QH_HW                   QhHw;
    UINT32                  Signature;
    EHC_QH                  *NextQh;    // The queue head pointed to by horizontal link
    LIST_ENTRY              Qtds;       // The list of QTDs to this queue head
    UINTN                   Interval;
};

// Hardware QH strcture.
typedef struct {
    UINT32                  HorizonLink;

    // Endpoint capabilities/Characteristics DWord 1 and DWord 2
    UINT32                  DeviceAddr   : 7;
    UINT32                  Inactive     : 1;
    UINT32                  EpNum        : 4;
    UINT32                  EpSpeed      : 2;
    UINT32                  DtCtrl       : 1;
    UINT32                  ReclaimHead  : 1;
    UINT32                  MaxPacketLen : 11;
    UINT32                  CtrlEp       : 1;
    UINT32                  NakReload    : 4;

    UINT32                  SMask        : 8;
    UINT32                  CMask        : 8;
    UINT32                  HubAddr      : 7;
    UINT32                  PortNum      : 7;
    UINT32                  Multiplier   : 2;

    // Transaction execution overlay area
    UINT32                  CurQtd;
    UINT32                  NextQtd;
    UINT32                  AltQtd;

    UINT32                  Status       : 8;
    UINT32                  Pid          : 2;
    UINT32                  ErrCnt       : 2;
    UINT32                  CurPage      : 3;
    UINT32                  Ioc          : 1;
    UINT32                  TotalBytes   : 15;
    UINT32                  DataToggle   : 1;

    UINT32                  Page[5];
    UINT32                  PageHigh[5];
} QH_HW;

// Software QTD strcture, this is used to manage all the
// QTD generated from a URB. Don't add fields before QtdHw.
struct _EHC_QTD {
    QTD_HW                  QtdHw;
    UINT32                  Signature;
    LIST_ENTRY              QtdList;   // The list of QTDs to one end point
    UINT8                   *Data;     // Buffer of the original data
    UINTN                   DataLen;   // Original amount of data in this QTD
};

// Hardware QTD strcture
typedef struct {
    UINT32                  NextQtd;
    UINT32                  AltNext;

    UINT32                  Status       : 8;
    UINT32                  Pid          : 2;
    UINT32                  ErrCnt       : 2;
    UINT32                  CurPage      : 3;
    UINT32                  Ioc          : 1;
    UINT32                  TotalBytes   : 15;
    UINT32                  DataToggle   : 1;

    UINT32                  Page[5];
    UINT32                  PageHigh[5];
} QTD_HW;

```

# EhcCreateQtd
```c
// Path: \MdeModulePkg\Bus\Pci\EhciDxe\EhciUrb.c
// 
// Create a single QTD to hold the data.
//
EHC_QTD *
EhcCreateQtd (
  IN USB2_HC_DEV          *Ehc,       // The EHCI device.
  IN UINT8                *Data,      // The cpu memory address of current data not associated with a QTD.
  IN UINT8                *DataPhy,   // The pci bus address of current data not associated with a QTD.
  IN UINTN                DataLen,    // The length of the data.
  IN UINT8                PktId,      // Packet ID to use in the QTD.
  IN UINT8                Toggle,     // Data toggle to use in the QTD.
  IN UINTN                MaxPacket   // Maximu packet length of the endpoint.
  )
{
    |-- EHC_QTD                 *Qtd;     // Software Qtd
        QTD_HW                  *QtdHw;   // Hardware Qtd
        UINTN                   Index;
        UINTN                   Len;
        UINTN                   ThisBufLen;

    // Allocate some memory for a Qtd from the host controller's memory pool 
    // which can be used to communicate with host controller.
    |-- Qtd = UsbHcAllocateMem (Ehc->MemPool, sizeof (EHC_QTD));

    // Config Qtd
    |-- Qtd->Signature    = EHC_QTD_SIG;
        Qtd->Data         = Data;
        Qtd->DataLen      = 0;

    // Initializes the head node of a doubly-linked list, and then return a pointer.
    |-- InitializeListHead (&Qtd->QtdList);

    // Config the HW QtdHw in the SW Qtd.
    |-- QtdHw             = &Qtd->QtdHw;
        QtdHw->NextQtd    = QTD_LINK (NULL, TRUE);  // Next Transfer Element Pointer, invalid default.
        QtdHw->AltNext    = QTD_LINK (NULL, TRUE);  // True means this field is invalid default.
        QtdHw->Status     = QTD_STAT_ACTIVE;        // Default to set Active.
        QtdHw->Pid        = PktId;
        QtdHw->ErrCnt     = QTD_MAX_ERR;
        QtdHw->Ioc        = 0;   // Interrupt On Complete
        QtdHw->TotalBytes = 0;   // Total number of bytes to be moved with this transfer descriptor. 
        QtdHw->DataToggle = Toggle;

    // Fill in the buffer pointors (page0-page4 in QtdHw)
    |-- if (Data != NULL) {
        |-- Len = 0;   // Used to count the Len of Data will be hold.
            for (Index = 0; Index <= QTD_MAX_BUFFER; Index++) {   // QTD_MAX_BUFFER = 4
                // Set the buffer point (Check page 41 EHCI Spec 1.0). No need to
                // compute the offset and clear Reserved fields. This is already
                // done in the data point.
                QtdHw->Page[Index]      = EHC_LOW_32BIT (DataPhy);
                QtdHw->PageHigh[Index]  = EHC_HIGH_32BIT (DataPhy);

                // The size of a Qtd Buffer is 4096-bytes. QTD_BUF_LEN = 4096
                // ThisBufLen = from DataPhy, the available space can be used in this Buffer.
                ThisBufLen = QTD_BUF_LEN - (EHC_LOW_32BIT (DataPhy) & QTD_BUF_MASK);

                // When Len + ThisBufLen >= DataLen, means Buffer space is enough to hold the data.
                if (Len + ThisBufLen >= DataLen) {
                    Len = DataLen;
                    break;
                }

                // Here means the current Buffer is not enough, it should use the next Buffer.
                Len += ThisBufLen;       // Current available buffer space.
                Data += ThisBufLen;      // Cpu mem addr of current data, means has stored ThisBufLen size data.
                DataPhy += ThisBufLen;   // Pci bus addr of current data.
            }

        // Need to fix the last pointer if the Qtd can't hold all the user's data to make sure that 
        // the length is in the unit of max packets. If it can hold all the data, there is no such need.
        |-- if (Len < DataLen) {    // Here means the len of all four Buffers cannot hold all the data.
                Len = Len - Len % MaxPacket;    // In this time, make sure the MaxPacket data to hold.
            }

        |-- QtdHw->TotalBytes = (UINT32) Len;
            Qtd->DataLen      = Len;
        }

    return Qtd;
}
```
```
[Comments]
1. Allocate some memory for a Qtd.
2. Config Qtd, init Qtd->QtdList, and config the HW QtdHw in the SW Qtd.
3. Config Page 0 - Page 4 in QtdHw.
```
# EhcCreateQh
```c
// Path: \MdeModulePkg\Bus\Pci\EhciDxe\EhciUrb.c
//
// Allocate and initialize a EHCI queue head.
//
EHC_QH *
EhcCreateQh (
  IN USB2_HC_DEV          *Ehci,   // The EHCI device.
  IN USB_ENDPOINT         *Ep      // The endpoint to create queue head for.
  )
{
    |-- EHC_QH                  *Qh;   // SW Qh
        QH_HW                   *QhHw; // HW Qh

    // Allocate memory for SW QH from the host controller's memory pool
    |-- Qh = UsbHcAllocateMem (Ehci->MemPool, sizeof (EHC_QH));

    // Config Qh
    |-- Qh->Signature       = EHC_QH_SIG;
        Qh->NextQh          = NULL;
        Qh->Interval        = Ep->PollRate;

        InitializeListHead (&Qh->Qtds);

        QhHw                = &Qh->QhHw;
        QhHw->HorizonLink   = QH_LINK (NULL, 0, TRUE);
        QhHw->DeviceAddr    = Ep->DevAddr;
        QhHw->Inactive      = 0;
        QhHw->EpNum         = Ep->EpAddr;
        QhHw->EpSpeed       = Ep->DevSpeed;
        QhHw->DtCtrl        = 0;
        QhHw->ReclaimHead   = 0;
        QhHw->MaxPacketLen  = (UINT32) Ep->MaxPacket;
        QhHw->CtrlEp        = 0;
        QhHw->NakReload     = QH_NAK_RELOAD;
        QhHw->HubAddr       = Ep->HubAddr;
        QhHw->PortNum       = Ep->HubPort;
        QhHw->Multiplier    = 1;
        QhHw->DataToggle    = Ep->Toggle;
        
        if (Ep->DevSpeed != EFI_USB_SPEED_HIGH) 
            QhHw->Status |= QTD_STAT_DO_SS;  // Status[0] = 0 ???

    // Special init for different transfer type
    |-- switch (Ep->Type)
        |-- case EHC_CTRL_TRANSFER:  // Control transfer
            // Set 1: means control transfer initialize data toggle from each QTD.
            |-- QhHw->DtCtrl = 1;
            // Low/Full speed endpoint and Control transfer -> Set Control Endpoint Flag (C) = 1.
            |-- if (Ep->DevSpeed != EFI_USB_SPEED_HIGH) -> QhHw->CtrlEp = 1;
        
        |-- case EHC_INT_TRANSFER_ASYNC:  // Interrupt transfer
            case EHC_INT_TRANSFER_SYNC:
            // Software should not use this feature for interrupt queue heads.
            |-- QhHw->NakReload = 0;  
                EhcInitIntQh (Ep, QhHw);  // Set the S-Mask and C-Mask
        
        |-- case EHC_BULK_TRANSFER:   // Bulk transfer
            // When High speed and Out, the status[0] bit for the Ping protocol.
            |-- if Ep->DevSpeed == High_Speed and Ep->Direction == Out -> QhHw->Status |= QTD_STAT_DO_PING;
    
    return Qh;            
}
```

