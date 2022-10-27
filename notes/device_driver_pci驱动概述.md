- [PCI基础知识](#pci基础知识)
  - [Header Type 0x0](#header-type-0x0)
  - [Header Type 0x1 (PCI-to-PCI bridge)](#header-type-0x1-pci-to-pci-bridge)
  - [Header Type 0x2 (PCI-to-CardBus bridge)](#header-type-0x2-pci-to-cardbus-bridge)
  - [Command Register](#command-register)
  - [Status Register](#status-register)
- [kernel代码](#kernel代码)
  - [驱动注册](#驱动注册)
  - [设备初始化步骤 Device Initialization Steps](#设备初始化步骤-device-initialization-steps)
    - [使能PCI设备 Enable the PCI device](#使能pci设备-enable-the-pci-device)
    - [Request MMIO/IOP resources](#request-mmioiop-resources)
    - [Set the DMA mask size](#set-the-dma-mask-size)
    - [Setup shared control data](#setup-shared-control-data)
    - [Initialize device registers](#initialize-device-registers)
    - [Register IRQ handler](#register-irq-handler)
  - [PCI device shutdown](#pci-device-shutdown)
    - [Stop IRQs on the device](#stop-irqs-on-the-device)
    - [Release the IRQ](#release-the-irq)
    - [Stop all DMA activity](#stop-all-dma-activity)
    - [Release DMA buffers](#release-dma-buffers)
    - [Unregister from other subsystems](#unregister-from-other-subsystems)
    - [Disable Device from responding to MMIO/IO Port addresses](#disable-device-from-responding-to-mmioio-port-addresses)
    - [Release MMIO/IO Port Resource(s)](#release-mmioio-port-resources)
  - [How to access PCI config space](#how-to-access-pci-config-space)
  - [1.7. Other interesting functions](#17-other-interesting-functions)
  - [Miscellaneous hints](#miscellaneous-hints)
  - [Vendor and device identifications](#vendor-and-device-identifications)
  - [Obsolete functions](#obsolete-functions)
  - [MMIO Space and "Write Posting"](#mmio-space-and-write-posting)
参考: https://www.kernel.org/doc/html/latest/PCI/pci.html

# PCI基础知识
## Header Type 0x0

This table is applicable if the Header Type is `0x0`.

| Register | Offset | Bits 31-24 | Bits 23-16 | Bits 15-8 | Bits 7-0 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 0x0 | 0x0 | Device ID | Vendor ID |
| 0x1 | 0x4 | Status | Command |
| 0x2 | 0x8 | Class code | Subclass | Prog IF | Revision ID |
| 0x3 | 0xC | BIST | Header type | Latency Timer | Cache Line Size |
| 0x4 | 0x10 | Base address #0 (BAR0) |
| 0x5 | 0x14 | Base address #1 (BAR1) |
| 0x6 | 0x18 | Base address #2 (BAR2) |
| 0x7 | 0x1C | Base address #3 (BAR3) |
| 0x8 | 0x20 | Base address #4 (BAR4) |
| 0x9 | 0x24 | Base address #5 (BAR5) |
| 0xA | 0x28 | Cardbus CIS Pointer |
| 0xB | 0x2C | Subsystem ID | Subsystem Vendor ID |
| 0xC | 0x30 | Expansion ROM base address |
| 0xD | 0x34 | Reserved | Capabilities Pointer |
| 0xE | 0x38 | Reserved |
| 0xF | 0x3C | Max latency | Min Grant | Interrupt PIN | Interrupt Line |

## Header Type 0x1 (PCI-to-PCI bridge)

This table is applicable if the Header Type is `0x1` (PCI-to-PCI bridge)

| Register | Offset | Bits 31-24 | Bits 23-16 | Bits 15-8 | Bits 7-0 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 0x0 | 0x0 | Device ID | Vendor ID |
| 0x1 | 0x4 | Status | Command |
| 0x2 | 0x8 | Class code | Subclass | Prog IF | Revision ID |
| 0x3 | 0xC | BIST | Header type | Latency Timer | Cache Line Size |
| 0x4 | 0x10 | Base address #0 (BAR0) |
| 0x5 | 0x14 | Base address #1 (BAR1) |
| 0x6 | 0x18 | Secondary Latency Timer | Subordinate Bus Number | Secondary Bus Number | Primary Bus Number |
| 0x7 | 0x1C | Secondary Status | I/O Limit | I/O Base |
| 0x8 | 0x20 | Memory Limit | Memory Base |
| 0x9 | 0x24 | Prefetchable Memory Limit | Prefetchable Memory Base |
| 0xA | 0x28 | Prefetchable Base Upper 32 Bits |
| 0xB | 0x2C | Prefetchable Limit Upper 32 Bits |
| 0xC | 0x30 | I/O Limit Upper 16 Bits | I/O Base Upper 16 Bits |
| 0xD | 0x34 | Reserved | Capability Pointer |
| 0xE | 0x38 | Expansion ROM base address |
| 0xF | 0x3C | Bridge Control | Interrupt PIN | Interrupt Line |

## Header Type 0x2 (PCI-to-CardBus bridge)

This table is applicable if the Header Type is `0x2` (PCI-to-CardBus bridge)

| Register | Offset | Bits 31-24 | Bits 23-16 | Bits 15-8 | Bits 7-0 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| 0x0 | 0x0 | Device ID | Vendor ID |
| 0x1 | 0x4 | Status | Command |
| 0x2 | 0x8 | Class code | Subclass | Prog IF | Revision ID |
| 0x3 | 0xC | BIST | Header type | Latency Timer | Cache Line Size |
| 0x4 | 0x10 | CardBus Socket/ExCa base address |
| 0x5 | 0x14 | Secondary status | Reserved | Offset of capabilities list |
| 0x6 | 0x18 | CardBus latency timer | Subordinate bus number | CardBus bus number | PCI bus number |
| 0x7 | 0x1C | Memory Base Address 0 |
| 0x8 | 0x20 | Memory Limit 0 |
| 0x9 | 0x24 | Memory Base Address 1 |
| 0xA | 0x28 | Memory Limit 1 |
| 0xB | 0x2C | I/O Base Address 0 |
| 0xC | 0x30 | I/O Limit 0 |
| 0xD | 0x34 | I/O Base Address 1 |
| 0xE | 0x38 | I/O Limit 1 |
| 0xF | 0x3C | Bridge Control | Interrupt PIN | Interrupt Line |
| 0x10 | 0x40 | Subsystem Vendor ID | Subsystem Device ID |
| 0x11 | 0x44 | 16-bit PC Card legacy mode base address |

## Command Register

Here is the layout of the Command register:

| Bits 11-15 | Bit 10 | Bit 9 | Bit 8 | Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bit 2 | Bit 1 | Bit 0 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Reserved | Interrupt Disable | Fast Back-to-Back Enable | SERR# Enable | Reserved | Parity Error Response | VGA Palette Snoop | Memory Write and Invalidate Enable | Special Cycles | Bus Master | Memory Space | I/O Space |

* **Interrupt Disable**: If set to 1 the assertion of the devices INTx# signal is disabled; otherwise, assertion of the signal is enabled.
* **Fast Back-Back Enable**: If set to 1 indicates a device is allowed to generate fast back-to-back transactions; otherwise, fast back-to-back transactions are only allowed to the same agent.
* **SERR# Enable**: If set to 1 the SERR# driver is enabled; otherwise, the driver is disabled.
* **Bit 7**: As of revision 3.0 of the PCI local bus specification this bit is hardwired to 0\. In earlier versions of the specification this bit was used by devices and may have been hardwired to 0, 1, or implemented as a read/write bit.
* **Parity Error Response**: If set to 1 the device will take its normal action when a parity error is detected; otherwise, when an error is detected, the device will set bit 15 of the Status register (Detected Parity Error Status Bit), but will not assert the PERR# (Parity Error) pin and will continue operation as normal.
* **VGA Palette Snoop**: If set to 1 the device does not respond to palette register writes and will snoop the data; otherwise, the device will trate palette write accesses like all other accesses.
* **Memory Write and Invalidate Enable**: If set to 1 the device can generate the Memory Write and Invalidate command; otherwise, the Memory Write command must be used.
* **Special Cycles**: If set to 1 the device can monitor Special Cycle operations; otherwise, the device will ignore them.
* **Bus Master**: If set to 1 the device can behave as a bus master; otherwise, the device can not generate PCI accesses.
* **Memory Space**: If set to 1 the device can respond to Memory Space accesses; otherwise, the device's response is disabled.
* **I/O Space**: If set to 1 the device can respond to I/O Space accesses; otherwise, the device's response is disabled.

If the kernel configures the BARs of the devices, the kernel also have to enable bits 0 and 1 for it to activate.

## Status Register

Here is the layout of the Status register:

| Bit 15 | Bit 14 | Bit 13 | Bit 12 | Bit 11 | Bits 9-10 | Bit 8 | Bit 7 | Bit 6 | Bit 5 | Bit 4 | Bit 3 | Bits 0-2 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| Detected Parity Error | Signaled System Error | Received Master Abort | Received Target Abort | Signaled Target Abort | DEVSEL Timing | Master Data Parity Error | Fast Back-to-Back Capable | Reserved | 66 MHz Capable | Capabilities List | Interrupt Status | Reserved |

* **Detected Parity Error**: This bit will be set to 1 whenever the device detects a parity error, even if parity error handling is disabled.
* **Signalled System Error**: This bit will be set to 1 whenever the device asserts SERR#.
* **Received Master Abort**: This bit will be set to 1, by a master device, whenever its transaction (except for Special Cycle transactions) is terminated with Master-Abort.
* **Received Target Abort**: This bit will be set to 1, by a master device, whenever its transaction is terminated with Target-Abort.
* **Signalled Target Abort**: This bit will be set to 1 whenever a target device terminates a transaction with Target-Abort.
* **DEVSEL Timing**: Read only bits that represent the slowest time that a device will assert DEVSEL# for any bus command except Configuration Space read and writes. Where a value of `0x0` represents fast timing, a value of `0x1` represents medium timing, and a value of `0x2` represents slow timing.
* **Master Data Parity Error**: This bit is only set when the following conditions are met. The bus agent asserted PERR# on a read or observed an assertion of PERR# on a write, the agent setting the bit acted as the bus master for the operation in which the error occurred, and bit 6 of the Command register (Parity Error Response bit) is set to 1.
* **Fast Back-to-Back Capable**: If set to 1 the device can accept fast back-to-back transactions that are not from the same agent; otherwise, transactions can only be accepted from the same agent.
* **Bit 6**: As of revision 3.0 of the PCI Local Bus specification this bit is reserved. In revision 2.1 of the specification this bit was used to indicate whether or not a device supported User Definable Features.
* **66 MHz Capable**: If set to 1 the device is capable of running at 66 MHz; otherwise, the device runs at 33 MHz.
* **Capabilities List**: If set to 1 the device implements the pointer for a New Capabilities Linked list at offset `0x34`; otherwise, the linked list is not available.
* **Interrupt Status**: Represents the state of the device's INTx# signal. If set to 1 and bit 10 of the Command register (Interrupt Disable bit) is set to 0 the signal will be asserted; otherwise, the signal will be ignored.

Recall that the PCI devices follow **little ENDIAN** ordering. The lower addresses contain the least significant portions of the field. Software to manipulate this structure must take particular care that the endian-ordering follows the PCI devices, not the CPUs.

# kernel代码
## 驱动注册
PCI drivers "discover" PCI devices in a system via `pci_register_driver()`. Actually, it's the other way around. When the PCI generic code discovers a new device, the driver with a matching "description" will be notified. Details on this below.

`pci_register_driver()` leaves most of the probing for devices to the PCI layer and supports online insertion/removal of devices [thus supporting hot-pluggable PCI, CardBus, and Express-Card in a single driver]. `pci_register_driver()` call requires passing in a table of function pointers and thus dictates the high level structure of a driver.

Once the driver knows about a PCI device and takes ownership, the driver generally needs to perform the following initialization:

*   Enable the device
*   Request MMIO/IOP resources
*   Set the DMA mask size (for both coherent and streaming DMA)
*   Allocate and initialize shared control data (`pci_allocate_coherent()`)
*   Access device configuration space (if needed)
*   Register IRQ handler ([`request_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.request_irq "request_irq"))
*   Initialize non-PCI (i.e. LAN/SCSI/etc parts of the chip)
*   Enable DMA/processing engines

When done using the device, and perhaps the module needs to be unloaded, the driver needs to take the follow steps:

*   Disable the device from generating IRQs
*   Release the IRQ ([`free_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.free_irq "free_irq"))
*   Stop all DMA activity
*   Release DMA buffers (both streaming and coherent)
*   Unregister from other subsystems (e.g. scsi or netdev)
*   Release MMIO/IOP resources
*   Disable the device

## 设备初始化步骤 [Device Initialization Steps](https://www.kernel.org/doc/html/latest/PCI/pci.html#device-initialization-steps)

As noted in the introduction, most PCI drivers need the following steps for device initialization:

*   Enable the device
*   Request MMIO/IOP resources
*   Set the DMA mask size (for both coherent and streaming DMA)
*   Allocate and initialize shared control data (pci_allocate_coherent())
*   Access device configuration space (if needed)
*   Register IRQ handler ([`request_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.request_irq "request_irq"))
*   Initialize non-PCI (i.e. LAN/SCSI/etc parts of the chip)
*   Enable DMA/processing engines.

The driver can access PCI config space registers at any time. (Well, almost. When running BIST, config space can go away…but that will just result in a PCI Bus Master Abort and config reads will return garbage).

### 使能PCI设备 [Enable the PCI device](https://www.kernel.org/doc/html/latest/PCI/pci.html#enable-the-pci-device)

Before touching any device registers, the driver needs to enable the PCI device by calling [`pci_enable_device()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_enable_device "pci_enable_device"). This will:

*   wake up the device if it was in suspended state,
*   allocate I/O and memory regions of the device (if BIOS did not),
*   allocate an IRQ (if BIOS did not).

使能pci的DMA master: 对应pci配置空间的通用寄存器Command Register的Bus Master位.  
[`pci_set_master()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_set_master "pci_set_master") will enable DMA by setting the bus master bit in the `PCI_COMMAND` register. It also fixes the latency timer value if it's set to something bogus by the BIOS. [`pci_clear_master()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_clear_master "pci_clear_master") will disable DMA by clearing the bus master bit.

If the PCI device can use the PCI Memory-Write-Invalidate transaction, call [`pci_set_mwi()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_set_mwi "pci_set_mwi"). This enables the `PCI_COMMAND` bit for Mem-Wr-Inval and also ensures that the cache line size register is set correctly. Check the return value of [`pci_set_mwi()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_set_mwi "pci_set_mwi") as not all architectures or chip-sets may support Memory-Write-Invalidate. Alternatively, if Mem-Wr-Inval would be nice to have but is not required, call [`pci_try_set_mwi()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_try_set_mwi "pci_try_set_mwi") to have the system do its best effort at enabling Mem-Wr-Inval.

### [Request MMIO/IOP resources](https://www.kernel.org/doc/html/latest/PCI/pci.html#request-mmio-iop-resources)

Memory (MMIO), and I/O port addresses should NOT be read directly from the PCI device config space. Use the values in the pci_dev structure as the PCI "bus address" might have been remapped to a "host physical" address by the arch/chip-set specific kernel support.

See [The io_mapping functions](https://www.kernel.org/doc/html/latest/driver-api/io-mapping.html) for how to access device registers or device memory.

The device driver needs to call [`pci_request_region()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_request_region "pci_request_region") to verify no other device is already using the same address resource. Conversely, drivers should call [`pci_release_region()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_release_region "pci_release_region") AFTER calling [`pci_disable_device()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_disable_device "pci_disable_device"). The idea is to prevent two devices colliding on the same address range.

Generic flavors of [`pci_request_region()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_request_region "pci_request_region") are `request_mem_region()` (for MMIO ranges) and `request_region()` (for IO Port ranges). Use these for address resources that are not described by "normal" PCI BARs.

Also see [`pci_request_selected_regions()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_request_selected_regions "pci_request_selected_regions") below.

### Set the DMA mask size
While all drivers should explicitly indicate the DMA capability (e.g. 32 or 64 bit) of the PCI bus master, devices with more than 32-bit bus master capability for streaming data need the driver to "register" this capability by calling `pci_set_dma_mask()` with appropriate parameters. In general this allows more efficient DMA on systems where System RAM exists above 4G _physical_ address.

Drivers for all PCI-X and PCIe compliant devices must call `pci_set_dma_mask()` as they are 64-bit DMA devices.

Similarly, drivers must also "register" this capability if the device can directly address "consistent memory" in System RAM above 4G physical address by calling `pci_set_consistent_dma_mask()`. Again, this includes drivers for all PCI-X and PCIe compliant devices. Many 64-bit "PCI" devices (before PCI-X) and some PCI-X devices are 64-bit DMA capable for payload ("streaming") data but not control ("consistent") data.

### [Setup shared control data](https://www.kernel.org/doc/html/latest/PCI/pci.html#setup-shared-control-data)

Once the DMA masks are set, the driver can allocate "consistent" (a.k.a. shared) memory. See [Dynamic DMA mapping using the generic device](https://www.kernel.org/doc/html/latest/core-api/dma-api.html) for a full description of the DMA APIs. This section is just a reminder that it needs to be done before enabling DMA on the device.

### [Initialize device registers](https://www.kernel.org/doc/html/latest/PCI/pci.html#initialize-device-registers)

Some drivers will need specific "capability" fields programmed or other "vendor specific" register initialized or reset. E.g. clearing pending interrupts.

### [Register IRQ handler](https://www.kernel.org/doc/html/latest/PCI/pci.html#register-irq-handler)

While calling [`request_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.request_irq "request_irq") is the last step described here, this is often just another intermediate step to initialize a device. This step can often be deferred until the device is opened for use.

All interrupt handlers for IRQ lines should be registered with `IRQF_SHARED` and use the devid to map IRQs to devices (remember that all PCI IRQ lines can be shared).

[`request_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.request_irq "request_irq") will associate an interrupt handler and device handle with an interrupt number. Historically interrupt numbers represent IRQ lines which run from the PCI device to the Interrupt controller. With MSI and MSI-X (more below) the interrupt number is a CPU "vector".

[`request_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.request_irq "request_irq") also enables the interrupt. Make sure the device is quiesced and does not have any interrupts pending before registering the interrupt handler.

MSI and MSI-X are PCI capabilities. Both are "Message Signaled Interrupts" which deliver interrupts to the CPU via a DMA write to a Local APIC. The fundamental difference between MSI and MSI-X is how multiple "vectors" get allocated. MSI requires contiguous blocks of vectors while MSI-X can allocate several individual ones.

MSI capability can be enabled by calling pci_alloc_irq_vectors() with the PCI_IRQ_MSI and/or PCI_IRQ_MSIX flags before calling [`request_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.request_irq "request_irq"). This causes the PCI support to program CPU vector data into the PCI device capability registers. Many architectures, chip-sets, or BIOSes do NOT support MSI or MSI-X and a call to pci_alloc_irq_vectors with just the PCI_IRQ_MSI and PCI_IRQ_MSIX flags will fail, so try to always specify PCI_IRQ_LEGACY as well.

Drivers that have different interrupt handlers for MSI/MSI-X and legacy INTx should chose the right one based on the msi_enabled and msix_enabled flags in the pci_dev structure after calling pci_alloc_irq_vectors.

There are (at least) two really good reasons for using MSI:

1.  MSI is an exclusive interrupt vector by definition. This means the interrupt handler doesn't have to verify its device caused the interrupt.

2.  MSI avoids DMA/IRQ race conditions. DMA to host memory is guaranteed to be visible to the host CPU(s) when the MSI is delivered. This is important for both data coherency and avoiding stale control data. This guarantee allows the driver to omit MMIO reads to flush the DMA stream.

See drivers/infiniband/hw/mthca/ or drivers/net/tg3.c for examples of MSI/MSI-X usage.

## [PCI device shutdown](https://www.kernel.org/doc/html/latest/PCI/pci.html#pci-device-shutdown)

When a PCI device driver is being unloaded, most of the following steps need to be performed:

*   Disable the device from generating IRQs
*   Release the IRQ ([`free_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.free_irq "free_irq"))
*   Stop all DMA activity
*   Release DMA buffers (both streaming and consistent)
*   Unregister from other subsystems (e.g. scsi or netdev)
*   Disable device from responding to MMIO/IO Port addresses
*   Release MMIO/IO Port resource(s)

### [Stop IRQs on the device](https://www.kernel.org/doc/html/latest/PCI/pci.html#stop-irqs-on-the-device)

How to do this is chip/device specific. If it's not done, it opens the possibility of a "screaming interrupt" if (and only if) the IRQ is shared with another device.

When the shared IRQ handler is "unhooked", the remaining devices using the same IRQ line will still need the IRQ enabled. Thus if the "unhooked" device asserts IRQ line, the system will respond assuming it was one of the remaining devices asserted the IRQ line. Since none of the other devices will handle the IRQ, the system will "hang" until it decides the IRQ isn't going to get handled and masks the IRQ (100,000 iterations later). Once the shared IRQ is masked, the remaining devices will stop functioning properly. Not a nice situation.

This is another reason to use MSI or MSI-X if it's available. MSI and MSI-X are defined to be exclusive interrupts and thus are not susceptible to the "screaming interrupt" problem.

### [Release the IRQ](https://www.kernel.org/doc/html/latest/PCI/pci.html#release-the-irq)

Once the device is quiesced (no more IRQs), one can call [`free_irq()`](https://www.kernel.org/doc/html/latest/core-api/genericirq.html#c.free_irq "free_irq"). This function will return control once any pending IRQs are handled, "unhook" the drivers IRQ handler from that IRQ, and finally release the IRQ if no one else is using it.

### [Stop all DMA activity](https://www.kernel.org/doc/html/latest/PCI/pci.html#stop-all-dma-activity)

It's extremely important to stop all DMA operations BEFORE attempting to deallocate DMA control data. Failure to do so can result in memory corruption, hangs, and on some chip-sets a hard crash.

Stopping DMA after stopping the IRQs can avoid races where the IRQ handler might restart DMA engines.

While this step sounds obvious and trivial, several "mature" drivers didn't get this step right in the past.

### [Release DMA buffers](https://www.kernel.org/doc/html/latest/PCI/pci.html#release-dma-buffers)

Once DMA is stopped, clean up streaming DMA first. I.e. unmap data buffers and return buffers to "upstream" owners if there is one.

Then clean up "consistent" buffers which contain the control data.

See [Dynamic DMA mapping using the generic device](https://www.kernel.org/doc/html/latest/core-api/dma-api.html) for details on unmapping interfaces.

### [Unregister from other subsystems](https://www.kernel.org/doc/html/latest/PCI/pci.html#unregister-from-other-subsystems)

Most low level PCI device drivers support some other subsystem like USB, ALSA, SCSI, NetDev, Infiniband, etc. Make sure your driver isn't losing resources from that other subsystem. If this happens, typically the symptom is an Oops (panic) when the subsystem attempts to call into a driver that has been unloaded.

### [Disable Device from responding to MMIO/IO Port addresses](https://www.kernel.org/doc/html/latest/PCI/pci.html#disable-device-from-responding-to-mmio-io-port-addresses)

`io_unmap()` MMIO or IO Port resources and then call [`pci_disable_device()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_disable_device "pci_disable_device"). This is the symmetric opposite of [`pci_enable_device()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_enable_device "pci_enable_device"). Do not access device registers after calling [`pci_disable_device()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_disable_device "pci_disable_device").

### [Release MMIO/IO Port Resource(s)](https://www.kernel.org/doc/html/latest/PCI/pci.html#release-mmio-io-port-resource-s)

Call [`pci_release_region()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_release_region "pci_release_region") to mark the MMIO or IO Port range as available. Failure to do so usually results in the inability to reload the driver.

## [How to access PCI config space](https://www.kernel.org/doc/html/latest/PCI/pci.html#how-to-access-pci-config-space)

You can use `pci_(read|write)_config_(byte|word|dword)` to access the config space of a device represented by `struct pci_dev *`. All these functions return 0 when successful or an error code (`PCIBIOS_…`) which can be translated to a text string by pcibios_strerror. Most drivers expect that accesses to valid PCI devices don't fail.

If you don't have a struct pci_dev available, you can call `pci_bus_(read|write)_config_(byte|word|dword)` to access a given device and function on that bus.

If you access fields in the standard portion of the config header, please use symbolic names of locations and bits declared in <linux/pci.h>.

If you need to access Extended PCI Capability registers, just call [`pci_find_capability()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_find_capability "pci_find_capability") for the particular capability and it will find the corresponding register block for you.

## 1.7. Other interesting functions[](https://www.kernel.org/doc/html/latest/PCI/pci.html#other-interesting-functions)

|func|desc
|--|--
|[`pci_get_domain_bus_and_slot()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_get_domain_bus_and_slot "pci_get_domain_bus_and_slot")|Find pci_dev corresponding to given domain, bus and slot and number. If the device is found, its reference count is increased.
|[`pci_set_power_state()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_set_power_state "pci_set_power_state")|Set PCI Power Management state (0=D0 … 3=D3)
|[`pci_find_capability()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_find_capability "pci_find_capability")|Find specified capability in device's capability list.
|`pci_resource_start()`|Returns bus start address for a given PCI region
|`pci_resource_end()`|Returns bus end address for a given PCI region
|`pci_resource_len()`|Returns the byte length of a PCI region
|`pci_set_drvdata()`|Set private driver data pointer for a pci_dev
|`pci_get_drvdata()`|Return private driver data pointer for a pci_dev
|[`pci_set_mwi()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_set_mwi "pci_set_mwi")|Enable Memory-Write-Invalidate transactions.
|[`pci_clear_mwi()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_clear_mwi "pci_clear_mwi")|Disable Memory-Write-Invalidate transactions.

## [Miscellaneous hints](https://www.kernel.org/doc/html/latest/PCI/pci.html#miscellaneous-hints)

When displaying PCI device names to the user (for example when a driver wants to tell the user what card has it found), please use pci_name(pci_dev).

Always refer to the PCI devices by a pointer to the pci_dev structure. All PCI layer functions use this identification and it's the only reasonable one. Don't use bus/slot/function numbers except for very special purposes – on systems with multiple primary buses their semantics can be pretty complex.

Don't try to turn on Fast Back to Back writes in your driver. All devices on the bus need to be capable of doing it, so this is something which needs to be handled by platform and generic code, not individual drivers.

## [Vendor and device identifications](https://www.kernel.org/doc/html/latest/PCI/pci.html#vendor-and-device-identifications)

Do not add new device or vendor IDs to include/linux/pci_ids.h unless they are shared across multiple drivers. You can add private definitions in your driver if they're helpful, or just use plain hex constants.

The device IDs are arbitrary hex numbers (vendor controlled) and normally used only in a single location, the pci_device_id table.

Please DO submit new vendor/device IDs to [https://pci-ids.ucw.cz/](https://pci-ids.ucw.cz/). There's a mirror of the pci.ids file at [https://github.com/pciutils/pciids](https://github.com/pciutils/pciids).

## [Obsolete functions](https://www.kernel.org/doc/html/latest/PCI/pci.html#obsolete-functions)

There are several functions which you might come across when trying to port an old driver to the new PCI interface. They are no longer present in the kernel as they aren't compatible with hotplug or PCI domains or having sane locking.

|func|desc
|--|--|
|`pci_find_device()`|Superseded by [`pci_get_device()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_get_device "pci_get_device")
|`pci_find_subsys()`|Superseded by [`pci_get_subsys()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_get_subsys "pci_get_subsys")
|`pci_find_slot()`|Superseded by [`pci_get_domain_bus_and_slot()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_get_domain_bus_and_slot "pci_get_domain_bus_and_slot")
|[`pci_get_slot()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_get_slot "pci_get_slot")|Superseded by [`pci_get_domain_bus_and_slot()`](https://www.kernel.org/doc/html/latest/driver-api/pci/pci.html#c.pci_get_domain_bus_and_slot "pci_get_domain_bus_and_slot")

The alternative is the traditional PCI device driver that walks PCI device lists. This is still possible but discouraged.

## [MMIO Space and "Write Posting"](https://www.kernel.org/doc/html/latest/PCI/pci.html#mmio-space-and-write-posting)

Converting a driver from using I/O Port space to using MMIO space often requires some additional changes. Specifically, "write posting" needs to be handled. Many drivers (e.g. tg3, acenic, sym53c8xx_2) already do this. I/O Port space guarantees write transactions reach the PCI device before the CPU can continue. Writes to MMIO space allow the CPU to continue before the transaction reaches the PCI device. HW weenies call this "Write Posting" because the write completion is "posted" to the CPU before the transaction has reached its destination.

Thus, timing sensitive code should add readl() where the CPU is expected to wait before doing other work. The classic "bit banging" sequence works fine for I/O Port space:

```c
for (i = 8; --i; val >>= 1) {
        outb(val & 1, ioport_reg);      /* write bit */
        udelay(10);
}
```

The same sequence for MMIO space should be:

```c
for (i = 8; --i; val >>= 1) {
        writeb(val & 1, mmio_reg);      /* write bit */
        readb(safe_mmio_reg);           /* flush posted write */
        udelay(10);
}
```

It is important that "safe_mmio_reg" not have any side effects that interferes with the correct operation of the device.

Another case to watch out for is when resetting a PCI device. Use PCI Configuration space reads to flush the `writel()`. This will gracefully handle the PCI master abort on all platforms if the PCI device is expected to not respond to a `readl()`. Most x86 platforms will allow MMIO reads to master abort (a.k.a. "Soft Fail") and return garbage (e.g. ~0). But many RISC platforms will crash (a.k.a."Hard Fail").
