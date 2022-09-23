- [CN70XX ddr ecc](#cn70xx-ddr-ecc)
- [配置文件里的odt配置](#配置文件里的odt配置)
- [如何打印所有寄存器?](#如何打印所有寄存器)
- [ddr测试](#ddr测试)

# CN70XX ddr ecc
in uboot
```
setenv ddr_maximum_adjacent_rlevel_delay_increment 1
```

in lib_octeon_shared.c, do the following changes:
```c
    #define RLEVEL_BITMASK_BUBBLE_BITS_ERROR        4
   Replace with:
    #define RLEVEL_BITMASK_BUBBLE_BITS_ERROR        12

    #define RLEVEL_ADJACENT_DELAY_ERROR             2
   Replace with:
    #define RLEVEL_ADJACENT_DELAY_ERROR             30

    #define RLEVEL_NONSEQUENTIAL_DELAY_ERROR        5
   Replace with:
    #define RLEVEL_NONSEQUENTIAL_DELAY_ERROR        50
```

# 配置文件里的odt配置
 
注意这里的配置和那个结构体对不上
```
#define OCTEON_EVB7100_CN61XX_DRAM_ODT_1RANK_CONFIGURATION \
    /* DIMMS   reserved  WODT_MASK                LMCX_MODEREG_PARAMS1            RODT_CTL    RODT_MASK    reserved */ \
    /* =====   ======== ============== ========================================== ========= ============== ======== */ \
    /*   1 */ {   0,    0x00000001ULL, OCTEON_EVB7100_MODEREG_PARAMS1_1RANK_1SLOT,    3,     0x00000000ULL,  0  },      \
    /*   2 */ {   0,    0x00050005ULL, OCTEON_EVB7100_MODEREG_PARAMS1_1RANK_2SLOT,    3,     0x00010004ULL,  0  }
#define OCTEON_EVB7100_CN61XX_DRAM_ODT_2RANK_CONFIGURATION \
    /* DIMMS   reserved  WODT_MASK                LMCX_MODEREG_PARAMS1            RODT_CTL    RODT_MASK    reserved */ \
    /* =====   ======== ============== ========================================== ========= ============== ======== */ \
    /*   1 */ {   0,    0x00000101ULL, OCTEON_EVB7100_MODEREG_PARAMS1_2RANK_1SLOT,    3,     0x00000000ULL,    0  },    \
    /*   2 */ {   0,    0x09090606ULL, OCTEON_EVB7100_MODEREG_PARAMS1_2RANK_2SLOT,    3,     0x01010404ULL,    0  }
#define OCTEON_EVB7100_CN61XX_DRAM_ODT_4RANK_CONFIGURATION \
    /* DIMMS   reserved  WODT_MASK                LMCX_MODEREG_PARAMS1            RODT_CTL    RODT_MASK    reserved */ \
    /* =====   ======== ============== ========================================== ========= ============== ======== */ \
    /*   1 */ {   0,    0x01030203ULL, OCTEON_EVB7100_MODEREG_PARAMS1_4RANK_1SLOT,    3,     0x01010202ULL,    0  }
 ```
 ```
        static const unsigned char rodt_ohms[] = { 0, 20, 30, 40, 60, 120 };
        static const unsigned char rtt_nom_ohms[] = { 0, 60, 120, 40, 20, 30 };
        static const unsigned char rtt_nom_table[] = { 0, 2, 1, 3, 5, 4 };
        static const unsigned char rtt_wr_ohms[]  = { 0, 60, 120 };
        static const unsigned char dic_ohms[]     = { 40, 34 };
```

结构体c
```
typedef struct {
    uint8_t odt_ena;
    uint64_t odt_mask;
#if defined(SWIG) || defined(SWIGLUA)
    uint64_t odt_mask1;
#else
    cvmx_lmcx_modereg_params1_t odt_mask1;
#endif
    uint8_t qs_dic;
    uint64_t rodt_ctl;
    uint8_t dic;
} dimm_odt_config_t;
```

# 如何打印所有寄存器?
在代码里面加
```
uint32_t model = cvmx_get_proc_id();
if (cvmx_get_core_num() == 0)
    cvmx_csr_db_print_decode_by_prefix(model, "lmc", 1);
```

# ddr测试
```c
//内存测试se, 在uboot下面运行
tftp hw-ddr2
多核带ecc
bootoct 0 coremask=0x0f endbootargs -ecc
多核不带ecc
bootoct 0 coremask=0x0f endbootargs
单核带ecc
bootoct 0 -ecc
单核不带ecc
bootoct 0
tftp hw-write-ddr2
bootoct 0 coremask=0x0f 
//打开ddr打印
Octeon cust_fpxtb(PKGA)# # setenv ddr_verbose 1
Octeon cust_fpxtb(PKGA)# # setenv ddr_debug 1
setenv ddr_trace_init 1
Octeon cust_fpxtb(PKGA)# # saveenv
//查看是否打开ecc
OCTEON_REMOTE_DEBUG=1 OCTEON_REMOTE_PROTOCOL=GDB:135.251.9.60,30000  oct-remote-csr cvmx_lmc0_config
//查单双bit Ecc统计
OCTEON_REMOTE_DEBUG=1 OCTEON_REMOTE_PROTOCOL=GDB:135.251.9.60,30000  oct-remote-csr cvmx_lmc0_int
//单双ecc
Octeon cust_fpxtb(PKGA)# # read64 0x00011800880001F0
attempting to read from addr: 0x80011800880001f0
0x80011800880001f0: 0x0
//强制read leveling
Octeon cust_fpxtb(PKGA)# # setenv ddr_rlevel_rank0_byte0 3
Octeon cust_fpxtb(PKGA)# # saveenv
//设auto ODT
setenv  ddr_rodt_ctl_auto 1
//loop数值, 设为10好点
setenv ddr_rlevel_average
//recompute --更差
ddr_rlevel_compute

//强制一组经验好参数
if ((s = getenv("ddr_good")) != NULL)
{                                    
    lmc_rlevel_rank.cn70xx.byte4 = 6;
    lmc_rlevel_rank.cn70xx.byte3 = 6;
    lmc_rlevel_rank.cn70xx.byte2 = 6;
    lmc_rlevel_rank.cn70xx.byte1 = 4;
    lmc_rlevel_rank.cn70xx.byte0 = 4;
    ddr_print("******Force DDR good********\n");
}

//强制rleveling
Octeon cust_fpxtb(PKGA)# # setenv ddr_rlevel_rank0_byte4 6
Octeon cust_fpxtb(PKGA)# # setenv ddr_rlevel_rank0_byte3 6
Octeon cust_fpxtb(PKGA)# # setenv ddr_rlevel_rank0_byte2 6
Octeon cust_fpxtb(PKGA)# # setenv ddr_rlevel_rank0_byte1 4
Octeon cust_fpxtb(PKGA)# # setenv ddr_rlevel_rank0_byte0 4
Octeon cust_fpxtb(PKGA)# # saveenv
```
