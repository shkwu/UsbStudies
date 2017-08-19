# EFI_PCI_IO_PROTOCOL

```c
//
// The EFI_PCI_IO_PROTOCOL provides the basic Memory, I/O, PCI configuration, 
// and DMA interfaces used to abstract accesses to PCI controllers. 
// There is one EFI_PCI_IO_PROTOCOL instance for each PCI controller on a PCI bus. 
// A device driver that wishes to manage a PCI controller in a system will have to 
// retrieve the EFI_PCI_IO_PROTOCOL instance that is associated with the PCI controller. 
//
struct _EFI_PCI_IO_PROTOCOL {
  // Polls an address in PCI memory space until an exit condition is met, or a timeout occurs.
  EFI_PCI_IO_PROTOCOL_POLL_IO_MEM         PollMem;

  // Polls an address in PCI I/O space until an exit condition is met, or a timeout occurs.
  EFI_PCI_IO_PROTOCOL_POLL_IO_MEM         PollIo;

  // Allows BAR relative reads or writes to PCI memory space.
  EFI_PCI_IO_PROTOCOL_ACCESS              Mem;   // Mem.Read() or Mem.Write()

  // Allows BAR relative reads or writes to PCI I/O space.
  EFI_PCI_IO_PROTOCOL_ACCESS              Io;    // Io.Read() or Io.Write()

  // Allows PCI controller relative reads or writes to PCI configuration space.
  EFI_PCI_IO_PROTOCOL_CONFIG_ACCESS       Pci;   // Pci.Read() or Pci.Write()

  // Allows one region of PCI memory space to be copied to another region of PCI memory space.
  EFI_PCI_IO_PROTOCOL_COPY_MEM            CopyMem; 

  // Provides the PCI controller–specific address needed to access system memory for DMA.
  EFI_PCI_IO_PROTOCOL_MAP                 Map;

  // Releases any resources allocated by Map(). 
  EFI_PCI_IO_PROTOCOL_UNMAP               Unmap;

  // Allocates pages that are suitable for a common buffer mapping.
  EFI_PCI_IO_PROTOCOL_ALLOCATE_BUFFER     AllocateBuffer;

  // Frees pages that were allocated with AllocateBuffer(). 
  EFI_PCI_IO_PROTOCOL_FREE_BUFFER         FreeBuffer;

  // Flushes all PCI posted write transactions to system memory.
  EFI_PCI_IO_PROTOCOL_FLUSH               Flush;

  // Retrieves this PCI controller’s current PCI bus number, device number, and function number.
  EFI_PCI_IO_PROTOCOL_GET_LOCATION        GetLocation;

  // Performs an operation on the attributes that this PCI controller supports. 
  EFI_PCI_IO_PROTOCOL_ATTRIBUTES          Attributes;

  // Gets the attributes that this PCI controller supports setting on a BAR using SetBarAttributes(), 
  // and retrieves the list of resource descriptors for a BAR.
  EFI_PCI_IO_PROTOCOL_GET_BAR_ATTRIBUTES  GetBarAttributes;

  // Sets the attributes for a range of a BAR on a PCI controller.
  EFI_PCI_IO_PROTOCOL_SET_BAR_ATTRIBUTES  SetBarAttributes;

  //
  // The size, in bytes, of the ROM image.
  //
  UINT64                                  RomSize;

  //
  // A pointer to the in memory copy of the ROM image. The PCI Bus Driver is responsible 
  // for allocating memory for the ROM image, and copying the contents of the ROM to memory. 
  // The contents of this buffer are either from the PCI option ROM that can be accessed 
  // through the ROM BAR of the PCI controller, or it is from a platform-specific location. 
  // The Attributes() function can be used to determine from which of these two sources 
  // the RomImage buffer was initialized.
  // 
  VOID                                    *RomImage;
};
```

## PollMem()
```c
//
// Reads from the memory space of a PCI controller. 
// Returns either when the polling exit criteria is satisfied or after a defined duration. 
//
typedef
EFI_STATUS
(EFIAPI *EFI_PCI_IO_PROTOCOL_POLL_IO_MEM)(
  IN  EFI_PCI_IO_PROTOCOL          *This,     // A pointer to the EFI_PCI_IO_PROTOCOL instance.
  IN  EFI_PCI_IO_PROTOCOL_WIDTH    Width,     // Signifies the width of the memory or I/O operations.
  IN  UINT8                        BarIndex,  // The BAR index of the standard PCI Configuration header to use 
                                              // as the base address for the memory operation to perform.
  IN  UINT64                       Offset,    // The offset within the selected BAR to start the memory operation.
  IN  UINT64                       Mask,      // Mask used for the polling criteria.
  IN  UINT64                       Value,     // The comparison value used for the polling exit criteria.
  IN  UINT64                       Delay,     // The number of 100 ns units to poll.
  OUT UINT64                       *Result    // Pointer to the last value read from the memory location.
  );
```
This function provides a standard way to poll a PCI memory location. A PCI memory read
operation is performed at the PCI memory address specified by `BarIndex` and `Offset` for
the width specified by `Width`. The result of this PCI memory read operation is stored in
`Result`. This PCI memory read operation is repeated until either a timeout of `Delay 100 ns`
units has expired, or `(Result & Mask)` is equal to `Value`.

## PollIo()
Similar to PollMem().


## Mem.Read() & Mem.Write()
```c
//
// Enable a PCI driver to access PCI controller registers in the PCI memory or I/O space.
// 
typedef
EFI_STATUS
(EFIAPI *EFI_PCI_IO_PROTOCOL_IO_MEM)(
  IN EFI_PCI_IO_PROTOCOL              *This,
  IN     EFI_PCI_IO_PROTOCOL_WIDTH    Width,    // Signifies the width of the memory or I/O operations.
  IN     UINT8                        BarIndex, // The BAR index of the standard PCI Configuration header to use 
                                                // as the base address for the memory or I/O operation to perform.
  IN     UINT64                       Offset,   // The offset within selected BAR to start Mem or I/O operations.
  IN     UINTN                        Count,    // The number of memory or I/O operations to perform.
  IN OUT VOID                         *Buffer   // For read operations, the destination buffer to store results. 
                                                // For write operations, the source buffer to write data from. 
  );
```
The Mem.Read(), Mem.Write() or Io.Read(), Io.Write() functions enable a driver to access controller registers
in the PCI memory or I/O space.

If `Width` is `EfiPciIoWidthUint8`, `EfiPciIoWidthUint16`, `EfiPciIoWidthUint32`, or
`EfiPciIoWidthUint64`, then both `Address` and `Buffer` are incremented for each of the
`Count` operations performed.

... 

Please refer to UEFI Spec 2.7, pp.839