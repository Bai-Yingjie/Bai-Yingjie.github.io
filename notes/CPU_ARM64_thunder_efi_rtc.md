- [现象](#现象)
- [从rtc-efi.ko说开去](#从rtc-efiko说开去)

# 现象
很简单, RTC时间不对, 每次重启就回到1970年.

# 从rtc-efi.ko说开去
还记得最开始fedora启动的时候, 就是这个ko老打印错误, 当时只是简单的删掉它了, 没有多想.  
但现在回过头来再看RTC问题的时候, 才发现这个ko就是key.

这个ko的代码很简单, 在`drivers/rtc/rtc-efi.c`  
两个重要的函数, `efi_read_time()`和`efi_set_time()`

既然时间不对, 我们就先来看`efi_read_time()`
```c
static int efi_read_time(struct device *dev, struct rtc_time *tm)
{
	efi_status_t status;
	efi_time_t eft;
	efi_time_cap_t cap;

	status = efi.get_time(&eft, &cap);

	if (status != EFI_SUCCESS) {
		/* should never happen */
		dev_err(dev, "can't read time\n");
		return -EINVAL;
	}

	if (!convert_from_efi_time(&eft, tm))
		return -EIO;

	return rtc_valid_tm(tm);
}
```

这里面有个`efi.get_time(&eft, &cap)`, 应该就是获取时间了, 而正如这个ko的名字rtc-efi所暗示的, 它不是直接调用linux驱动来读时间, 而是调efi的run time service.

这个`efi.get_time`在哪呢? `include/linux/efi.h`里面说的很清楚, efi所有的runtime的东东都在这个结构体里面
```c
/*
 * All runtime access to EFI goes through this structure:
 */
extern struct efi {
	efi_system_table_t *systab;	/* EFI system table */
	unsigned int runtime_version;	/* Runtime services version */
	unsigned long mps;		/* MPS table */
	unsigned long acpi;		/* ACPI table  (IA64 ext 0.71) */
	unsigned long acpi20;		/* ACPI table  (ACPI 2.0) */
	unsigned long smbios;		/* SMBIOS table (32 bit entry point) */
	unsigned long smbios3;		/* SMBIOS table (64 bit entry point) */
	unsigned long sal_systab;	/* SAL system table */
	unsigned long boot_info;	/* boot info table */
	unsigned long hcdp;		/* HCDP table */
	unsigned long uga;		/* UGA table */
	unsigned long uv_systab;	/* UV system table */
	unsigned long fw_vendor;	/* fw_vendor */
	unsigned long runtime;		/* runtime table */
	unsigned long config_table;	/* config tables */
	unsigned long esrt;		/* ESRT table */
efi_get_time_t *get_time;
	efi_set_time_t *set_time;
	efi_get_wakeup_time_t *get_wakeup_time;
	efi_set_wakeup_time_t *set_wakeup_time;
	efi_get_variable_t *get_variable;
	efi_get_next_variable_t *get_next_variable;
	efi_set_variable_t *set_variable;
	efi_set_variable_nonblocking_t *set_variable_nonblocking;
	efi_query_variable_info_t *query_variable_info;
	efi_update_capsule_t *update_capsule;
	efi_query_capsule_caps_t *query_capsule_caps;
	efi_get_next_high_mono_count_t *get_next_high_mono_count;
	efi_reset_system_t *reset_system;
	efi_set_virtual_address_map_t *set_virtual_address_map;
	struct efi_memory_map *memmap;
	unsigned long flags;
} efi;
```

# 从UEFI看起
实际操作硬件的地方在uefi, 在`ArmPlatformPkg/ThunderPkg/Drivers/Ds1337RtcDxe/Ds1337RtcDxe.c`

在Ds1337RtcDxeInitialize()这个初始化函数中, 就挂了几个函数
```c
EFI_STATUS
EFIAPI
Ds1337RtcDxeInitialize (
  IN EFI_HANDLE         ImageHandle,
  IN EFI_SYSTEM_TABLE   *SystemTable
  )
{
    ...
    SystemTable->RuntimeServices->GetTime       = GetTime;
    SystemTable->RuntimeServices->SetTime       = SetTime;
    SystemTable->RuntimeServices->GetWakeupTime = GetWakeupTime;
    SystemTable->RuntimeServices->SetWakeupTime = SetWakeupTime;
}
```
真正去读RTC的是里面的GetTime函数, 这个函数调用`ThunderxTwsiRead8Bytes()`去最终读取RTC时间.

# kernel调用efi的service机制是什么?
我猜这里面的关键就应该是这个SystemTable了, 但还有些需要考虑的, 比如uefi传递这个指针, 是虚拟地址吗? linux应该知道这个映射吗? 或者直接使用物理地址? 

还是先从kernel的调用的地方看起, 回到前面的`efi_read_time()`
这个函数调用了
```c
status = efi.get_time(&eft, &cap)
```
这里面的efi看起来是一个全局的结构体变量, 而get_time是哪个?

在`drivers/firmware/efi/runtime-wrappers.c`中, 它被初始化为`virt_efi_get_time`
```c
void efi_native_runtime_setup(void)
{
	efi.get_time = virt_efi_get_time;
	efi.set_time = virt_efi_set_time;
	efi.get_wakeup_time = virt_efi_get_wakeup_time;
	efi.set_wakeup_time = virt_efi_set_wakeup_time;
	efi.get_variable = virt_efi_get_variable;
	efi.get_next_variable = virt_efi_get_next_variable;
	efi.set_variable = virt_efi_set_variable;
	efi.set_variable_nonblocking = virt_efi_set_variable_nonblocking;
	efi.get_next_high_mono_count = virt_efi_get_next_high_mono_count;
	efi.reset_system = virt_efi_reset_system;
	efi.query_variable_info = virt_efi_query_variable_info;
	efi.update_capsule = virt_efi_update_capsule;
	efi.query_capsule_caps = virt_efi_query_capsule_caps;
}
```
最后调用`status = efi_call_virt(get_time, tm, tc);`
这里面的`efi_call_virt`是个宏, 在`arch/arm64/include/asm/efi.h`中定义
```c
#define efi_call_virt(f, ...)		\
({                                  \
    efi_##f##_t *__f;				\
    efi_status_t __s;				\
                                    \
    kernel_neon_begin();			\
    efi_virtmap_load();				\
    __f = efi.systab->runtime->f;	\
    __s = __f(__VA_ARGS__);			\
    efi_virtmap_unload();			\
    kernel_neon_end();				\
    __s;							\
})
```
我们能够得出一下几点:

* 要想调用efi的函数, 必须先load efi的内存映射; 此时使用`efi_mm, arch/arm64/kernel/efi.c`

* 实际调用的是`efi.systab->runtime->f`, 很显然, 这个f的地址应该是uefi填的, 是在uefi内存映射下的地址

* 看起来linux调用efi的service并没有改变CPU的运行级别

* neon是适用于ARM Cortex-A系列处理器的一种128位SIMD(Single Instruction, Multiple Data,单指令、多数据)扩展结构
