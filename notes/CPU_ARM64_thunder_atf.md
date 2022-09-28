- [thunder的main, 不知道被哪里调用](#thunder的main-不知道被哪里调用)
- [atf能控制kernel看到那些pci设备?](#atf能控制kernel看到那些pci设备)
- [有对板类型的判断](#有对板类型的判断)
- [一些默认值](#一些默认值)
- [bl1 bl2](#bl1-bl2)
- [bl31](#bl31)

# thunder的main, 不知道被哪里调用
```c
atf/plat/thunder/bootstrap/main.c
    这里面有些初始化
    flash_init()
    下面是管SMMU的, Initialize thunder io view of non secure software
    init_thunder_io()
```

# atf能控制kernel看到那些pci设备?
```c
struct ecam_device{
    int ecam_id;
    int bus;
    int dev;
    int fun;
    int ns_visible;
    int nxt_fn;
    ecam_probe probe_fn;
    unsigned long probe_arg;
};

struct ecam_device devs0[] = {
    {0, 0, 1, 0, TRUE, -1, NULL, 0},  /* PCCBR_MRML */
    {0, 0, 2, 0, FALSE, -1, NULL, 0}, /* SMMU 0 */
    {0, 0, 3, 0, FALSE, -1, NULL, 0}, /* GIC */
    {0, 0, 4, 0, FALSE, -1, NULL, 0}, /* GTI */
    {0, 0, 6, 0, TRUE, -1, NULL, 0},  /* GPIO */
    {0, 0, 7, 0, FALSE, -1, NULL, 0}, /* MPI */
    {0, 0, 8, 0, FALSE, -1, NULL, 0}, /* MIO_PTP */
    {0, 0, 9, 0, FALSE, -1, NULL, 0}, /* RNM */
    {0, 0, 16, 0, TRUE, -1, NULL, 0}, /* USB 0 */
    {0, 0, 17, 0, TRUE, -1, NULL, 0}, /* USB 1 */
    {0, 1,  0, 0, TRUE, 0, ecam_probe_true, 0},     /* MRML */
    {0, 1,  0, 1, TRUE, 0, ecam_probe_true, 0},     /* RST */
    {0, 1,  0, 5, TRUE, 0, ecam_probe_true, 0},     /* OCX */
    {0, 1,  0, 12, TRUE, 0, ecam_probe_true, 0},    /* MIO_EMM */
    {0, 1,  0, 64, FALSE, -1, ecam_probe_false, 0}, /* UAA 0 */
    {0, 1,  0, 65, FALSE, -1, ecam_probe_false, 0}, /* UAA 1 */
    {0, 1 , 0, 72, TRUE, 0, ecam_probe_true, 0}, /* TWSI 0 */
    {0, 1 , 0, 73, TRUE, 0, ecam_probe_true, 0}, /* TWSI 1 */
    {0, 1 , 0, 74, TRUE, 0, ecam_probe_true, 0}, /* TWSI 2 */
    {0, 1 , 0, 75, TRUE, 0, ecam_probe_true, 0}, /* TWSI 3 */
    {0, 1 , 0, 76, TRUE, 0, ecam_probe_true, 0}, /* TWSI 4 */
    {0, 1 , 0, 77, TRUE, 0, ecam_probe_true, 0}, /* TWSI 5 */

    {0, 1 , 0, 80, TRUE, 0, ecam_probe_true, 0}, /* LMC 0 */
    {0, 1 , 0, 81, TRUE, 0, ecam_probe_true, 0}, /* LMC 1 */
    {0, 1 , 0, 82, TRUE, 0, ecam_probe_true, 0}, /* LMC 2 */
    {0, 1 , 0, 83, TRUE, 0, ecam_probe_true, 0}, /* LMC 3 */

    {0, 1 , 0, 112, TRUE, 0, ecam_probe_pem, 0}, /* PEM 0 */
    {0, 1 , 0, 113, TRUE, 0, ecam_probe_pem, 1}, /* PEM 1 */
    {0, 1 , 0, 114, TRUE, 0, ecam_probe_pem, 2}, /* PEM 2 */
    {0, 1 , 0, 115, TRUE, 0, ecam_probe_pem, 3}, /* PEM 3 */
    {0, 1 , 0, 116, TRUE, 0, ecam_probe_pem, 4}, /* PEM 4 */
    {0, 1 , 0, 117, TRUE, 0, ecam_probe_pem, 5}, /* PEM 5 */

    {0, 1,  0, 128, TRUE, 0, ecam_probe_bgx, 0}, /* BGX 0 */
    {0, 1,  0, 129, TRUE, 0, ecam_probe_bgx, 1}, /* BGX 1 */

    {0, 2,  0, 0, TRUE, -1, NULL, 0}, /* RAD */
    {0, 3,  0, 0, TRUE, -1, NULL, 0}, /* ZIP */
    {0, 4,  0, 0, TRUE, -1, NULL, 0}, /* HFA */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};

struct ecam_device devs1[] = {
    {1, 0,  1, 0, FALSE, -1, NULL, 0},            /* SMMU 1 */
    {1, 0,  4, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 0 */
    {1, 0,  5, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 1 */
    {1, 0,  6, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 2 */
    {1, 0,  7, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 3 */
    {1, 0,  8, 0, TRUE, -1, ecam_probe_sata, 3},  /* SATA 4 */
    {1, 0,  9, 0, TRUE, -1, ecam_probe_sata, 3},  /* SATA 5 */
    {1, 0,  10, 0, TRUE, -1, ecam_probe_sata, 3}, /* SATA 6 */
    {1, 0,  11, 0, TRUE, -1, ecam_probe_sata, 3}, /* SATA 7 */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};

struct ecam_device devs2[] = {
    {2, 0,  1, 0, FALSE, -1, NULL, 0},            /* SMMU 2 */
    {2, 0,  2, 0, TRUE, -1, NULL, 0},             /* PCCBR_NIC */
    {2, 0,  3, 0, TRUE, -1, NULL, 0},             /* TNS */
    {2, 1,  0, 0, TRUE, -1, NULL, 0},             /* NIC */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};

struct ecam_device devs3[] = {
    {3, 0,  1, 0, FALSE, -1, NULL, 0},            /* SMMU 3 */
    {3, 0,  4, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 8 */
    {3, 0,  5, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 9 */
    {3, 0,  6, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 10 */
    {3, 0,  7, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 11 */
    {3, 0,  8, 0, TRUE, -1, ecam_probe_sata, 7},  /* SATA 12 */
    {3, 0,  9, 0, TRUE, -1, ecam_probe_sata, 7},  /* SATA 13 */
    {3, 0,  10, 0, TRUE, -1, ecam_probe_sata, 7}, /* SATA 14 */
    {3, 0,  11, 0, TRUE, -1, ecam_probe_sata, 7}, /* SATA 15 */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};

struct ecam_device devs4[] = {
    {0, 0, 1, 0, TRUE, -1, NULL, 0},  /* PCCBR_MRML */
    {0, 0, 2, 0, FALSE, -1, NULL, 0}, /* SMMU 0 */
    {0, 0, 3, 0, FALSE, -1, NULL, 0}, /* GIC */
    {0, 0, 4, 0, FALSE, -1, NULL, 0}, /* GTI */
    {0, 0, 6, 0, TRUE, -1, NULL, 0},  /* GPIO */
    {0, 0, 7, 0, FALSE, -1, NULL, 0}, /* MPI */
    {0, 0, 8, 0, FALSE, -1, NULL, 0}, /* MIO_PTP */
    {0, 0, 9, 0, FALSE, -1, NULL, 0}, /* RNM */
    {0, 0, 16, 0, TRUE, -1, NULL, 0}, /* USB 0 */
    {0, 0, 17, 0, TRUE, -1, NULL, 0}, /* USB 1 */
    {0, 1,  0, 0, TRUE, 0, ecam_probe_true, 0},     /* MRML */
    {0, 1,  0, 1, TRUE, 0, ecam_probe_true, 0},     /* RST */
    {0, 1,  0, 5, TRUE, 0, ecam_probe_true, 0},     /* OCX */
    {0, 1,  0, 12, TRUE, 0, ecam_probe_true, 0},    /* MIO_EMM */
    {0, 1,  0, 64, FALSE, -1, ecam_probe_false, 0}, /* UAA 0 */
    {0, 1,  0, 65, FALSE, -1, ecam_probe_false, 0}, /* UAA 1 */
    {0, 1 , 0, 72, TRUE, 0, ecam_probe_true, 0}, /* TWSI 0 */
    {0, 1 , 0, 73, TRUE, 0, ecam_probe_true, 0}, /* TWSI 1 */
    {0, 1 , 0, 74, TRUE, 0, ecam_probe_true, 0}, /* TWSI 2 */
    {0, 1 , 0, 75, TRUE, 0, ecam_probe_true, 0}, /* TWSI 3 */
    {0, 1 , 0, 76, TRUE, 0, ecam_probe_true, 0}, /* TWSI 4 */
    {0, 1 , 0, 77, TRUE, 0, ecam_probe_true, 0}, /* TWSI 5 */

    {0, 1 , 0, 80, TRUE, 0, ecam_probe_true, 0}, /* LMC 0 */
    {0, 1 , 0, 81, TRUE, 0, ecam_probe_true, 0}, /* LMC 1 */
    {0, 1 , 0, 82, TRUE, 0, ecam_probe_true, 0}, /* LMC 2 */
    {0, 1 , 0, 83, TRUE, 0, ecam_probe_true, 0}, /* LMC 3 */

    {0, 1 , 0, 112, TRUE, 0, ecam_probe_pem, 0}, /* PEM 0 */
    {0, 1 , 0, 113, TRUE, 0, ecam_probe_pem, 1}, /* PEM 1 */
    {0, 1 , 0, 114, TRUE, 0, ecam_probe_pem, 2}, /* PEM 2 */
    {0, 1 , 0, 115, TRUE, 0, ecam_probe_pem, 3}, /* PEM 3 */
    {0, 1 , 0, 116, TRUE, 0, ecam_probe_pem, 4}, /* PEM 4 */
    {0, 1 , 0, 117, TRUE, 0, ecam_probe_pem, 5}, /* PEM 5 */

    {0, 1,  0, 128, TRUE, 0, ecam_probe_bgx, 0}, /* BGX 0 */
    {0, 1,  0, 129, TRUE, 0, ecam_probe_bgx, 1}, /* BGX 1 */

    {0, 2,  0, 0, TRUE, -1, NULL, 0}, /* RAD */
    {0, 3,  0, 0, TRUE, -1, NULL, 0}, /* ZIP */
    {0, 4,  0, 0, TRUE, -1, NULL, 0}, /* HFA */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};

struct ecam_device devs5[] = {
    {1, 0,  1, 0, FALSE, -1, NULL, 0},            /* SMMU 1 */
    {1, 0,  4, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 0 */
    {1, 0,  5, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 1 */
    {1, 0,  6, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 2 */
    {1, 0,  7, 0, TRUE, -1, ecam_probe_sata, 2},  /* SATA 3 */
    {1, 0,  8, 0, TRUE, -1, ecam_probe_sata, 3},  /* SATA 4 */
    {1, 0,  9, 0, TRUE, -1, ecam_probe_sata, 3},  /* SATA 5 */
    {1, 0,  10, 0, TRUE, -1, ecam_probe_sata, 3}, /* SATA 6 */
    {1, 0,  11, 0, TRUE, -1, ecam_probe_sata, 3}, /* SATA 7 */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};

struct ecam_device devs6[] = {
    {2, 0,  1, 0, FALSE, -1, NULL, 0},            /* SMMU 2 */
    {2, 0,  2, 0, TRUE, -1, NULL, 0},             /* PCCBR_NIC */
    {2, 0,  3, 0, TRUE, -1, NULL, 0},             /* TNS */
    {2, 1,  0, 0, TRUE, -1, NULL, 0},             /* NIC */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};

struct ecam_device devs7[] = {
    {3, 0,  1, 0, FALSE, -1, NULL, 0},
    {3, 0,  4, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 8 */
    {3, 0,  5, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 9 */
    {3, 0,  6, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 10 */
    {3, 0,  7, 0, TRUE, -1, ecam_probe_sata, 6},  /* SATA 11 */
    {3, 0,  8, 0, TRUE, -1, ecam_probe_sata, 7},  /* SATA 12 */
    {3, 0,  9, 0, TRUE, -1, ecam_probe_sata, 7},  /* SATA 13 */
    {3, 0,  10, 0, TRUE, -1, ecam_probe_sata, 7}, /* SATA 14 */
    {3, 0,  11, 0, TRUE, -1, ecam_probe_sata, 7}, /* SATA 15 */
    {-1, 0, 0, 0,  0, -1, NULL, 0},
};
```

# 有对板类型的判断

strcmp(env_ptr, "BOARD=crb_1s") 

这里似乎只有通过twsi向BMC发shutdown, 在PSCI服务中.

这难道和不能整体关机有关?

# 一些默认值
`atf/plat/thunder/include/thunder_def.h`
```
#define DRAM_TOTAL_SIZE			0x80000000

/* Location of trusted dram on the base thunder */
#define TZDRAM_BASE			0x00000000
#define TZDRAM_SIZE			0x00200000 //2M

#define NTDRAM_BASE			(TZDRAM_BASE + TZDRAM_SIZE)
#define NTDRAM_SIZE			(DRAM_TOTAL_SIZE - TZDRAM_SIZE)

#define SHARED_MEM_BASE			NTDRAM_BASE
#define SHARED_MEM_SIZE			0x00200000

#define BL1_BASE			0x00001000
#define BL1_MAX_SIZE			0x00079000

#define BL31_BASE			0x00080000
#define BL31_MAX_SIZE			0x00100000
#define BL31_LIMIT			(BL31_BASE + BL31_MAX_SIZE)

#define BL2_BASE			0x00100000
#define BL2_MAX_SIZE			0x00080000
#define BL2_LIMIT			(BL2_BASE + BL2_MAX_SIZE)
```

# bl1 bl2
大概浏览了一下, 大约就是加载下一个bl然后运行, 期间要设置好各自的内存map. bootloader一直都比较混乱

# bl31
```c
bl31_main()
    bl31_arch_setup()
    bl31_platform_setup()
        gic_setup()
        thunder_pwrc_setup()
        thunder_setup_topology()
    bl31_lib_init()
    runtime_svc_init()
        //运行时服务, 都是静态编译到下面的标号的, 一个服务是一个描述符
        &__RT_SVC_DESCS_START__
        //每个服务rt_svc_descs[index].init
    bl31_prepare_next_image_entry()
```