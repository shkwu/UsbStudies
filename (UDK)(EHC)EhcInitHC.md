# EhcInitHC
```c
//
// Initialize the HC hardware.
//   EHCI spec lists the five things to do to initialize the hardware:
//   1. Program CTRLDSSEGMENT  --->  This operation does in EhcInitSched().
//   2. Set USBINTR to enable interrupts
//   3. Set periodic list base
//   4. Set USBCMD, interrupt threshold, frame list size etc
//   5. Write 1 to CONFIGFLAG to route all ports to EHCI
//
EFI_STATUS
EhcInitHC (
  IN USB2_HC_DEV          *Ehc   // The device to init.
  )
{
    //
    // Allocate the periodic frame and associated memeory
    // management facilities if not already done.
    //
    |-- if (Ehc->PeriodFrame != NULL) {
            EhcFreeSched (Ehc); // Write both EHC_FRAME_BASE and EHC_ASYNC_HEAD registers to 0x00.
        }
        Status = EhcInitSched (Ehc);  // Initialize the schedule data structure. 

    // 1. Clear USBINTR to disable all the interrupt. UEFI works by polling.
    |-- EhcWriteOpReg (Ehc, EHC_USBINTR_OFFSET, 0);   // 0x08, USB interrutp offset

    // 2. Start the Host Controller
    |-- EhcSetOpRegBit (Ehc, EHC_USBCMD_OFFSET, USBCMD_RUN); // Set EHC_USBCMD[USBCMD_RUN] = 1

    // 3. Power up all ports if EHCI has Port Power Control (PPC) support
    |-- if (Ehc->HcStructParams & HCSP_PPC)    // EHCI supports Port Power Control (PPC)
            for (Index = 0; Index < (UINT8) (Ehc->HcStructParams & HCSP_NPORTS); Index++)
                // For each port, set EHC_PORT_STAT[PORTSC_POWER] = 1
                EhcWriteOpReg (Ehc, (UINT32) (EHC_PORT_STAT_OFFSET + (4 * Index)), RegVal);

    // 4. Set all ports routing to EHC, set EHC_CONFIG_FLAG[CF] = 1.
    |-- EhcSetOpRegBit (Ehc, EHC_CONFIG_FLAG_OFFSET, CONFIGFLAG_ROUTE_EHC);

    // 5. Enable the periodic schedule and asynchrounous schedule, then wait Status bit changed.
    |-- Status = EhcEnablePeriodSchd (Ehc, EHC_GENERIC_TIMEOUT);
    |-- Status = EhcEnableAsyncSchd (Ehc, EHC_GENERIC_TIMEOUT);

    return EFI_SUCCESS;
}
```
```
[Notes] 
1. Port Power Control (PPC): Bit 4 in HCSPARAMS register. 
   This field indicates whether the host controller implementation includes port power control. 
   A one in this bit indicates the ports have port power switches. 
   A zero in this bit indicates the port do not have port power switches. 
   The value of this field affects the functionality of the Port Power field in each PORTSC register.

2. After enabling the periodic schedule and asynchrounous schedule, it should wait the status bit changed.

[Comments]
1. Initialize the schedule data structure --> EhcInitSched().
2. Clear USBINTR to disable all the interrupt by writting 0x00 to USB interrutp register.
3. Check if Port Power Control (PPC) support or not, if ture, set EHC_PORT_STAT[PORTSC_POWER] = 1.
4. Set EHC_CONFIG_FLAG[CF] = 1 to route all ports to EHC.
5. In EHC_USBCMD register, enable the periodic schedule and asynchrounous schedule.
```
