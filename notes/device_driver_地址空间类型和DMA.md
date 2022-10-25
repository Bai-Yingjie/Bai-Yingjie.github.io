- [地址类型](#地址类型)
- [概念说明](#概念说明)
  - [不同地址的转换](#不同地址的转换)
  - [DMA类型](#dma类型)
- [具体代码](#具体代码)
  - [哪些内存可以DMA?](#哪些内存可以dma)
  - [DMA addressing capabilities](#dma-addressing-capabilities)
  - [Using Consistent DMA mappings](#using-consistent-dma-mappings)
  - [DMA Direction](#dma-direction)
  - [Using Streaming DMA mappings](#using-streaming-dma-mappings)
  - [Handling Errors](#handling-errors)
  - [Optimizing Unmap State Space Consumption](#optimizing-unmap-state-space-consumption)
  - [Platform Issues](#platform-issues)
- [Dynamic DMA mapping using the generic device](#dynamic-dma-mapping-using-the-generic-device)
  - [Part I - dma_API](#part-i---dma_api)
  - [Part Ia - Using large DMA-coherent buffers](#part-ia---using-large-dma-coherent-buffers)
  - [Part Ib - Using small DMA-coherent buffers](#part-ib---using-small-dma-coherent-buffers)
  - [Part Ic - DMA addressing limitations](#part-ic---dma-addressing-limitations)
  - [Part Id - Streaming DMA mappings](#part-id---streaming-dma-mappings)
  - [Part II - Non-coherent DMA allocations](#part-ii---non-coherent-dma-allocations)

# 地址类型
* 虚拟地址: kernel运行在虚拟地址, kmalloc(), vmalloc()等返回的都是虚拟地址, 可以用`void *`来表示
* 物理地址: 用`phys_addr_t`或`resource_size_t`来表示, 在`/proc/iomem`是物理地址. 驱动要用的话, 要先[`ioremap()`](https://www.kernel.org/doc/html/latest/driver-api/device-io.html#c.ioremap "ioremap")到虚拟地址
* bus地址: 是对IO设备来说的. 如果一个IO设备由MMIO地址(memory mapped IO), 或者使用DMA来读写系统内存, 那这个设备使用的地址就是bus address. 有的系统下bus地址就是物理地址, 但大部分情况下, bus地址要经过IOMMU转换. IOMMU可以任意map物理地址和bus地址.

从设备角度来看, 它操做的是bus地址. 个人理解bus地址就是每个设备视角下的"虚拟地址", IOMMU就是负责翻译物理地址和bus地址的. 这个翻译对设备来说是隔离的, 比如在一个64位系统下, 某个pcie设备使能了64bit DMA, 但设备可以只用经过IOMMU的32bit DMA地址.

# 概念说明
## 不同地址的转换
```
             CPU                  CPU                  Bus
           Virtual              Physical             Address
           Address              Address               Space
            Space                Space

          +-------+             +------+             +------+
          |       |             |MMIO  |   Offset    |      |
          |       |  Virtual    |Space |   applied   |      |
        C +-------+ --------> B +------+ ----------> +------+ A
          |       |  mapping    |      |   by host   |      |
+-----+   |       |             |      |   bridge    |      |   +--------+
|     |   |       |             +------+             |      |   |        |
| CPU |   |       |             | RAM  |             |      |   | Device |
|     |   |       |             |      |             |      |   |        |
+-----+   +-------+             +------+             +------+   +--------+
          |       |  Virtual    |Buffer|   Mapping   |      |
        X +-------+ --------> Y +------+ <---------- +------+ Z
          |       |  mapping    | RAM  |   by IOMMU
          |       |             |      |
          |       |             |      |
          +-------+             +------+
```
在系统枚举期间, kernel知道IO设备和所有的MMIO空间, 也知道host gridge的空间.  
比如一个pci设备有个BAR, kernel读取BAR的bus地址`(A)`, 转换成CPU的物理地址`(B)`. B地址存储在`resource`结构体里, 在`/proc/iomem`里面体现.

上面是kernel发现一个pci设备, 并读取pci标准的BAR寄存器, 然后分配resource空间的过程, 这个过程是generic的, 不需要驱动参与.

到驱动参与的时候, 驱动声明它可以own一个device, 它就可以使用[`ioremap()`](https://www.kernel.org/doc/html/latest/driver-api/device-io.html#c.ioremap "ioremap")把这个物理地址`(B)`转换成虚拟地址`(C)`. 驱动就可以使用`ioread32(C)`来访问这个device的bus地址`(A)`

如果设备支持DMA, 驱动就用[`kmalloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc "kmalloc")来申请一个连续的buffer, 返回虚拟地址`(X)`. 虚拟地址`(X)`通过系统MMU转换成物理地址`(Y)`.
* 驱动可以使用虚拟地址X来访问这个buffer
* 但DMA控制器不走CPU的MMU, 就没法用X

在没有IOMMU的系统中, 设备可以直接使用物理地址`(Y)`; 但有IOMMU的系统, 需要通过IOMMU把物理地址`(Y)`转换成DMA地址`(Z)`. 这就是DMA API的由来:  
比如`dma_map_single()`用传入的虚拟地址`(X)`配置好需要的IOMMU映射, 返回DMA地址`(Z)`. 驱动在把这个DMA地址`(Z)`告诉device: 往这个地址`(Z)`做DMA. 因为IOMMU已经把物理地址`(Y)`映射成了`(Z)`.

DMA transfer完成后, 驱动再unmap这个映射.

DMA的API是架构无关的. 驱动应该使用架构无关的API, 比如`dma_map_*()`而不是具体总线的`pci_map_*()`

要使用DMA的API:
```c
#include <linux/dma-mapping.h>
```

参考: [设备地址](platform_device_driver.md)

## DMA类型
DMA有两种类型:
* Consistent/synchronous/coherent: 反正都是一致的意思. 就是说CPU和设备对DMA buffer的操做即使不用显示的flush操做, 对方都看得到. 用`dma_alloc_coherent()` API的就是consistent方式. 比如:
    * Network card DMA ring descriptors.
    * SCSI adapter mailbox command data structures.
    * Device firmware microcode executed out of main memory.

* Streaming/asynchronous/outside the coherency domain: 通常是一次DMA transfer用的, 用完马上unmap. 设备可以优化顺序访问. 用`dma_map_{single,sg}()` API的是streaming
The interfaces for using this type of mapping were designed in such a way that an implementation can make whatever performance optimizations the hardware allows. To this end, when using such mappings you must be explicit about what you want to happen.
    * Networking buffers transmitted/received by a device. 注意这里是指packet的data, 不是descriptor
    * Filesystem buffers written/read by a SCSI device.
    
Neither type of DMA mapping has alignment restrictions that come from the underlying bus, although some devices may have such restrictions. Also, systems with caches that aren’t DMA-coherent will work better when the underlying buffers don’t share cache lines with other data.

注意, 对Consistent DMA来说, 必要的memory barrier还要driver自己写.  
Consistent DMA memory does not preclude the usage of proper memory barriers. The CPU may reorder stores to consistent memory just as it may normal memory. Example: if it is important for the device to see the first word of a descriptor updated before the second, you must do something like:
```c
desc->word0 = address;
wmb();
desc->word1 = DESC_VALID;
```
in order to get correct behavior on all platforms.

Also, on some platforms your driver may need to flush CPU write buffers in much the same way as it needs to flush write buffers found in PCI bridges (such as by reading a register’s value after writing it).


# 具体代码
## 哪些内存可以DMA?
可以的:
* [`kmalloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc "kmalloc")
* [`kmem_cache_alloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmem_cache_alloc "kmem_cache_alloc")
* `__get_free_page*()`

不行的:
* [`vmalloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.vmalloc "vmalloc")
* `kmap()`: 和`vmalloc()`差不多.
* cacheline不对齐的地址

好像block IO和network的buffer保证能DMA

## DMA addressing capabilities
调用下面的函数来配置DMA的属性
The standard 64-bit addressing device would do something like this:
```c
int dma_set_mask_and_coherent(struct device *dev, u64 mask);
```
If the device only supports 32-bit addressing for descriptors in the coherent allocations, but supports full 64-bits for streaming mappings it would look like this:
```c
if (dma_set_mask(dev, DMA_BIT_MASK(64))) {
        dev_warn(dev, "mydev: No suitable DMA available\n");
        goto ignore_this_device;
}
```
The coherent mask will always be able to set the same or a smaller mask as the streaming mask. However for the rare case that a device driver only uses consistent allocations, one would have to check the return value from dma_set_coherent_mask().

Finally, if your device can only drive the low 24-bits of address you might do something like:
```c
if (dma_set_mask(dev, DMA_BIT_MASK(24))) {
        dev_warn(dev, "mydev: 24-bit DMA addressing not available\n");
        goto ignore_this_device;
}
```

When dma_set_mask() or dma_set_mask_and_coherent() is successful, and returns zero, the kernel saves away this mask you have provided. The kernel will use this information later when you make DMA mappings.

## Using Consistent DMA mappings
To allocate and map large (PAGE_SIZE or so) consistent DMA regions, you should do:
```c
dma_addr_t dma_handle;

cpu_addr = dma_alloc_coherent(dev, size, &dma_handle, gfp);
```

where device is a `struct device *`. This may be called in interrupt context with the GFP_ATOMIC flag.

Size is the length of the region you want to allocate, in bytes.

This routine will allocate RAM for that region, so it acts similarly to __get_free_pages() (but takes size instead of a page order). If your driver needs regions sized smaller than a page, you may prefer using the dma_pool interface, described below.

The consistent DMA mapping interfaces, will by default return a DMA address which is 32-bit addressable. Even if the device indicates (via the DMA mask) that it may address the upper 32-bits, consistent allocation will only return > 32-bit addresses for DMA if the consistent DMA mask has been explicitly changed via dma_set_coherent_mask(). This is true of the dma_pool interface as well.

dma_alloc_coherent() returns two values: the virtual address which you can use to access it from the CPU and dma_handle which you pass to the card.

The CPU virtual address and the DMA address are both guaranteed to be aligned to the smallest PAGE_SIZE order which is greater than or equal to the requested size. This invariant exists (for example) to guarantee that if you allocate a chunk which is smaller than or equal to 64 kilobytes, the extent of the buffer you receive will not cross a 64K boundary.

To unmap and free such a DMA region, you call:

```c
dma_free_coherent(dev, size, cpu_addr, dma_handle);
```

where dev, size are the same as in the above call and cpu_addr and dma_handle are the values dma_alloc_coherent() returned to you. This function may not be called in interrupt context.

If your driver needs lots of smaller memory regions, you can write custom code to subdivide pages returned by dma_alloc_coherent(), or you can use the dma_pool API to do that. A dma_pool is like a kmem_cache, but it uses dma_alloc_coherent(), not __get_free_pages(). Also, it understands common hardware constraints for alignment, like queue heads needing to be aligned on N byte boundaries.

Create a dma_pool like this:

```c
struct dma_pool *pool;

pool = dma_pool_create(name, dev, size, align, boundary);
```

The “name” is for diagnostics (like a kmem_cache name); dev and size are as above. The device’s hardware alignment requirement for this type of data is “align” (which is expressed in bytes, and must be a power of two). If your device has no boundary crossing restrictions, pass 0 for boundary; passing 4096 says memory allocated from this pool must not cross 4KByte boundaries (but at that time it may be better to use dma_alloc_coherent() directly instead).

Allocate memory from a DMA pool like this:

```c
cpu_addr = dma_pool_alloc(pool, flags, &dma_handle);
```

flags are GFP_KERNEL if blocking is permitted (not in_interrupt nor holding SMP locks), GFP_ATOMIC otherwise. Like dma_alloc_coherent(), this returns two values, cpu_addr and dma_handle.

Free memory that was allocated from a dma_pool like this:

```c
dma_pool_free(pool, cpu_addr, dma_handle);
```

where pool is what you passed to [`dma_pool_alloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_alloc "dma_pool_alloc"), and cpu_addr and dma_handle are the values [`dma_pool_alloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_alloc "dma_pool_alloc") returned. This function may be called in interrupt context.

Destroy a dma_pool by calling:

```c
dma_pool_destroy(pool);
```

Make sure you’ve called [`dma_pool_free()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_free "dma_pool_free") for all memory allocated from a pool before you destroy the pool. This function may not be called in interrupt context.

## DMA Direction[](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html#dma-direction "Permalink to this headline")

The interfaces described in subsequent portions of this document take a DMA direction argument, which is an integer and takes on one of the following values:

```c
DMA_BIDIRECTIONAL
DMA_TO_DEVICE
DMA_FROM_DEVICE
DMA_NONE
```

You should provide the exact DMA direction if you know it.

DMA_TO_DEVICE means “from main memory to the device” DMA_FROM_DEVICE means “from the device to main memory” It is the direction in which the data moves during the DMA transfer.

You are _strongly_ encouraged to specify this as precisely as you possibly can.

If you absolutely cannot know the direction of the DMA transfer, specify DMA_BIDIRECTIONAL. It means that the DMA can go in either direction. The platform guarantees that you may legally specify this, and that it will work, but this may be at the cost of performance for example.

The value DMA_NONE is to be used for debugging. One can hold this in a data structure before you come to know the precise direction, and this will help catch cases where your direction tracking logic has failed to set things up properly.

Another advantage of specifying this value precisely (outside of potential platform-specific optimizations of such) is for debugging. Some platforms actually have a write permission boolean which DMA mappings can be marked with, much like page protections in the user program address space. Such platforms can and do report errors in the kernel logs when the DMA controller hardware detects violation of the permission setting.

Only streaming mappings specify a direction, consistent mappings implicitly have a direction attribute setting of DMA_BIDIRECTIONAL.

The SCSI subsystem tells you the direction to use in the ‘sc_data_direction’ member of the SCSI command your driver is working on.

For Networking drivers, it’s a rather simple affair. For transmit packets, map/unmap them with the DMA_TO_DEVICE direction specifier. For receive packets, just the opposite, map/unmap them with the DMA_FROM_DEVICE direction specifier.

## Using Streaming DMA mappings[](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html#using-streaming-dma-mappings "Permalink to this headline")

The streaming DMA mapping routines **can be called from interrupt context**. There are two versions of each map/unmap, one which will map/unmap a single memory region, and one which will map/unmap a **scatterlist**.

To map a single region, you do:
```c
struct device *dev = &my_dev->dev;
dma_addr_t dma_handle;
void *addr = buffer->ptr;
size_t size = buffer->len;

dma_handle = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle)) {
        /*
         * reduce current DMA mapping usage,
         * delay and try again later or
         * reset driver.
         */
        goto map_error_handling;
}
```

and to unmap it:
```c
dma_unmap_single(dev, dma_handle, size, direction);
```

You should call dma_mapping_error() as dma_map_single() could fail and return error. Doing so will ensure that the mapping code will work correctly on all DMA implementations without any dependency on the specifics of the underlying implementation. Using the returned address without checking for errors could result in failures ranging from panics to silent data corruption. The same applies to dma_map_page() as well.

You should call dma_unmap_single() when the DMA activity is finished, e.g., from the interrupt which told you that the DMA transfer is done. 在DMA完成中断里调用unmap.

直接使用buffer->ptr的坏处是不能用HIGHMEM区间. 用`dma_map_page()`可以使用HIGHMEM内存:  
Using CPU pointers like this for single mappings has a disadvantage: you cannot reference HIGHMEM memory in this way. Thus, there is a map/unmap interface pair akin to dma_{map,unmap}_single(). These interfaces deal with page/offset pairs instead of CPU pointers. Specifically:
```c
struct device *dev = &my_dev->dev;
dma_addr_t dma_handle;
struct page *page = buffer->page;
unsigned long offset = buffer->offset;
size_t size = buffer->len;

dma_handle = dma_map_page(dev, page, offset, size, direction);
if (dma_mapping_error(dev, dma_handle)) {
        /*
         * reduce current DMA mapping usage,
         * delay and try again later or
         * reset driver.
         */
        goto map_error_handling;
}

...

dma_unmap_page(dev, dma_handle, size, direction);
```
Here, “offset” means byte offset within the given page.

You should call dma_mapping_error() as dma_map_page() could fail and return error as outlined under the dma_map_single() discussion.

You should call dma_unmap_page() when the DMA activity is finished, e.g., from the interrupt which told you that the DMA transfer is done.

With scatterlists, you map a region gathered from several regions by:
这里的`sg`是`scatter-gather`的意思, 对应`single`
```c
int i, count = dma_map_sg(dev, sglist, nents, direction);
struct scatterlist *sg;

for_each_sg(sglist, sg, count, i) {
        hw_address[i] = sg_dma_address(sg);
        hw_len[i] = sg_dma_len(sg);
}
```
where nents is the number of entries in the sglist.

The implementation is free to merge several consecutive sglist entries into one (e.g. if DMA mapping is done with PAGE_SIZE granularity, any consecutive sglist entries can be merged into one provided the first one ends and the second one starts on a page boundary - in fact this is a huge advantage for cards which either cannot do scatter-gather or have very limited number of scatter-gather entries) and returns the actual number of sg entries it mapped them to. On failure 0 is returned.

Then you should loop count times (note: this can be less than nents times) and use sg_dma_address() and sg_dma_len() macros where you previously accessed sg->address and sg->length as shown above.

To unmap a scatterlist, just call:
```c
dma_unmap_sg(dev, sglist, nents, direction);
```
Again, make sure DMA activity has already finished.

> The ‘nents’ argument to the dma_unmap_sg call must be the _same_ one you passed into the dma_map_sg call, it should _NOT_ be the ‘count’ value _returned_ from the dma_map_sg call.

Every dma_map_{single,sg}() call should have its dma_unmap_{single,sg}() counterpart, because the DMA address space is a shared resource and you could render the machine unusable by consuming all DMA addresses.

If you need to use the same streaming DMA region multiple times and touch the data in between the DMA transfers, the buffer needs to be synced properly in order for the CPU and device to see the most up-to-date and correct copy of the DMA buffer.

DMA传输的时候, CPU也想改这个buffer的data, 需要用下面的API:
So, firstly, just map it with dma_map_{single,sg}(), and after each DMA transfer call either:
```c
dma_sync_single_for_cpu(dev, dma_handle, size, direction);
```
or:
```c
dma_sync_sg_for_cpu(dev, sglist, nents, direction);
```
as appropriate.  
> The ‘nents’ argument to `dma_sync_sg_for_cpu()` and `dma_sync_sg_for_device()` must be the same passed to `dma_map_sg()`. It is _NOT_ the count returned by `dma_map_sg()`.

After the last DMA transfer call one of the DMA unmap routines `dma_unmap_{single,sg}()`. If you don’t touch the data from the first `dma_map_*()` call till `dma_unmap_*()`, then you don’t have to call the `dma_sync_*()` routines at all.

Here is pseudo code which shows a situation in which you would need to use the dma_sync_*() interfaces:
```c
my_card_setup_receive_buffer(struct my_card *cp, char *buffer, int len)
{
        dma_addr_t mapping;

        mapping = dma_map_single(cp->dev, buffer, len, DMA_FROM_DEVICE);
        if (dma_mapping_error(cp->dev, mapping)) {
                /*
                 * reduce current DMA mapping usage,
                 * delay and try again later or
                 * reset driver.
                 */
                goto map_error_handling;
        }

        cp->rx_buf = buffer;
        cp->rx_len = len;
        cp->rx_dma = mapping;

        give_rx_buf_to_card(cp);
}

...

my_card_interrupt_handler(int irq, void *devid, struct pt_regs *regs)
{
        struct my_card *cp = devid;

        ...
        if (read_card_status(cp) == RX_BUF_TRANSFERRED) {
                struct my_card_header *hp;

                /* Examine the header to see if we wish
                 * to accept the data.  But synchronize
                 * the DMA transfer with the CPU first
                 * so that we see updated contents.
                 */
                dma_sync_single_for_cpu(&cp->dev, cp->rx_dma,
                                        cp->rx_len,
                                        DMA_FROM_DEVICE);

                /* Now it is safe to examine the buffer. */
                hp = (struct my_card_header *) cp->rx_buf;
                if (header_is_ok(hp)) {
                        dma_unmap_single(&cp->dev, cp->rx_dma, cp->rx_len,
                                         DMA_FROM_DEVICE);
                        pass_to_upper_layers(cp->rx_buf);
                        make_and_setup_new_rx_buf(cp);
                } else {
                        /* CPU should not write to
                         * DMA_FROM_DEVICE-mapped area,
                         * so dma_sync_single_for_device() is
                         * not needed here. It would be required
                         * for DMA_BIDIRECTIONAL mapping if
                         * the memory was modified.
                         */
                        give_rx_buf_to_card(cp);
                }
        }
}
```
Drivers converted fully to this interface should not use virt_to_bus() any longer, nor should they use bus_to_virt(). Some drivers have to be changed a little bit, because there is no longer an equivalent to bus_to_virt() in the dynamic DMA mapping scheme - you have to always store the DMA addresses returned by the dma_alloc_coherent(), [`dma_pool_alloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_alloc "dma_pool_alloc"), and dma_map_single() calls (dma_map_sg() stores them in the scatterlist itself if the platform supports dynamic DMA mapping in hardware) in your driver structures and/or in the card registers.

All drivers should be using these interfaces with no exceptions. It is planned to completely remove virt_to_bus() and bus_to_virt() as they are entirely deprecated. Some ports already do not provide these as it is impossible to correctly support them.

注意, 驱动不要再用`virt_to_bus()` and `bus_to_virt()`, 以后kernel就不维护这两个API了.

## Handling Errors[](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html#handling-errors "Permalink to this headline")

DMA address space is limited on some architectures and an allocation failure can be determined by:

*   checking if dma_alloc_coherent() returns NULL or dma_map_sg returns 0
*   checking the dma_addr_t returned from dma_map_single() and dma_map_page() by using dma_mapping_error():
```c
dma_addr_t dma_handle;
dma_handle = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle)) {
        /*
         * reduce current DMA mapping usage,
         * delay and try again later or
         * reset driver.
         */
        goto map_error_handling;
}
```
*   unmap pages that are already mapped, when mapping error occurs in the middle of a multiple page mapping attempt. These example are applicable to dma_map_page() as well.

Example 1:
```c
dma_addr_t dma_handle1;
dma_addr_t dma_handle2;

dma_handle1 = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle1)) {
        /*
         * reduce current DMA mapping usage,
         * delay and try again later or
         * reset driver.
         */
        goto map_error_handling1;
}
dma_handle2 = dma_map_single(dev, addr, size, direction);
if (dma_mapping_error(dev, dma_handle2)) {
        /*
         * reduce current DMA mapping usage,
         * delay and try again later or
         * reset driver.
         */
        goto map_error_handling2;
}

...

map_error_handling2:
        dma_unmap_single(dma_handle1);
map_error_handling1:
```

Example 2:
```c
/*
 * if buffers are allocated in a loop, unmap all mapped buffers when
 * mapping error is detected in the middle
 */

dma_addr_t dma_addr;
dma_addr_t array[DMA_BUFFERS];
int save_index = 0;

for (i = 0; i < DMA_BUFFERS; i++) {

        ...

        dma_addr = dma_map_single(dev, addr, size, direction);
        if (dma_mapping_error(dev, dma_addr)) {
                /*
                 * reduce current DMA mapping usage,
                 * delay and try again later or
                 * reset driver.
                 */
                goto map_error_handling;
        }
        array[i].dma_addr = dma_addr;
        save_index++;
}

...

map_error_handling:

for (i = 0; i < save_index; i++) {

        ...

        dma_unmap_single(array[i].dma_addr);
}
```
Networking drivers must call dev_kfree_skb() to free the socket buffer and return NETDEV_TX_OK if the DMA mapping fails on the transmit hook (ndo_start_xmit). This means that the socket buffer is just dropped in the failure case.

SCSI drivers must return SCSI_MLQUEUE_HOST_BUSY if the DMA mapping fails in the queuecommand hook. This means that the SCSI subsystem passes the command to the driver again later.

## Optimizing Unmap State Space Consumption[](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html#optimizing-unmap-state-space-consumption "Permalink to this headline")

On many platforms, `dma_unmap_{single,page}()` is simply a nop. Therefore, keeping track of the mapping address and length is a waste of space. Instead of filling your drivers up with ifdefs and the like to “work around” this (which would defeat the whole purpose of a portable API) the following facilities are provided.

Actually, instead of describing the macros one by one, we’ll transform some example code.

1.  Use `DEFINE_DMA_UNMAP_{ADDR,LEN}` in state saving structures. Example, before:

    ```c
    struct ring_state {
            struct sk_buff *skb;
            dma_addr_t mapping;
            __u32 len;
    };
    ```

    after:

    ```c
    struct ring_state {
            struct sk_buff *skb;
            DEFINE_DMA_UNMAP_ADDR(mapping);
            DEFINE_DMA_UNMAP_LEN(len);
    };


2.  Use dma_unmap_{addr,len}_set() to set these values. Example, before:

    ```c
    ringp->mapping = FOO;
    ringp->len = BAR;
    ```

    after:

    ```c
    dma_unmap_addr_set(ringp, mapping, FOO);
    dma_unmap_len_set(ringp, len, BAR);
    ```

3.  Use dma_unmap_{addr,len}() to access these values. Example, before:

    ```c
    dma_unmap_single(dev, ringp->mapping, ringp->len,
                     DMA_FROM_DEVICE);
    ```

    after:

    ```
    dma_unmap_single(dev,
                     dma_unmap_addr(ringp, mapping),
                     dma_unmap_len(ringp, len),
                     DMA_FROM_DEVICE);
    ```

It really should be self-explanatory. We treat the ADDR and LEN separately, because it is possible for an implementation to only need the address in order to perform the unmap operation.

## Platform Issues[](https://www.kernel.org/doc/html/latest/core-api/dma-api-howto.html#platform-issues "Permalink to this headline")

If you are just writing drivers for Linux and do not maintain an architecture port for the kernel, you can safely skip down to “Closing”.

1.  Struct scatterlist requirements.

    You need to enable CONFIG_NEED_SG_DMA_LENGTH if the architecture supports IOMMUs (including software IOMMU).

2.  ARCH_DMA_MINALIGN

    Architectures must ensure that kmalloc’ed buffer is DMA-safe. Drivers and subsystems depend on it. If an architecture isn’t fully DMA-coherent (i.e. hardware doesn’t ensure that data in the CPU cache is identical to data in main memory), ARCH_DMA_MINALIGN must be set so that the memory allocator makes sure that kmalloc’ed buffer doesn’t share a cache line with the others. See arch/arm/include/asm/cache.h as an example.

    Note that ARCH_DMA_MINALIGN is about DMA memory alignment constraints. You don’t need to worry about the architecture data alignment constraints (e.g. the alignment constraints about 64-bit objects).
    
# Dynamic DMA mapping using the generic device
## Part I - dma_API[](https://www.kernel.org/doc/html/latest/core-api/dma-api.html#part-i-dma-api "Permalink to this headline")

To get the dma_API, you must #include <linux/dma-mapping.h>. This provides dma_addr_t and the interfaces described below.

A dma_addr_t can hold any valid DMA address for the platform. It can be given to a device to use as a DMA source or target. A CPU cannot reference a dma_addr_t directly because there may be translation between its physical address space and the DMA address space.

## Part Ia - Using large DMA-coherent buffers
```
void *
dma_alloc_coherent(struct device *dev, size_t size,
                   dma_addr_t *dma_handle, gfp_t flag)
```
返回的`void *`就是新申请buffer的CPU的虚拟地址, 而`dma_addr_t *dma_handle`是DMA地址, 即可以给device用的bus地址.  
Consistent memory is memory for which a write by either the device or the processor can immediately be read by the processor or device without having to worry about caching effects. (You may however need to make sure to flush the processor’s write buffers before telling devices to read that memory.)

This routine allocates a region of <size> bytes of consistent memory.

It returns a pointer to the allocated region (in the processor’s virtual address space) or NULL if the allocation failed.

It also returns a <dma_handle> which may be cast to an unsigned integer the same width as the bus and given to the device as the DMA address base of the region.

Note: consistent memory can be expensive on some platforms, and the minimum allocation length may be as big as a page, so you should consolidate your requests for consistent memory as much as possible. The simplest way to do that is to use the dma_pool calls (see below).

The flag parameter (dma_alloc_coherent() only) allows the caller to specify the `GFP_` flags (see [`kmalloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc "kmalloc")) for the allocation (the implementation may choose to ignore flags that affect the location of the returned memory, like GFP_DMA).
```c
void
dma_free_coherent(struct device *dev, size_t size, void *cpu_addr,
                  dma_addr_t dma_handle)
```
Free a region of consistent memory you previously allocated. dev, size and dma_handle must all be the same as those passed into dma_alloc_coherent(). cpu_addr must be the virtual address returned by the dma_alloc_coherent().

Note that unlike their sibling allocation calls, these routines may only be called with IRQs enabled.

## Part Ib - Using small DMA-coherent buffers
To get this part of the dma_API, you must #include <linux/dmapool.h>

Many drivers need lots of small DMA-coherent memory regions for DMA descriptors or I/O buffers. Rather than allocating in units of a page or more using dma_alloc_coherent(), you can use DMA pools. These work much like a struct kmem_cache, except that they use the DMA-coherent allocator, not __get_free_pages(). Also, they understand common hardware constraints for alignment, like queue heads needing to be aligned on N-byte boundaries.

```c
struct dma_pool *
dma_pool_create(const char *name, struct device *dev,
                size_t size, size_t align, size_t alloc);
```
[`dma_pool_create()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_create "dma_pool_create") initializes a pool of DMA-coherent buffers for use with a given device. It must be called in a context which can sleep.

The “name” is for diagnostics (like a struct kmem_cache name); dev and size are like what you’d pass to dma_alloc_coherent(). The device’s hardware alignment requirement for this type of data is “align” (which is expressed in bytes, and must be a power of two). If your device has no boundary crossing restrictions, pass 0 for alloc; passing 4096 says memory allocated from this pool must not cross 4KByte boundaries.

```c
void *
dma_pool_zalloc(struct dma_pool *pool, gfp_t mem_flags,
                dma_addr_t *handle)
```

Wraps [`dma_pool_alloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_alloc "dma_pool_alloc") and also zeroes the returned memory if the allocation attempt succeeded.

```c
void *
dma_pool_alloc(struct dma_pool *pool, gfp_t gfp_flags,
               dma_addr_t *dma_handle);
```
This allocates memory from the pool; the returned memory will meet the size and alignment requirements specified at creation time. Pass GFP_ATOMIC to prevent blocking, or if it’s permitted (not in_interrupt, not holding SMP locks), pass GFP_KERNEL to allow blocking. Like dma_alloc_coherent(), this returns two values: an address usable by the CPU, and the DMA address usable by the pool’s device.

```c
void
dma_pool_free(struct dma_pool *pool, void *vaddr,
              dma_addr_t addr);
```
This puts memory back into the pool. The pool is what was passed to [`dma_pool_alloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_alloc "dma_pool_alloc"); the CPU (vaddr) and DMA addresses are what were returned when that routine allocated the memory being freed.

```c
void
dma_pool_destroy(struct dma_pool *pool);
```
[`dma_pool_destroy()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.dma_pool_destroy "dma_pool_destroy") frees the resources of the pool. It must be called in a context which can sleep. Make sure you’ve freed all allocated memory back to the pool before you destroy it.

## Part Ic - DMA addressing limitations
主要是介绍
```c
int
dma_set_mask_and_coherent(struct device *dev, u64 mask)

int
dma_set_mask(struct device *dev, u64 mask)

int
dma_set_coherent_mask(struct device *dev, u64 mask)

u64
dma_get_required_mask(struct device *dev)

size_t
dma_max_mapping_size(struct device *dev);

bool
dma_need_sync(struct device *dev, dma_addr_t dma_addr);

unsigned long
dma_get_merge_boundary(struct device *dev);

```

## Part Id - Streaming DMA mappings
```c
dma_addr_t
dma_map_single(struct device *dev, void *cpu_addr, size_t size,
               enum dma_data_direction direction)
dma_addr_t
dma_map_page(struct device *dev, struct page *page,
             unsigned long offset, size_t size,
             enum dma_data_direction direction)

dma_addr_t
dma_map_resource(struct device *dev, phys_addr_t phys_addr, size_t size,
                 enum dma_data_direction dir, unsigned long attrs)

int
dma_map_sg(struct device *dev, struct scatterlist *sg,
           int nents, enum dma_data_direction direction)
```
用这个API需要注意:
>Not all memory regions in a machine can be mapped by this API. Further, contiguous kernel virtual space may not be contiguous as physical memory. Since this API does not provide any scatter/gather capability, it will fail if the user tries to map a non-physically contiguous piece of memory. For this reason, memory to be mapped by this API should be obtained from sources which guarantee it to be physically contiguous (like kmalloc).

>Further, the DMA address of the memory must be within the dma_mask of the device (the dma_mask is a bit mask of the addressable region for the device, i.e., if the DMA address of the memory ANDed with the dma_mask is still equal to the DMA address, then the device can perform DMA to the memory). To ensure that the memory allocated by kmalloc is within the dma_mask, the driver may specify various platform-dependent flags to restrict the DMA address range of the allocation (e.g., on x86, GFP_DMA guarantees to be within the first 16MB of available DMA addresses, as required by ISA devices).

>Note also that the above constraints on physical contiguity and dma_mask may not apply if the platform has an IOMMU (a device which maps an I/O DMA address to a physical memory address). However, to be portable, device driver writers may _not_ assume that such an IOMMU exists.

另外, memory coherency是以cacheline为单位的, 但一般编译的时候不知道. 那么一般的做法是使用page对齐的方式, 因为page对齐一定也是cachline对齐的.  
>Memory coherency operates at a granularity called the cache line width. In order for memory mapped by this API to operate correctly, the mapped region must begin exactly on a cache line boundary and end exactly on one (to prevent two separately mapped regions from sharing a single cache line). Since the cache line size may not be known at compile time, the API will not enforce this requirement. Therefore, it is recommended that driver writers who don’t take special care to determine the cache line size at run time only map virtual regions that begin and end on page boundaries (which are guaranteed also to be cache line boundaries).

>DMA_TO_DEVICE synchronisation must be done after the last modification of the memory region by the software and before it is handed off to the device. Once this primitive is used, memory covered by this primitive should be treated as read-only by the device. If the device may write to it at any point, it should be DMA_BIDIRECTIONAL (see below).

>DMA_FROM_DEVICE synchronisation must be done before the driver accesses data that may be changed by the device. This memory should be treated as read-only by the driver. If the driver needs to write to it at any point, it should be DMA_BIDIRECTIONAL (see below).

>DMA_BIDIRECTIONAL requires special handling: it means that the driver isn’t sure if the memory was modified before being handed off to the device and also isn’t sure if the device will also modify it. Thus, you must always sync bidirectional memory twice: once before the memory is handed off to the device (to make sure all memory changes are flushed from the processor) and once before the data may be accessed after being used by the device (to make sure any processor cache lines are updated with data that the device may have changed).

在DMA传输前后, 必须要代码显式调用下面的sync函数来保证数据一致性:
```c
void
dma_sync_single_for_cpu(struct device *dev, dma_addr_t dma_handle,
                        size_t size,
                        enum dma_data_direction direction)

void
dma_sync_single_for_device(struct device *dev, dma_addr_t dma_handle,
                           size_t size,
                           enum dma_data_direction direction)

void
dma_sync_sg_for_cpu(struct device *dev, struct scatterlist *sg,
                    int nents,
                    enum dma_data_direction direction)

void
dma_sync_sg_for_device(struct device *dev, struct scatterlist *sg,
                       int nents,
                       enum dma_data_direction direction)
```
You must do this:

*   Before reading values that have been written by DMA from the device (use the DMA_FROM_DEVICE direction)
*   After writing values that will be written to the device using DMA (use the DMA_TO_DEVICE) direction
*   before _and_ after handing memory to the device if the memory is DMA_BIDIRECTIONAL

## Part II - Non-coherent DMA allocations
These APIs allow to allocate pages that are guaranteed to be DMA addressable by the passed in device, but which need explicit management of memory ownership for the kernel vs the device.

If you don’t understand how cache line coherency works between a processor and an I/O device, you should not be using this part of the API.

```c
struct page *
dma_alloc_pages(struct device *dev, size_t size, dma_addr_t *dma_handle,
                enum dma_data_direction dir, gfp_t gfp)
```
This routine allocates a region of <size> bytes of non-coherent memory. It returns a pointer to first struct page for the region, or NULL if the allocation failed. The resulting struct page can be used for everything a struct page is suitable for.

It also returns a <dma_handle> which may be cast to an unsigned integer the same width as the bus and given to the device as the DMA address base of the region.

The dir parameter specified if data is read and/or written by the device, see dma_map_single() for details.

The gfp parameter allows the caller to specify the `GFP_` flags (see [`kmalloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc "kmalloc")) for the allocation, but rejects flags used to specify a memory zone such as GFP_DMA or GFP_HIGHMEM.

Before giving the memory to the device, dma_sync_single_for_device() needs to be called, and before reading memory written by the device, dma_sync_single_for_cpu(), just like for streaming DMA mappings that are reused.

```c
int
dma_mmap_pages(struct device *dev, struct vm_area_struct *vma,
               size_t size, struct page *page)
```
Map an allocation returned from dma_alloc_pages() into a user address space. dev and size must be the same as those passed into dma_alloc_pages(). page must be the pointer returned by dma_alloc_pages().

```c
void *
dma_alloc_noncoherent(struct device *dev, size_t size,
                dma_addr_t *dma_handle, enum dma_data_direction dir,
                gfp_t gfp)
```
This routine is a convenient wrapper around dma_alloc_pages that returns the kernel virtual address for the allocated memory instead of the page structure.

```c
struct sg_table *
dma_alloc_noncontiguous(struct device *dev, size_t size,
                        enum dma_data_direction dir, gfp_t gfp,
                        unsigned long attrs);
```
This routine allocates <size> bytes of non-coherent and possibly non-contiguous memory. It returns a pointer to struct sg_table that describes the allocated and DMA mapped memory, or NULL if the allocation failed. The resulting memory can be used for struct page mapped into a scatterlist are suitable for.

The return sg_table is guaranteed to have 1 single DMA mapped segment as indicated by sgt->nents, but it might have multiple CPU side segments as indicated by sgt->orig_nents.

The dir parameter specified if data is read and/or written by the device, see dma_map_single() for details.

The gfp parameter allows the caller to specify the `GFP_` flags (see [`kmalloc()`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kmalloc "kmalloc")) for the allocation, but rejects flags used to specify a memory zone such as GFP_DMA or GFP_HIGHMEM.

The attrs argument must be either 0 or DMA_ATTR_ALLOC_SINGLE_PAGES.

Before giving the memory to the device, dma_sync_sgtable_for_device() needs to be called, and before reading memory written by the device, dma_sync_sgtable_for_cpu(), just like for streaming DMA mappings that are reused.

```c
void *
dma_vmap_noncontiguous(struct device *dev, size_t size,
        struct sg_table *sgt)
```
Return a contiguous kernel mapping for an allocation returned from dma_alloc_noncontiguous(). dev and size must be the same as those passed into dma_alloc_noncontiguous(). sgt must be the pointer returned by dma_alloc_noncontiguous().

```c
int
dma_mmap_noncontiguous(struct device *dev, struct vm_area_struct *vma,
                       size_t size, struct sg_table *sgt)
```
Map an allocation returned from dma_alloc_noncontiguous() into a user address space. dev and size must be the same as those passed into dma_alloc_noncontiguous(). sgt must be the pointer returned by dma_alloc_noncontiguous().
