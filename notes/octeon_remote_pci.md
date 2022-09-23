host/remote-lib/octeon-remote-pci.c阅读

# 总体pcie操作集, 提供上层使用的回调
```c
int octeon_remote_pci(octeon_remote_funcs_t *remote_funcs)
{
    remote_funcs->open = pci_open;
    remote_funcs->close = pci_close;
    remote_funcs->read_csr = pci_read_csr;
    remote_funcs->write_csr = pci_write_csr;
    remote_funcs->read_csr32 = pci_read_csr32;
    remote_funcs->write_csr32 = pci_write_csr32;
    remote_funcs->read_mem = pci_read_mem;
    remote_funcs->write_mem = pci_write_mem;
    remote_funcs->get_model = pci_get_model;
    remote_funcs->start_cores = pci_start_cores;
    remote_funcs->stop_cores = pci_stop_cores;
    remote_funcs->get_running_cores = pci_get_running_cores;
    remote_funcs->get_num_cores = pci_get_num_cores;
    remote_funcs->get_core_state = pci_get_core_state;
    remote_funcs->set_core_state = pci_set_core_state;
    remote_funcs->reset = pci_reset;
    remote_funcs->get_sample = pci_get_sample;

    if (getenv("OCTEON_PCI_DEBUG"))
        remote_funcs->debug++;

    return 0;
}
```

## pci_open
```c
int pci_open(const char *remote_spec)
  pci_get_device(int device)
    in = fopen("/proc/bus/pci/devices", "r");
    //这个文件里面每行都是个pcie设备
    while (!feof(in))
      //用fscanf读出相关信息
      fscanf(in, "%2x%2x %8x %x %Lx %Lx %Lx %Lx %Lx %Lx %Lx %Lx %Lx %Lx",省略)
      //找到对应的id, 得到br0和br1的地址和大小(物理地址)
      octeon_pci_bar0_address octeon_pci_bar1_address octeon_pci_bar0_size octeon_pci_bar1_size
      //检查是否打开了配置空间的master位, 注意写/proc/bus/pci/devices这个文件可以直接修改配置空间
  //使用/dev/mem来mmap64得到虚拟地址
  octeon_pci_bar0_ptr = octeon_remote_map(octeon_pci_bar0_address, octeon_pci_bar0_size, &bar0_cookie);
    mmap64(NULL, alength, PROT_READ|PROT_WRITE, MAP_SHARED, file_handle, physical_address & (int64_t)pagemask);
  //同样的, 获得bar1虚拟地址
  //这里是一些有关pcie变量的初始化?
  setup_globals()
```

## pci_read_csr
```c
uint64_t pci_read_csr(uint64_t physical_address)
  octeon2
    OCTEON_SLI_ADDR直接访问
    其他通过窗口寄存器(octeon_pci_bar0_win_rd_addr)转
  octeon3
    都通过octeon_pci_bar0_win_rd_addr转
```

## pci_write_mem
```c
void pci_write_mem(uint64_t physical_address, const void *buffer_ptr, int length)
    //原则上按照4M一块来写(还记得吗, bar1是16*4M的访问模式)
    //但这里有个问题, 就是physical_address应该是个任意地址, 而且length也应该是任意的
    //所以是这样访问, 把physical_address...length分成三大部分
    //不到4M的头+n*4M中间的+不到4M的尾
    char *buffer = (char*)buffer_ptr;
    uint32_t block_mask = (1<<22) - 1;
    uint64_t end_address = physical_address + length;


    /* We need to do writes in 4MB aligned chunks. This way we only need to
        mess with one BAR1 index register */
    do
    {
        void *ptr = octeon_pci_bar1_ptr + (physical_address & block_mask);
        if (end_address > (physical_address & ~(uint64_t)block_mask) + block_mask + 1)
            length = block_mask + 1 - (physical_address & block_mask);
        else
            length = end_address - physical_address;
        //只用到16个窗口的index0, 修改其基址
        pci_bar1_setup(physical_address);
        //根据地址对齐方式来拷贝
        fast_memcpy(ptr, buffer, length);
        buffer += length;
        physical_address += length;
    } while (physical_address < end_address);
```
