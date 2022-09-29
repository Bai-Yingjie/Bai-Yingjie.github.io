- [什么是VFIO](#什么是vfio)
- [绑定驱动](#绑定驱动)
- [VFIO要解决的问题](#vfio要解决的问题)
  - [VFIO是个device驱动](#vfio是个device驱动)

# 什么是VFIO
* 用户态驱动框架
* Virtual Function IO
* 基于IOMMU的DMA和中断隔离
* 完整的device的访问(MMIO, IO port, PCI config)
# 绑定驱动
比如在跑dpdk之前, 先要绑定用户态驱动框架, 比如
```
sudo usertools/dpdk-devbind.py --status 
sudo usertools/dpdk-devbind.py -b uio_pci_generic 04:00.0 
```
和这里的uio框架并列的, 能被status show出来的driver, 就有vfio: vfio-pci. -- 虽然我没用过...
通过VFIO, 可以:
* 给device assign VFIO driver(bind), 或者unbind
* 有更好的安全模式
* device隔离
* 移植性好

# VFIO要解决的问题
PCI-assign的问题:
* 资源的访问和secure boot不兼容
* IOMMU的颗粒度不满足
* device的所有权模型不好
* PCI only
## VFIO是个device驱动
* 支持模块化的设备驱动后端
* vfio-pci可以bind到非桥类的pci device
* pci-stub是个"桩"driver, 无法访问
* 被VFIO接管的device无法被其他host driver使用

