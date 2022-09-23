BDK是boot阶段执行的程序, 类似boot, 主要负责初始化和检测硬件.

- [编译BDK](#编译bdk)
- [bdk from uboot](#bdk-from-uboot)
- [octeon main](#octeon-main)
  - [bdk 流程](#bdk-流程)
- [78的配置](#78的配置)

# 编译BDK
```
apt install gcc-multilib
export BDK_ROOT=/home/byj/repo/git/octeon/bdk-trunk
export OCTEON_ROOT=/home/byj/repo/hg/octeon-sdk/OCTEON-SDK-3.1.1
export PATH=${PATH}:${BDK_ROOT}/bin:${OCTEON_ROOT}/tools/bin
make distclean
make
```

# bdk from uboot
```
usb start
fatls usb 0:1 /target/
fatload usb 0:1 0x400000 /target/bdk-boot-cn78xx.bin
go 0x400000
```

# octeon main
在`bdk-boot/bdk-full-no-romfs.c`

## bdk 流程
```c
__start(bdk-start.S)
__init(bdk-start.S)
拷贝code(header + data)到0x80002000  --为什么不是cache? 还是说在没有flush之前, cache里面的东西不动?
用COP0_CVMMEMCTL设置栈在cache中, 大小是54*128, 详细见HRM小节2.5 Virtual Addresses and CVMSEG
__bdk_init(bdk-init.c)
    __bdk_fs_init()
    __bdk_init_exception()
    如果没有初始化内存, 则锁cache
    ...
__bdk_init_main(bdk-init-main.c)
    cp0, config, bootbus, mdio初始化
main(bdk-full-no-romfs.c) 新的进程
    bdk_lua_start()
        bdk_fs_remote_init()
        bdk_fs_rom_init()
        bdk_fs_mem_init()
        bdk_fs_nor_init()
        bdk_fs_mmc_init()
        bdk_fs_mpi_init()
        bdk_fs_pcie_init()
        bdk_fs_ram_init()
        bdk_fs_xmodem_init()
        bdk_fs_sata_init()
        __bdk_lua_main() lua.c
            lua_State *L = luaL_newstate()
                bdk_lua_init()
                    luaL_getsubtable(L, LUA_REGISTRYINDEX, "_PRELOAD");
                    PRELOAD("bit64", luaopen_bit64);
                    PRELOAD("readline", luaopen_readline);
                    PRELOAD("rpc-support", luaopen_rpc_support);
                    //host模式下
                    PRELOAD("socket", luaopen_socket_core)
                    PRELOAD("oremote-internal", luaopen_oremote)
                    //target模式下
                    PRELOAD("cavium-internal", luaopen_cavium);
                        //注册子项, cavium_model, cavium_c, cavium_config, cavium_constants, cavium_perf, cavium_mmc, dram, csr
                        //比如cavium_c
                        lua_newtable(L)
                        //把每个function都注册到lua里面, 这些function都通过一个cavium_c_call来调用
                        //同时单独注册了bdk_csr_read, bdk_csr_write, 因为这两个都是宏
                        lua_setfield(L, -2, "c"); //把这个newtable填到L里面, key就是c
            lua_pushcfunction(L, &pmain);
            lua_pushinteger(L, argc);  /* 1st argument */
            lua_pushlightuserdata(L, argv); /* 2nd argument */
            status = lua_pcall(L, 2, 1, 0);
            result = lua_toboolean(L, -1);  /* get result */
            finalreport(L, status);
            lua_close(L);
        pmain()
            luaL_openlibs()
            handle_luainit()
            handle_script()
            dotty()
        /rom/init.lua
            while Ture:
                /rom/main.lua
                    board-setup
                    board-ebb7800
                        //其他的调用方法有octeon.c.bdk_csr_read, octeon.c.bdk_csr_write, 还有全在libbdk/bdk-functions.c里面
                        local set_config = octeon.c.bdk_config_set
                        set_config(octeon.CONFIG_PHY_IF0_PORT1, 0xff000001)
                        ...
```

# 78的配置
```c
void __bdk_require_depends(void)
{
    BDK_REQUIRE(QLM);
    BDK_REQUIRE(PCIE);
    BDK_REQUIRE(PCIE_EEPROM);
    BDK_REQUIRE(FS_PCIE);
    BDK_REQUIRE(GPIO);
    BDK_REQUIRE(RNG);
    BDK_REQUIRE(KEY_MEMORY);
    BDK_REQUIRE(MPI);
    BDK_REQUIRE(DRAM_CONFIG);
    BDK_REQUIRE(DRAM_TEST);
    BDK_REQUIRE(ENVIRONMENT);
    BDK_REQUIRE(FS_XMODEM);
    BDK_REQUIRE(FS_RAM);
    BDK_REQUIRE(FS_SATA);
    BDK_REQUIRE(CSR_DB);
    BDK_REQUIRE(TRAFFIC_GEN);
    BDK_REQUIRE(ERROR_DECODE);
    BDK_REQUIRE(SATA);
    BDK_REQUIRE(TWSI);
    BDK_REQUIRE(USB);
}
```