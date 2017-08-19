# EhcInitSched
```c
// 
// Initialize the schedule data structure such as frame list.
// 
EFI_STATUS
EhcInitSched (
  IN USB2_HC_DEV          *Ehc
  )
{

    // First initialize the periodical schedule data:
    // 1. Allocate and map the memory for the frame list
    |-- Status = PciIo->AllocateBuffer (
            PciIo,
            AllocateAnyPages,
            EfiBootServicesData,  // The type of memory to allocate.
            Pages,                // The number of pages to allocate.
            &Buf,                 // Return a pointer to the allocated mem.
            0                     // Bit mask of attributes for allocated mem.
            );
        Status = PciIo->Map (
            PciIo,
            // Indicates if the bus master is going to read or write to sys mem.
            EfiPciIoOperationBusMasterCommonBuffer,  
            Buf,        // The sys mem address to map to the PCI controller.
            &Bytes,     
            // The resulting map address for the bus master PCI controller to use to access the hosts sys mem.
            &PhyAddr,    
            &Map        // A resulting value to pass to Unmap().
            );

    // PeriodFrame: the buffer pointed by this pointer is used to store pci bus address of the QH descriptor.
    |-- Ehc->PeriodFrame      = Buf;
        Ehc->PeriodFrameMap   = Map;

    // Program FRAMELISTBASE register with the low 32 bit addr (use mapping addr)
    // Program CTRLDSSEGMENT register with the high 32 bit addr (support 64-bit DMA)
    |-- EhcWriteOpReg (Ehc, EHC_FRAME_BASE_OFFSET, EHC_LOW_32BIT (PhyAddr));
        EhcWriteOpReg (Ehc, EHC_CTRLDSSEG_OFFSET, EHC_HIGH_32BIT (PhyAddr));

    // Init memory management pool for the host controller.
    |-- Ehc->MemPool = UsbHcInitMemPool (   // Host memory pool
                   PciIo,
                   Ehc->Support64BitDma,     // True or False for supporting 64bit DMA
                   EHC_HIGH_32BIT (PhyAddr)  // The 4G memory area.
                   );

    // 2. Create the help QTD/QH for the EHCI device.
    |-- Status = EhcCreateHelpQ (Ehc);

    // 3. Initialize the frame list entries then set the registers.
    |-- Ehc->PeriodFrameHost = AllocateZeroPool(EHC_FRAME_LEN * sizeof (UINTN));
        // Calculate the corresponding pci bus address according to the Mem parameter.
        PciAddr  = UsbHcGetPciAddressForHostMem (Ehc->MemPool, Ehc->PeriodOne, sizeof (EHC_QH));
        for (Index = 0; Index < EHC_FRAME_LEN; Index++)
            // Store the pci bus address of the QH in period frame list, store the same addr???
            ((UINT32 *)(Ehc->PeriodFrame))[Index] = QH_LINK (PciAddr, EHC_TYPE_QH, FALSE);
            // Store the host address of the QH in period frame list, the same addr???
            ((UINTN *)(Ehc->PeriodFrameHost))[Index] = (UINTN)Ehc->PeriodOne;

    // Second initialize the asynchronous schedule:
    // Only need to set the AsynListAddr register to the reclamation header.
    |-- PciAddr = UsbHcGetPciAddressForHostMem (Ehc->MemPool, Ehc->ReclaimHead, sizeof (EHC_QH));
        EhcWriteOpReg (Ehc, EHC_ASYNC_HEAD_OFFSET, EHC_LOW_32BIT (PciAddr));
    
    return EFI_SUCCESS;
}

```
```
[Notes]
1. PciIo->Map ??? 
2. Ehc->Support64BitDma sets in EhcDriverBindingStart.
3. Host Mem Addr and Pci Bus Addr ???
4. #define QH_LINK(Addr, Type, Term) \
          ((UINT32) ((EHC_LOW_32BIT (Addr) & 0xFFFFFFE0) | (Type) | ((Term) ? 1 : 0)))
5. AllocateZeroPool, this function is only used to allocated host memory space from Ehc->MemPool.
6. UsbHcGetPciAddressForHostMem, this function is used to caculate the real physcal addr for Ehc
   from Ehc->MemPool and a host memory address.

[Comments]
1. First initialize the periodical schedule data.
  + Allocate and map the memory for the frame list, then set Ehc->PeriodFrame and Ehc->PeriodFrameMap.
  + Set the FRAMELISTBASE register with low-32-bits mapped address.
  + Set CTRLDSSEGMENT register with the high 32 bit addr.
  + Init Ehc->MemPool for the host controller.
  + Create the help QTD/QH for the EHCI device.
  + Initialize the frame list entries. 
  + When set the frame list register ???
2. Second initialize the asynchronous schedule
  + Set the AsynListAddr register to the reclamation header.
```

