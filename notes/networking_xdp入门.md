- [Prerequisites](#prerequisites)
- [Task 1: Write and run a simple program with XDP](#task-1-write-and-run-a-simple-program-with-xdp)
  - [Task 1.1: Write a program to drop all packets](#task-11-write-a-program-to-drop-all-packets)
  - [Task 1.2: Build and dump the BPF object](#task-12-build-and-dump-the-bpf-object)
  - [Task 1.3: Load a BPF object](#task-13-load-a-bpf-object)
  - [Task 1.4: Show information on a running BPF object](#task-14-show-information-on-a-running-bpf-object)
  - [Task 1.5: Unload the XDP program](#task-15-unload-the-xdp-program)
- [Task 2: Drop specific packets with XDP](#task-2-drop-specific-packets-with-xdp)
- [Task 3: Map and count the processed packets](#task-3-map-and-count-the-processed-packets)
- [Task 4: Load XDP objects with the custom loader](#task-4-load-xdp-objects-with-the-custom-loader)
- [Summary](#summary)

Get started with XDP
原文: https://developers.redhat.com/blog/2021/04/01/get-started-with-xdp#

XDP (eXpress Data Path) is a powerful new networking feature in Linux that enables high-performance programmable access to networking packets before they enter the networking stack. But XDP has a high learning curve. Many developers have written introduction blogs for this feature, such as Paolo Abeni's [*Achieving high-performance, low-latency networking with XDP: Part I *](https://developers.redhat.com/blog/2018/12/06/achieving-high-performance-low-latency-networking-with-xdp-part-1/)and Toke's *[Using the eXpress Data Path (XDP) in Red Hat Enterprise Linux 8](https://www.redhat.com/en/blog/using-express-data-path-xdp-red-hat-enterprise-linux-8)*.

XDP is based on [extended Berkeley Packet Filter (eBPF)](https://opensource.com/article/17/9/intro-ebpf) and is still fast-moving. The eBPF/XDP coding format and style are also changing. So developers are creating tools and frameworks to make eBPF and XDP applications easy to write. Two of these resources, the [libbpf](https://github.com/libbpf/libbpf.git) library and the [xdp-tools ](https://github.com/xdp-project/xdp-tools)utilities, are the topics of this article.

The article shows how to start writing XDP programs through the following tasks:

1.  Write and run a small introductory program:
    1.  Write a program to drop all packets.
    2.  Build and view a BPF object.
    3.  Load a BPF object.
    4.  Show information on a running BPF object.
    5.  Unload a BPF object.
2.  Extend the program to let you deal with specific types of packets.
3.  Use a packet counter to use BPF maps.
4.  Add a customized userspace tool to load the BPF program.

The reader needs to be familiar with C code and IP header structures. All examples are tested with Red Hat Enterprise Linux (RHEL) 8.3.

Prerequisites
-------------

To prepare the developing environment, install the following packages:

```shell
$ sudo dnf install clang llvm gcc libbpf libbpf-devel libxdp libxdp-devel xdp-tools bpftool kernel-headers
```

Task 1: Write and run a simple program with XDP
-----------------------------------------------

This section teaches the minimal tasks you need to use XDP.

### Task 1.1: Write a program to drop all packets

Let's start with a simple XDP program in C:

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp_drop")
int xdp_drop_prog(struct xdp_md *ctx)
{
    return XDP_DROP;
}

char _license[] SEC("license") = "GPL";
```

The `linux/bpf.h` header file is provided by the kernel-header package, which defines all the supported BPF helpers and xdp_actions, like the `XDP_DROP` action used in this example.

The `bpf/bpf_helpers.h` header is provided by libbpf-devel, which provides some useful eBPF macros, like the `SEC` macro used in this example. `SEC`, short for *section,* is used to place a fragment of the compiled object in different ELF sections, which we will see in later output from the `llvm-objdump` command.

The `xdp_drop_prog()` function takes a parameter `struct xdp_md *ctx`, which we have not used yet. I will talk about it later. This function returns `XDP_DROP`, which means we will drop all incoming packets. Other XDP actions include `XDP_PASS`, `XDP_TX`, and `XDP_REDIRECT`.

Finally, the last line formally specifies the license associated with this program. Some eBPF helpers are accessible only by GPL-licensed programs, and the verifier will use this information to enforce such a restriction.

### Task 1.2: Build and dump the BPF object

Let's build the program in the previous section with `clang`:

```shell
**$ clang -O2 -g -Wall -target bpf -c xdp_drop.c -o xdp_drop.o**
```

The `-O` option specified which optimization level to use,  and`-g` generates debugging information.

You can use `llvm-objdump` to show the ELF format after the build. `llvm-objdump` is very useful if you want to know what a program does and you don't have the source code. The `-h` option displays the sections in the object, and the `-S` option displays the source interleaved with the disassembled object code. We'll show each of those options in turn.

```shell
**$ llvm-objdump -h xdp_drop.o**
xdp_drop:       file format ELF64-BPF

Sections:
Idx Name            Size     VMA              Type
  0                 00000000 0000000000000000
  1 .strtab         000000ad 0000000000000000
  2 .text           00000000 0000000000000000 TEXT
  3 xdp_drop        00000010 0000000000000000 TEXT
  4 license         00000004 0000000000000000 DATA
  5 .debug_str      00000125 0000000000000000
  6 .debug_abbrev   000000ba 0000000000000000
  7 .debug_info     00000114 0000000000000000
  8 .rel.debug_info 000001c0 0000000000000000
  9 .BTF            000001df 0000000000000000
 10 .rel.BTF        00000010 0000000000000000
 11 .BTF.ext        00000050 0000000000000000
 12 .rel.BTF.ext    00000020 0000000000000000
 13 .eh_frame       00000030 0000000000000000 DATA
 14 .rel.eh_frame   00000010 0000000000000000
 15 .debug_line     00000084 0000000000000000
 16 .rel.debug_line 00000010 0000000000000000
 17 .llvm_addrsig   00000002 0000000000000000
 18 .symtab         000002d0 0000000000000000

**$ llvm-objdump -S -no-show-raw-insn xdp_drop.o**
xdp_drop:       file format ELF64-BPF

Disassembly of section xdp_drop:

0000000000000000 xdp_drop_prog:
;       return XDP_DROP;
       0:       r0 = 1
       1:       exit

```

### Task 1.3: Load a BPF object

After you build the object, there are multiple ways to load it.

> **Warning**: Do not load test XDP programs on the default interface. Instead, use the [veth interface](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/#veth) for testing. This is advisable to protect you from losing network connectivity because the program is dropping packets.

The easiest way to load the program is using the `ip` command, like this:

```shell
**$ sudo ip link set veth1 xdpgeneric obj xdp_drop.o sec xdp_drop**
```

But`ip` doesn't support the [BPF Type Format (BTF) ](https://www.kernel.org/doc/html/latest/bpf/btf.html)type map that we will talk about later. Although the latest [ip-next ](https://git.kernel.org/pub/scm/network/iproute2/iproute2-next.git/commit/?id=f98ce50046b433687c0a661b6b9107a0603d1058)release has fixed this issue by adding libbpf support, it has not been backported to the main line yet.

The recommended way to load the XDP object on RHEL is using `xdp-loader`. This command depends on libbpf, which has full BTF support, and is the only way to load multiple programs on one interface.

To get Red Hat support for XDP, use `libxdp`, as explained in the article [XDP is conditionally supported in RHEL 8.3 release notes](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/8.3_release_notes/rhel-8-3-0-release#enhancement_networking).

Now let's load the object on interface `veth1` with `xdp-loader`. We specify `-m sbk` to use skb mode. Other possible modes include native and offload. But because these modes are not supported on all NIC drivers, we will just use skb mode in this article. The`-s xdp_drop` option specifies the use of the section we created, `xdp_drop`:

```shell
**$ sudo xdp-loader load -m skb -s xdp_drop veth1 xdp_drop.o**
```

### Task 1.4: Show information on a running BPF object

There are also multiple ways to show information about a loaded XDP program:

```shell
**$ sudo xdp-loader status** CURRENT XDP PROGRAM STATUS:

Interface        Prio  Program name     Mode     ID   Tag               Chain actions
-------------------------------------------------------------------------------------
lo               <no XDP program>
ens3             <no XDP program>
veth1                  xdp_dispatcher   skb      15   d51e469e988d81da
 =>              50    xdp_drop_prog             20   57cd311f2e27366b  XDP_PASS

$ **sudo bpftool prog show**
15: xdp  name xdp_dispatcher  tag d51e469e988d81da  gpl
        loaded_at 2021-01-13T03:24:43-0500  uid 0
        xlated 616B  jited 638B  memlock 4096B  map_ids 8
        btf_id 12
20: ext  name xdp_drop_prog  tag 57cd311f2e27366b  gpl
        loaded_at 2021-01-13T03:24:43-0500  uid 0
        xlated 16B  jited 40B  memlock 4096B
        btf_id 16

$ **sudo ip link show veth1**
4: veth1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpgeneric qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ba:4d:98:21:3b:b3 brd ff:ff:ff:ff:ff:ff link-netns ns
    prog/xdp id 15 tag d51e469e988d81da jited

```

If you load your program with `ip` cmd, only one XDP program can be loaded simultaneously. If you load your program with`xdp-loader`, two programs will be loaded by default. One is `xdp_dispatcher`, created by `xdp_loader`, and the other is `xdp_drop_prog`, written by us. The second command issued above shows that `xdp_dispatcher` is running with ID 15 and `xdp_drop_prog` is running with ID 20.

### Task 1.5: Unload the XDP program

If you use `ip cmd` to load the program, you can unload the program through:

```shell
**$ sudo ip link set veth1 xdpgeneric off**
```

Use the `xdp` flag that corresponds to the way you loaded the file. In this example, we specified `xdpgeneric off` because we loaded our program beginning with `ip link set veth1 xdpgeneric obj`. Specify `xdp off` if you loaded your beginning with `ip link set veth1 xdp obj`.

To unload all XDP programs on an interface, issue `xdp-loader` with the `-a` option:

```shell
**$ sudo xdp-loader unload -a veth1**
```

Task 2: Drop specific packets with XDP
--------------------------------------

The first example dropped every packet, which has no practical use. Now let's do some real stuff. The example in this section drops all IPv6 packets:

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <linux/if_ether.h>
#include <arpa/inet.h>

SEC("xdp_drop")
int xdp_drop_prog(struct xdp_md *ctx)
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
    __u16 h_proto;

    if (data + sizeof(struct ethhdr) > data_end)
        return XDP_DROP;

    h_proto = eth->h_proto;

    if (h_proto == htons(ETH_P_IPV6))
        return XDP_DROP;

    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";

```

Compare the code just shown with the first program. Here we added two more header files:`linux/if_ether.h` to get the`ethhdr` struct and `arpa/inet.h` to get the `htons()` function.

The `struct xdp_md` is also used in this example. It's defined (in Linux 5.10) like this:

```c
struct xdp_md {
    __u32 data;
    __u32 data_end;
    __u32 data_meta;
    /* Below access go through struct xdp_rxq_info */
    __u32 ingress_ifindex; /* rxq->dev->ifindex */
    __u32 rx_queue_index;  /* rxq->queue_index  */

    __u32 egress_ifindex;  /* txq->dev->ifindex */
};
```

The packet contents lie between `ctx->data` and `ctx->data_end`. The data starts with an Ethernet header, so we assign the data to`ethhdr` like this:

```c
    void *data = (void *)(long)ctx->data;
    struct ethhdr *eth = data;
```

When accessing the data in `struct ethhdr`, we must make sure we don't access invalid areas by checking whether `data + sizeof(struct ethhdr) > data_end`, and returning without further action if it's true. This check is compulsory by the [BPF verifer](https://www.kernel.org/doc/html/latest/networking/filter.html#ebpf-verifier) that verifies your program at runtime.

Then, determine whether the protocol in the Ethernet header is IPv6 by checking `h_proto == htons(ETH_P_IPV6)`, and if it is, drop the packet by returning `XDP_DROP`. For other packets, we just return `XDP_PASS`.

Task 3: Map and count the processed packets
-------------------------------------------

In the previous example, we dropped IPv6 packets. In this example, we will keep track of how many packets we dropped. The example introduces another BPF feature: maps. BPF maps are used to share data between the kernel and userspace. We can update the map data in the kernel and read it from userspace, or vice versa.

Here is an example of a new BTF-defined map (introduced by upstream [commit abd29c931459](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git/commit/?id=abd29c931459)):

```c
struct {
        __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
        __type(key, __u32);
        __type(value, long);
        __uint(max_entries, 1);
} rxcnt SEC(".maps");
```

The map is named as `rxcnt` with type `BPF_MAP_TYPE_PERCPU_ARRAY`. This type indicates that we will have one instance of the map per CPU core; thus, if you have 4 cores, you will have 4 instances of the map. We will use each map to count how many packets are processed per core. The rest of the structure defines the key/value type and limits the maximum number entries to 1, because we need to count only one number (the number of received IPv6 packets). In C code, it would look like we defined an array on each CPU with a size of one, .e.g, `unsigned int rxcnt[1]`.

Our program looks up the value of the `rxcnt` entry with the function `bpf_map_lookup_elem()` and updates the value. Here is the full code:

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>
#include <linux/if_ether.h>
#include <arpa/inet.h>

struct {
        __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
        __type(key, __u32);
        __type(value, long);
        __uint(max_entries, 1);
} rxcnt SEC(".maps");

SEC("xdp_drop_ipv6")
int xdp_drop_ipv6_prog(struct xdp_md *ctx)
{
        void *data_end = (void *)(long)ctx->data_end;
        void *data = (void *)(long)ctx->data;
        struct ethhdr *eth = data;
        __u16 h_proto;
        __u32 key = 0;
        long *value;

        if (data + sizeof(struct ethhdr) > data_end)
                return XDP_DROP;

        h_proto = eth->h_proto;

        if (h_proto == htons(ETH_P_IPV6)) {
                value = bpf_map_lookup_elem(&rxcnt, &key);
                if (value)
                        *value += 1;
                return XDP_DROP;
        }

        return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

Let's name the new program `xdp_drop_ipv6_count.c` and build it to create the object file `xdp_drop_ipv6_count.o`. After loading the object, send some IPv6 packets to this interface. Using the `bpftool map show` command, we can see that the `rxcnt` ID in our map is 13\. Then we can use `bpftool map dump id 13` to show that 13 packets were processed on CPU 0 and 7 packets were processed on CPU 1:

```shell
**$ sudo xdp-loader load -m skb -s xdp_drop_ipv6 veth1 xdp_drop_ipv6_count.o**
...*receive some IPv6 packets*

**$ sudo bpftool map show** bpftool map show
13: percpu_array  name rxcnt  flags 0x0
        key 4B  value 8B  max_entries 1  memlock 4096B
        btf_id 20
19: array  name xdp_disp.rodata  flags 0x480
        key 4B  value 84B  max_entries 1  memlock 8192B
        btf_id 28  frozen
# bpftool map dump id 13
[{
        "key": 0,
        "values": [{
                "cpu": 0,
                "value": 13
            },{
                "cpu": 1,
                "value": 7
            },{
                "cpu": 2,
                "value": 0
            },{
                "cpu": 3,
                "value": 0
            }
        ]
    }
]

```

BPF supports many more [map types](https://prototype-kernel.readthedocs.io/en/latest/bpf/ebpf_maps_types.html), such as BPF_MAP_TYPE_HASH, BPF_MAP_TYPE_ARRAY, etc.

Task 4: Load XDP objects with the custom loader
-----------------------------------------------

We can load the XDP objects with`ip` and show the map number with `bpftool`. But if we want more advanced features (to create, read, and write maps, attach XDP programs to interfaces, etc.), we need to write the loader ourselves.

Here is an example of how to show the total packets and the number of packets per second (PPS) we dropped. In this example, I hard-coded a lot of stuff such as kernel object name, section name, etc. These all can be set via parameters in your own code.

The purpose of each function is explained by comments in the code:

```c
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <linux/if_link.h>
#include <signal.h>
#include <net/if.h>
#include <assert.h>

/* In this example we use libbpf-devel and libxdp-devel */
#include <bpf/bpf.h>
#include <bpf/libbpf.h>
#include <xdp/libxdp.h>

/* We define the following global variables */
static int ifindex;
struct xdp_program *prog = NULL;

/* This function will remove XDP from the link when the program exits. */
static void int_exit(int sig)
{
    xdp_program__close(prog);
    exit(0);
}

/* This function will count the per-CPU number of packets and print out
 * the total number of dropped packets number and PPS (packets per second).
 */
static void poll_stats(int map_fd, int interval)
{
    int ncpus = libbpf_num_possible_cpus();
    if (ncpus < 0) {
        printf("Error get possible cpus\n");
        return;
    }
    long values[ncpus], prev[ncpus], total_pkts;
    int i, key = 0;

    memset(prev, 0, sizeof(prev));

    while (1) {
        long sum = 0;

        sleep(interval);
        assert(bpf_map_lookup_elem(map_fd, &key, values) == 0);
        for (i = 0; i < ncpus; i++)
            sum += (values[i] - prev[i]);
        if (sum) {
            total_pkts += sum;
            printf("total dropped %10llu, %10llu pkt/s\n",
                   total_pkts, sum / interval);
        }
        memcpy(prev, values, sizeof(values));
    }
}

int main(int argc, char *argv[])
{
    int prog_fd, map_fd, ret;
    struct bpf_object *bpf_obj;

    if (argc != 2) {
        printf("Usage: %s IFNAME\n", argv[0]);
        return 1;
    }

    ifindex = if_nametoindex(argv[1]);
    if (!ifindex) {
        printf("get ifindex from interface name failed\n");
        return 1;
    }

    /* load XDP object by libxdp */
    prog = xdp_program__open_file("xdp_drop_ipv6_count.o", "xdp_drop_ipv6", NULL);
    if (!prog) {
        printf("Error, load xdp prog failed\n");
        return 1;
    }

    /* attach XDP program to interface with skb mode
     * Please set ulimit if you got an -EPERM error.
     */
    ret = xdp_program__attach(prog, ifindex, XDP_MODE_SKB, 0);
    if (ret) {
        printf("Error, Set xdp fd on %d failed\n", ifindex);
        return ret;
    }

    /* Find the map fd from the bpf object */
    bpf_obj = xdp_program__bpf_obj(prog);
    map_fd = bpf_object__find_map_fd_by_name(bpf_obj, "rxcnt");
    if (map_fd < 0) {
        printf("Error, get map fd from bpf obj failed\n");
        return map_fd;
    }

    /* Remove attached program when it is interrupted or killed */
    signal(SIGINT, int_exit);
    signal(SIGTERM, int_exit);

    poll_stats(map_fd, 2);

    return 0;
}

```

Set the `ulimit` to unlimited and build the program with the `-lbpf -lxdp` flags. Then run the program, which shows the packet count output:

```shell
**$ sudo ulimit -l unlimited
$ gcc xdp_drop_ipv6_count_user.c -o xdp_drop_ipv6_count -lbpf -lxdp
$ sudo ./xdp_drop_ipv6_count veth1**
total dropped          2,          1 pkt/s
total dropped        129,         63 pkt/s
total dropped        311,         91 pkt/s
total dropped        492,         90 pkt/s
total dropped        674,         91 pkt/s
total dropped        856,         91 pkt/s
total dropped       1038,         91 pkt/s
^C

```

Summary
-------

This article helps you understand what a XDP program looks like, how to add a BPF map, and how to write a custom loader. To learn more about XDP programming, please visit [xdp-tutorial](https://github.com/xdp-project/xdp-tutorial).

*Last updated: April 7, 2022*