- [Introduction](#introduction)
  - [Bridge](#bridge)
    - [What is a bridge](#what-is-a-bridge)
    - [When to use a bridge](#when-to-use-a-bridge)
    - [How to create a bridge](#how-to-create-a-bridge)
  - [Bonding](#bonding)
    - [What is bonding](#what-is-bonding)
    - [When to use bonding](#when-to-use-bonding)
    - [How to create a bond](#how-to-create-a-bond)
  - [Team](#team)
    - [What is team](#what-is-team)
    - [When to use team](#when-to-use-team)
    - [How to create team](#how-to-create-team)
  - [VLAN](#vlan)
    - [What is VLAN](#what-is-vlan)
    - [When to use VLAN](#when-to-use-vlan)
    - [How to create VLAN](#how-to-create-vlan)
  - [VXLAN](#vxlan)
    - [What is VXLAN](#what-is-vxlan)
    - [When to use VXLAN](#when-to-use-vxlan)
    - [How to create vxlan](#how-to-create-vxlan)
  - [MACVLAN](#macvlan)
    - [What is MACVLAN](#what-is-macvlan)
    - [When to use MACVLAN](#when-to-use-macvlan)
    - [How to setup MACVLAN](#how-to-setup-macvlan)
  - [IPVLAN](#ipvlan)
    - [What is IPVLAN](#what-is-ipvlan)
    - [When to use IPVLAN](#when-to-use-ipvlan)
    - [How to setup IPVLAN](#how-to-setup-ipvlan)
  - [MACVTAP/IPVTAP](#macvtapipvtap)
    - [What is MACVTAP/IPVTAP](#what-is-macvtapipvtap)
    - [When to use MACVTAP/IPVTAP](#when-to-use-macvtapipvtap)
    - [How to create MACVTAP](#how-to-create-macvtap)
  - [MACsec](#macsec)
    - [What is MACsec](#what-is-macsec)
    - [When to use MACsec](#when-to-use-macsec)
    - [How to setup MACsec](#how-to-setup-macsec)
  - [VETH](#veth)
    - [What is VETH](#what-is-veth)
    - [When to use VETH](#when-to-use-veth)
    - [How to setup VETH](#how-to-setup-veth)
  - [VCAN](#vcan)
    - [What is VCAN](#what-is-vcan)
    - [When to use CAN](#when-to-use-can)
    - [How to create VCAN](#how-to-create-vcan)
  - [VXCAN](#vxcan)
    - [what is VXCAN](#what-is-vxcan)
    - [When to use VXCAN](#when-to-use-vxcan)
    - [How to setup VXCAN](#how-to-setup-vxcan)
  - [IPOIB](#ipoib)
    - [What is it](#what-is-it)
    - [When to use](#when-to-use)
    - [How to create](#how-to-create)
  - [nlmon](#nlmon)
    - [What is nlmon](#what-is-nlmon)
    - [When to use nlmon](#when-to-use-nlmon)
    - [How to create nlmon](#how-to-create-nlmon)
  - [Dummy](#dummy)
    - [What is Dummy](#what-is-dummy)
    - [When to use Dummy](#when-to-use-dummy)
    - [How to create Dummy](#how-to-create-dummy)
  - [IFB](#ifb)
    - [What is IFB](#what-is-ifb)
    - [When to use IFB](#when-to-use-ifb)
    - [How to create an IFB interface](#how-to-create-an-ifb-interface)

# Introduction

In this post I will give a brief introduction for all commonly used virtual interfaces. There is no code analysis. Only brief introduction and the usage on _Linux_. So anyone with a network background could have a look at this blog post. The interface list can be obtained with the command `ip link help`.

The first part is some frequently used interfaces and some interfaces that can be easily confused with one another. The second part will be a brief introduction of all kinds of tunnels(TBD).

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#bridge)Bridge

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-a-bridge)What is a bridge

A linux bridge behaves like a network switch. It forwards packets between interfaces that are connected on it. It’s usually used for forwarding packets on routers, gateways, or between VMs and network namespaces on a host. It also supports STP, VLAN filter and multicast snooping.

[![bridge](https://liuhangbin.netlify.app/bridge.png)](https://liuhangbin.netlify.app/bridge.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-a-bridge)When to use a bridge

When you want to establish communication channels between VMs, containers and your hosts.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-a-bridge)How to create a bridge


```
# ip link add br0 type bridge
# ip link set eth0 master br0
# ip link set tap1 master br0
# ip link set tap2 master br0
# ip link set veth1 master br0

```

This creates a bridge device named br0 and set two tap devices(tap1, tap2), a veth device(veth1) and a physical device(eth0) as its slaves, as shown in the picture.

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#bonding)Bonding

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-bonding)What is bonding

The Linux bonding driver provides a method for aggregating multiple network interfaces into a single logical “bonded” interface. The behavior of the bonded interfaces depends on the mode; generally speaking, modes provide either hot standby or load balancing services.

[![bond](https://liuhangbin.netlify.app/bond.png)](https://liuhangbin.netlify.app/bond.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-bonding)When to use bonding

When you want to increase your link speed or do failover on your server.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-a-bond)How to create a bond

```
ip link add bond1 type bond miimon 100 mode active-backup
ip link set eth0 master bond1
ip link set eth1 master bond1

```

This creates a bond interface named bond1 with mode active-backup. For other modes, please see: [Kernel bonding doc](https://www.kernel.org/doc/Documentation/networking/bonding.txt)

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#team)Team

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-team)What is team

Similar to bonding, the purpose of the team device is to provide a mechanism to team multiple NICs (ports) into one logical one (teamdev) at L2 layer.

[![team](https://liuhangbin.netlify.app/team.png)](https://liuhangbin.netlify.app/team.png)

The main thing to realize is that the Team project is not trying to replicate or mimic the bonding driver. What it does is to solve the same problem using a different approach, using e.g. lockless (RCU) TX/RX path and modular design.

But there are also some functional difference between bonding and team. e.g. team supports LACP load-balancing, NS/NA (IPV6) link monitoring, D-Bus interface, etc., that are absent in bonding. For further details about the differences between bonding and Team, see: [Bonding vs Team](https://github.com/jpirko/libteam/wiki/Bonding-vs.-Team-features).

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-team)When to use team

When you want to use some features that bonding doesn’t provide.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-team)How to create team

```
# teamd -o -n -U -d -t team0 -c '{"runner": {"name": "activebackup"},"link_watch": {"name": "ethtool"}}'
# ip link set eth0 down
# ip link set eth1 down
# teamdctl team0 port add eth0
# teamdctl team0 port add eth1

```

This creates a team interface named team0 with mode active-backup, and adds eth0, eth1 as team0’s sub-interfaces.

More info: a new driver called [net_failover](https://www.kernel.org/doc/html/latest/networking/net_failover.html) has been added to linux recently. It’s another failover master netdev for virtualization and manages a primary (a passthru/vf device) and standby (the original paravirtual interface) slave netdevs.

[![net_failover](https://liuhangbin.netlify.app/net_failover.png)](https://liuhangbin.netlify.app/net_failover.png)

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#vlan)VLAN

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-vlan)What is VLAN

VLAN, aka virtual LAN. It separates broadcast domains by adding tags to network packets. VLANs allow network administrators to group hosts under the same switch or between different switches.

The VLAN header looks like: [![vlan header](https://liuhangbin.netlify.app/vlan_01.png)](https://liuhangbin.netlify.app/vlan_01.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-vlan)When to use VLAN

When you want to seperate subnet in VMs, namespaces or hosts

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-vlan)How to create VLAN


```
# ip link add link eth0 name eth0.2 type vlan id 2
# ip link add link eth0 name eth0.3 type vlan id 3

```


This add VLAN 2 with name eth0.2 and vlan 3 with name eth0.3\. The topology looks like: [![vlan](https://liuhangbin.netlify.app/vlan.png)](https://liuhangbin.netlify.app/vlan.png)

_Note_: When configuring a VLAN, you need to make sure the switch connected to the host is able to handle VLAN tags, e.g. by setting the switch port to trunk mode.

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#vxlan)VXLAN

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-vxlan)What is VXLAN

VXLAN(Virtual eXtensible Local Area Network) protocol is a tunnelling protocol designed to solve the problem of limited VLAN IDs (4096) in IEEE 802.1q. It is described by [IETF RFC 7348](https://tools.ietf.org/html/rfc7348).

With a 24-bit segment ID, aka “VXLAN Network Identifier (VNI)”, VXLAN allows up to 2^24(16 777 216) virtual LANs, which is 4096 times of VLAN capacity.

VXLAN encapsulates Layer 2 frames with a VXLAN header into a UDP-IP packet, which looks like:

[![VXLAN Header](https://liuhangbin.netlify.app/vxlan_01.png)](https://liuhangbin.netlify.app/vxlan_01.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-vxlan)When to use VXLAN

VXLAN is typically deployed in data centers on virtualized hosts, which may be spread across multiple racks.

[![VXLAN TOPO](https://liuhangbin.netlify.app/vxlan.png)](https://liuhangbin.netlify.app/vxlan.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-vxlan)How to create vxlan


```
# ip link add vx0 type vxlan id 100 local 1.1.1.1 remote 2.2.2.2 dev eth0 dstport 4789

```

For reference, you can read the [vxlan kernel doc](https://www.kernel.org/doc/Documentation/networking/vxlan.txt) or [an other vxlan introduction](https://vincent.bernat.ch/en/blog/2017-vxlan-linux)

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#macvlan)MACVLAN

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-macvlan)What is MACVLAN

With VLAN, we can create multiple interfaces on top of a single one and filter packages based on VLAN tag. With MACVLAN, we can create multiple interfaces with different Layer 2 (i.e. Ethernet MAC) addresses on top of a single one .

Before MACVLAN, if users wanted to connect to physical network from VM or namespace, they would have needed to create TAP/VETH devices and attach one side to bridge, and attach a physical interface to the bridge on the host at the same time.

[![Namespaces on bridge](https://liuhangbin.netlify.app/br_ns.png)](https://liuhangbin.netlify.app/br_ns.png)

Now, with MACVLAN, they can bind a physical interface, associated to a MACVLAN, directly to namespaces, without the need for a bridge.

[![MACVLAN](https://liuhangbin.netlify.app/macvlan.png)](https://liuhangbin.netlify.app/macvlan.png)

There are four MACVLAN types:

1.  Private: doesn’t allow communication between MACVLAN instances on the same physical interface, even if the external switch supports hairpin mode.

    [![MACVLAN Private](https://liuhangbin.netlify.app/macvlan_01.png)](https://liuhangbin.netlify.app/macvlan_01.png)

2.  VEPA: data from one macvlan instance to the other on the same physical interface is transmitted over the physical interface. Either the attached switch needs to support hairpin mode, or there must be a TCP/IP router forwarding the packets in order to allow communication.

    [![MACVLAN VEPA](https://liuhangbin.netlify.app/macvlan_02.png)](https://liuhangbin.netlify.app/macvlan_02.png)

3.  Bridge: all endpoints are directly connected to each other with a simple bridge via physical interface.

    [![MACVLAN Bridge](https://liuhangbin.netlify.app/macvlan_03.png)](https://liuhangbin.netlify.app/macvlan_03.png)

4.  Passthru: allows a single VM to be connected directly to the physical interface.

    [![MACVLAN Passthru](https://liuhangbin.netlify.app/macvlan_04.png)](https://liuhangbin.netlify.app/macvlan_04.png)

5.  Source: the source mode is used to filter traffic based on a list of allowed source MAC addresses, to create MAC-based VLAN associations. Please see the [commit message](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net.git/commit/?id=79cf79abce71)

The type is chosen according to different needs. Bridge mode is the most commonly used

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-macvlan)When to use MACVLAN

Users want to connect directly to physical network from Containers.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-setup-macvlan)How to setup MACVLAN


```
# ip link add macv1 link eth0 type macvlan mode bridge
# ip link add macv2 link eth0 type macvlan mode bridge
# ip netns add net1
# ip netns add net2
# ip link set macvlan1 netns net1
# ip link set macvlan2 netns net2

```


This creates two new MACVLAN devices in bridge mode. And assigns these two devices to two different namespaces.

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#ipvlan)IPVLAN

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-ipvlan)What is IPVLAN

IPVLAN is similar to MACVLAN with the difference being that the endpoints have the same mac address.

[![IPVLAN](https://liuhangbin.netlify.app/ipvlan.png)](https://liuhangbin.netlify.app/ipvlan.png)

IPVLAN supports L2 and L3 mode. IPVLAN L2 mode acts like a MACVLAN in bridge mode. The parent interface looks like a bridge or switch.

[![IPVLAN L2](https://liuhangbin.netlify.app/ipvlan_01.png)](https://liuhangbin.netlify.app/ipvlan_01.png)

In IPVLAN l3 mode, the parent interface acts like a router and packets are routed between endpoints, which gives better scalability.

[![IPVLAN L3](https://liuhangbin.netlify.app/ipvlan_02.png)](https://liuhangbin.netlify.app/ipvlan_02.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-ipvlan)When to use IPVLAN

Quoting the [IPVLAN kernel doc](https://www.kernel.org/doc/Documentation/networking/ipvlan.txt)</dt>

<dd>MACVLAN and IPVLAN are very similar in many regards and the specific use case could very well define which device to choose. if one of the following situations defines your use case then you can choose to use ipvlan -

*   The Linux host that is connected to the external switch / router has policy configured that allows only one mac per port.
*   No of virtual devices created on a master exceed the mac capacity and puts the NIC in promiscuous mode and degraded performance is a concern.
*   If the slave device is to be put into the hostile / untrusted network namespace where L2 on the slave could be changed / misused.


### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-setup-ipvlan)How to setup IPVLAN

```
# ip netns add ns0
# ip link add name ipv1 link eth0 type ipvlan mode l2
# ip link set dev ipvl0 netns ns0

```


This creates an IPVLAN device named ipvl0 with mode L2, assigned to namespace ns0.

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#macvtapipvtap)MACVTAP/IPVTAP

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-macvtapipvtap)What is MACVTAP/IPVTAP

MACVTAP/IPVTAP is a new device driver meant to simplify virtualized bridged networking. When a MACVTAP/IPVTAP instance is created on top of a physical interface, the kernel also creates a character device /dev/tapX to be used just like a TUN/TAP device, which can be directly used by KVM/QEMU.

with MACVTAP/IPVTAP, we can replace the combination of TUN/TAP and bridge drivers with a single module.

[![MACVTAP](https://liuhangbin.netlify.app/macvtap.png)](https://liuhangbin.netlify.app/macvtap.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-macvtapipvtap)When to use MACVTAP/IPVTAP

Typically, this is used to make both the guest and the host show up directly on the switch to which the host is connected. The difference between MACVTAP and IPVTAP is same with MACVLAN/IPVLAN.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-macvtap)How to create MACVTAP

```
# ip link add link eth0 name macvtap0 type macvtap

```

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#macsec)MACsec

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-macsec)What is MACsec

MACsec is an IEEE standard for security in wired Ethernet LANs. Similiar to IPsec, as a layer 2 specification, MACsec can protect not only IP traffic, but also ARP, neighbour discovery, and DHCP. The MACSEC headers looks like

[![MACSEC Header](https://liuhangbin.netlify.app/macsec_01.png)](https://liuhangbin.netlify.app/macsec_01.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-macsec)When to use MACsec

The main use case for MACsec is to secure all messages on a standard LAN, includeing ARP, NS and DHCP message.

[![MACSEC](https://liuhangbin.netlify.app/macsec.png)](https://liuhangbin.netlify.app/macsec.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-setup-macsec)How to setup MACsec

```
# ip link add macsec0 link eth1 type macsec

```

_Note_: This only adds a MACsec device macsec0 on interface eth1\. For more detailed configurations, please see the “Configuration example” Section in [MACsec introduction by Sabrina](https://developers.redhat.com/blog/2016/10/14/macsec-a-different-solution-to-encrypt-network-traffic/)

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#veth)VETH

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-veth)What is VETH

This device is a local Ethernet tunnel. Devices are created in pairs.

Packets transmitted on one device in the pair are immediately received on the other device. When either device is down the link state of the pair is down.

[![VETH](https://liuhangbin.netlify.app/veth.png)](https://liuhangbin.netlify.app/veth.png)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-veth)When to use VETH

When namespaces need to communicate to the main host namespace, or between each other.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-setup-veth)How to setup VETH

```
# ip netns add net1
# ip netns add net2
# ip link add veth1 netns net1 type veth peer name veth2 netns net2

```

This creates two namespaces net1, net2 and a pair of veth devices, and assigns veth1 to namespace net1, and veth2 to namespace net2\. These two namespaces are connected with this veth pair. Assign a pair of IP addresses, and you can ping and communicate between the two namespaces.

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#vcan)VCAN

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-vcan)What is VCAN

Similar to the network loopback devices, VCAN offers a virtual local CAN (Controller Area Network) interface, so users can send/receive can message via VCAN interface. CAN is mostly used in the automotive field nowadays.

For more CAN protocol information, please refer to [kernel can doc](https://www.kernel.org/doc/Documentation/networking/can.txt)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-can)When to use CAN

When you want to test a CAN protocol implementation on the local host.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-vcan)How to create VCAN

```
# ip link add dev vcan1 type vcan

```

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#vxcan)VXCAN

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-vxcan)what is VXCAN

Similar to the veth(virtual ethernet) driver, vxcan(Virtual CAN Tunnel) implements a local CAN traffic tunnel between two virtual CAN network devices. When creating a vxcan, two VXCAN devices are created as a pair. When one end receives the packet it appears on its pair and vice versa. VXCAN can be used for cross namespace communication.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-vxcan)When to use VXCAN

When you want to send CAN message across namespaces.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-setup-vxcan)How to setup VXCAN

```
# ip netns add net1
# ip netns add net2
# ip link add vxcan1 netns net1 type vxcan peer name vxcan2 netns net2

```

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#ipoib)IPOIB

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-it)What is it

IPOIB a device to support IP-over-InfiniBand protocol. This transports IP packets over InfiniBand so you can use your IB device as a fast NIC.

The IPoIB driver supports two modes of operation: datagram and connected. In datagram mode, the IB UD (Unreliable Datagram) transport is used. In connected mode, the IB RC (Reliable Connected) transport is used. Connected mode takes advantage of the connected nature of the IB transport and allows an MTU up to the maximal IP packet size of 64K.

For more details, please see [ipoib kernel doc](https://www.kernel.org/doc/Documentation/infiniband/ipoib.txt)

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use)When to use

When you have an IB device and want to communicate with a remote host via IP.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create)How to create

```
# ip link add ipoib0 type ipoib mode connected

```

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#nlmon)nlmon

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-nlmon)What is nlmon

nlmon is a netlink monitor device

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-nlmon)When to use nlmon

When you want to monitor system netlink messages

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-nlmon)How to create nlmon

```
# ip link add nlmon0 type nlmon
# ip link set nlmon0 up
# tcpdump -i nlmon0 -w nlmsg.pcap

```

This creates a nlmon device named nlmon0 and sets it up. Use a packet sniffer (e.g. tcpdump) to capture netlink messages. Recent versions of wireshark feature decoding of netlink messages.

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#dummy)Dummy

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-dummy)What is Dummy

The dummy interface is entirely virtual, like e.g. the loopback interface. The purpose of dummy driver is to provide a device to route packets through, without actually transmitting them.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-dummy)When to use Dummy

It was used to make inactive SLIP address look like a real address for local programs. Nowadays it’s mostly used for testing and debugging.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-dummy)How to create Dummy

```
# ip link add dummy1 type dummy
# ip addr add 1.1.1.1/24 dev dummy1
# ip link set dummy1 up

```

## [](https://liuhangbin.netlify.app/post/virtual-ifaces/#ifb)IFB

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#what-is-ifb)What is IFB

The IFB (Intermediate Functional Block) driver supplies a device that allows concentration of traffic from several sources, and shaping incoming traffic instead of dropping it.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#when-to-use-ifb)When to use IFB

When you want to queue and shape incoming traffics.

### [](https://liuhangbin.netlify.app/post/virtual-ifaces/#how-to-create-an-ifb-interface)How to create an IFB interface

IFB is an alternative to tc filters for handling ingress traffic, by redirecting it to a virtual interface and treat is as egress traffic there.You need one ifb interface per physical interface, to redirect ingress traffic from eth0 to ifb0, eth1 to ifb1 and so on.

First, add ifb interfaces.

```
# ip link add ifb0 type ifb
# ip link set ifb0 up

```

And redirect ingress traffic from the physical interfaces to corresponding ifb interface. For eth0 -> ifb0:

```
# tc qdisc add dev eth0 handle ffff: ingress
# tc filter add dev eth0 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0

```

Again, repeat for eth1 -> ifb1, eth2 -> ifb2 and so on, until all the interfaces you want to shape are covered.

Now, you can apply all the rules you want. Egress rules for eth0 go as usual in eth0\. Let’s limit bandwidth, for example:

```
# tc qdisc add dev eth0 root handle 1: htb default 10
# tc class add dev eth0 parent 1: classid 1:1 htb rate 1mbit
# tc class add dev eth0 parent 1:1 classid 1:10 htb rate 1mbit

```

Ingress rules for eth0, now go as egress rules on ifb0 (whatever goes into ifb0 must come out, and only eth0 ingress traffic goes into ifb0). Again, a bandwidth limit example:

```
# tc qdisc add dev ifb0 root handle 1: htb default 10
# tc class add dev ifb0 parent 1: classid 1:1 htb rate 1mbit
# tc class add dev ifb0 parent 1:1 classid 1:10 htb rate 1mbit

```

The advantage of this approach is that egress rules are much more flexible than ingress filters. Filters only allow you to drop packets, not introduce wait times, for example. By handling ingress traffic as egress you can setup queue disciplines, with traffic classes and, if need be, filters. You get access to the whole tc tree, not only simple filters.

For more IFB qdisc use cases, please refer to: [Linux Foundation wiki - IFB](https://wiki.linuxfoundation.org/networking/ifb)

Author Hangbin Liu

LastMod 2019-09-18 [(d761a5f)](https://liuhangbin.netlify.app/commit/d761a5f188b48e70f7974fa427708143f343dce3 "Posts: add new virtual iface and tc usage")
