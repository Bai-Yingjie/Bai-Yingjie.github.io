记录浏览器里打开的, 还没来得及看完的, 有用的, 怕丢的topic

- [产品相关](#产品相关)
- [CPU相关](#cpu相关)
- [VM相关](#vm相关)
- [container相关](#container相关)
- [rust相关](#rust相关)
- [kernel相关](#kernel相关)
- [网络相关](#网络相关)
- [profiling相关](#profiling相关)
- [go相关](#go相关)
- [OS和shell](#os和shell)

# 产品相关
* [profiling 工具](https://acos.int.net.nokia.com/wiki/g/isamsw/Reborn%20platform%3A%20analysis%20tools#Memory_error_detection)
* [brugal iwf qemu 2018](https://acos.int.net.nokia.com/wiki/g/isamsw/BRUGAL%20IWF%20derisking#QEMU)
* [ihub team: IHUB Host Simulation](https://confluence.ext.net.nokia.com/display/core1/2021/03/25/IHUB+Host+Simulation)
* [devtool proposal 2014](https://acos.int.net.nokia.com/wiki/g/isamsw/Tools%20reorganization%20proposal)

# CPU相关
* [octeon 10 DPU](https://www.marvell.com/products/data-processing-units.html)
  * [ML](https://www.marvell.com/content/dam/marvell/en/public-collateral/embedded-processors/marvell-octeon-10-dpu-platform-white-paper.pdf)
* [AARCH64手册](https://developer.arm.com/documentation/102142/0100/)

# VM相关
* [cloud hypervisor](https://github.com/cloud-hypervisor/cloud-hypervisor/tree/main/docs)
* [kernel kvm手册](https://www.kernel.org/doc/html/latest/virt/kvm/api.html?highlight=kvm_ioeventfd)
* [vsock 博客](https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2020/04/18/vsock-internals)
* [cloudhypervisor vhost](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/vhost-user-net-testing.md)

# container相关
* [containerd runtime v2](https://github.com/containerd/containerd/tree/main/runtime/v2)

# rust相关
* [rust os对比](https://github.com/flosse/rust-os-comparison)
  * [RedOS](https://www.redox-os.org/)
  * [从零开始用rust写OS?](https://os.phil-opp.com/)
* [rust for linux 简单代码样例](https://github.com/Rust-for-Linux/linux/tree/rust/samples/rust)
* [rust命令行app入门手册](https://rust-cli.github.io/book/index.html)
* [The Rust Programming Language](https://doc.rust-lang.org/book/ch00-00-introduction.html)
* [The Rust Reference](https://doc.rust-lang.org/reference/introduction.html)
* [Easy Rust](https://fongyoong.github.io/easy_rust/Chapter_1.html)
* [Rust By Example](https://doc.rust-lang.org/rust-by-example/trait/drop.html)
* [嵌入式rust](https://docs.rust-embedded.org/book/intro/index.html)
* [rust std库手册](https://doc.rust-lang.org/stable/std/)
  * [std::sync::Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html)
  * [std::cmp::PartialOrd](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html)
  * [std::vec::Vec](https://web.mit.edu/rust-lang_v1.26.0/arch/amd64_ubuntu1404/share/doc/rust/html/std/vec/struct.Vec.html#method.shrink_to_fit)
  * [std::io::Read](https://doc.rust-lang.org/std/io/trait.Read.html)
* [rust编译器开发](https://blog.logrocket.com/debugging-rust-apps-with-gdb/)
* [rustp使用手册](https://rust-lang.github.io/rustup/index.html)
* [用gdb debug rust](https://blog.logrocket.com/debugging-rust-apps-with-gdb/)
* [用rust写kernel系列](https://os.phil-opp.com/minimal-rust-kernel/)
* [rust nvme驱动](https://lpc.events/event/16/contributions/1180/attachments/1017/1961/deck.pdf)
  * 性能和C几乎一样
* [rust async](https://rust-lang.github.io/async-book/01_getting_started/02_why_async.html)

# kernel相关
* [kernel kobject](https://www.kernel.org/doc/html/latest/core-api/kobject.html)

# 网络相关
* [RHEL 8 网络手册 ebpf](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/network-tracing-using-the-bpf-compiler-collection_configuring-and-managing-networking#doc-wrapper)
* [vpp interface](https://s3-docs.fd.io/vpp/23.02/gettingstarted/progressivevpp/interface.html)
* [使用vpp设计看k8s CNI](https://link.springer.com/chapter/10.1007/978-981-19-2456-9_18)
* [vpp](https://s3-docs.fd.io/vpp/23.10/)
  * [wiki](https://wiki.fd.io/view/VPP)
  * [startup.conf](https://my-vpp-docs.readthedocs.io/en/latest/gettingstarted/users/configuring/startup.html)
  * [创建interface](https://s3-docs.fd.io/vpp/23.06/gettingstarted/progressivevpp/interface.html)
  * [VPP in kubernetes](https://s3-docs.fd.io/vpp/23.10/usecases/contiv/index.html)
  * [vpp连接VM](https://s3-docs.fd.io/vpp/23.10/usecases/vhost/index.html)
  * [vpp add new plugin](https://fd.io/docs/vpp/v2101/gettingstarted/developers/add_plugin.html)
  * [vpp vxlan](https://wiki.fd.io/view/VPP/Using_VPP_as_a_VXLAN_Tunnel_Terminator)
  * [vpp testbench](https://s3-docs.fd.io/vpp/22.10/usecases/vpp_testbench/index.html)
* [Ligato/VPP Agent: vpp control plane management](https://github.com/ligato/vpp-agent)
* [Ligato/vpp-probe: debug vpp in distributed systems](https://github.com/ligato/vpp-probe)
* [OVS vxlan 准确, 详细](https://blog.oddbit.com/post/2021-04-17-vm-ovs-vxlan/)
* [Calio的数据面支持eBPF, VPP](https://github.com/projectcalico/calico)
* [eBPF和OVS](https://lpc.events/event/2/contributions/107/attachments/105/130/ovs-ebpf-afxdp-presentation.pdf)
  * 目的是用eBPF代码完全替代kernel的OVS DP
  * 效果一般, 实施难度大

# profiling相关
* [lwn: systemtap vs bpftrace](https://lwn.net/Articles/852112/)
* [github: iovisor bcc bpftrace](https://github.com/iovisor)
* [lwn: ebpf介绍](https://lwn.net/Articles/740157/)

# go相关
* [gobpf](https://github.com/iovisor/gobpf)

# OS和shell
* [bash官方手册](https://www.gnu.org/software/bash/manual/bash.html)