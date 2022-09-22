- [api](#api)
- [ioctl的宏是怎么定义的?](#ioctl的宏是怎么定义的)

# api
`components/driver/host/api/octeon_user.c`  
这里面都是对ioctl的封装, api包括:  
在octeon_user.h里面有详细说明  
最主要的发送  
```
octeon_send_request
```
检查发送后的情况
```
octeon_query_request
octeon_get_stats
```

还有pcie相关的接口:  
config空间
```
octeon_read_pcicfg_register
octeon_write_pcicfg_register
```
mem空间, 得到BAR0 BAR1的mapping地址
```
octeon_get_mapping_info
octeon_read32
octeon_write32
```

直接访问octeon芯片地址的接口
```
octeon_read_core_direct
octeon_write_core_direct
```

都是x86直接读写octeon, 以下两个和上面两个有什么区别?
```
octeon_read_core
octeon_write_core
```

# ioctl的宏是怎么定义的?
`components/driver/host/include/octeon_ioctl.h`
比如定义
```c
#define IOCTL_OCTEON_SEND_REQUEST   \
        _IOWR(OCTEON_MAGIC, OCTEON_SEND_REQUEST_CODE, octeon_soft_request_t)
```
这里面magic是
```c
#define OCTEON_MAGIC   0xC1
```
而__IOWR在内核树里面
```c
linux/include/uapi/asm-generic/ioctl.h

/* used to create numbers */
#define _IO(type,nr)        _IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)  _IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOW(type,nr,size)  _IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOWR(type,nr,size) _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOR_BAD(type,nr,size)  _IOC(_IOC_READ,(type),(nr),sizeof(size))
#define _IOW_BAD(type,nr,size)  _IOC(_IOC_WRITE,(type),(nr),sizeof(size))
#define _IOWR_BAD(type,nr,size) _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),sizeof(size))
#ifndef __KERNEL__
#define _IOC_TYPECHECK(t) (sizeof(t))
#endif
```
而IOC在
```c
#define _IOC(dir,type,nr,size) \
    (((dir)  << _IOC_DIRSHIFT) | \
     ((type) << _IOC_TYPESHIFT) | \
     ((nr)   << _IOC_NRSHIFT) | \
     ((size) << _IOC_SIZESHIFT))
```
dir是direction的意思

```c
#ifndef _IOC_NONE
# define _IOC_NONE  0U
#endif


#ifndef _IOC_WRITE
# define _IOC_WRITE 1U
#endif


#ifndef _IOC_READ
# define _IOC_READ  2U
#endif
```
还有一些宏可以在驱动里面用
```c
/* used to decode ioctl numbers.. */
#define _IOC_DIR(nr)        (((nr) >> _IOC_DIRSHIFT) & _IOC_DIRMASK)
#define _IOC_TYPE(nr)       (((nr) >> _IOC_TYPESHIFT) & _IOC_TYPEMASK)
#define _IOC_NR(nr)     (((nr) >> _IOC_NRSHIFT) & _IOC_NRMASK)
#define _IOC_SIZE(nr)       (((nr) >> _IOC_SIZESHIFT) & _IOC_SIZEMASK)
```
