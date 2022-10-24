- [General advice on monitoring and tuning the Linux networking stack](#general-advice-on-monitoring-and-tuning-the-linux-networking-stack)
- [Overview](#overview)
- [Detailed Look](#detailed-look)
  - [Protocol family registration](#protocol-family-registration)
  - [Sending network data via a socket](#sending-network-data-via-a-socket)
    - [`sock_sendmsg`, `__sock_sendmsg`, and `__sock_sendmsg_nosec`](#sock_sendmsg-__sock_sendmsg-and-__sock_sendmsg_nosec)
    - [`inet_sendmsg`](#inet_sendmsg)
  - [UDP protocol layer](#udp-protocol-layer)
    - [`udp_sendmsg`](#udp_sendmsg)
      - [UDP corking](#udp-corking)
      - [Get the UDP destination address and port](#get-the-udp-destination-address-and-port)
      - [Socket transmit bookkeeping and timestamping](#socket-transmit-bookkeeping-and-timestamping)
      - [Ancillary messages, via sendmsg](#ancillary-messages-via-sendmsg)
      - [Setting custom IP options](#setting-custom-ip-options)
      - [Multicast or unicast?](#multicast-or-unicast)
      - [Routing](#routing)
      - [Prevent the ARP cache from going stale with `MSG_CONFIRM`](#prevent-the-arp-cache-from-going-stale-with-msg_confirm)
      - [Fast path for uncorked UDP sockets: Prepare data for transmit](#fast-path-for-uncorked-udp-sockets-prepare-data-for-transmit)
        - [`ip_make_skb`](#ip_make_skb)
        - [Transmit the data!](#transmit-the-data)
      - [Slow path for corked UDP sockets with no preexisting corked data](#slow-path-for-corked-udp-sockets-with-no-preexisting-corked-data)
        - [`ip_append_data`](#ip_append_data)
        - [`__ip_append_data`](#__ip_append_data)
        - [Flushing corked sockets](#flushing-corked-sockets)
      - [Error accounting](#error-accounting)
    - [`udp_send_skb`](#udp_send_skb)
    - [Monitoring: UDP protocol layer statistics](#monitoring-udp-protocol-layer-statistics)
      - [`/proc/net/snmp`](#procnetsnmp)
      - [`/proc/net/udp`](#procnetudp)
    - [Tuning: Socket send queue memory](#tuning-socket-send-queue-memory)
  - [IP protocol layer](#ip-protocol-layer)
    - [`ip_send_skb`](#ip_send_skb)
    - [`ip_local_out` and `__ip_local_out`](#ip_local_out-and-__ip_local_out)
    - [netfilter and `nf_hook`](#netfilter-and-nf_hook)
    - [Destination cache](#destination-cache)
    - [`ip_output`](#ip_output)
    - [`ip_finish_output`](#ip_finish_output)
      - [Path MTU Discovery](#path-mtu-discovery)
    - [`ip_finish_output2`](#ip_finish_output2)
    - [`dst_neigh_output`](#dst_neigh_output)
    - [`neigh_hh_output`](#neigh_hh_output)
    - [`n->output`](#n-output)
      - [`neigh_resolve_output`](#neigh_resolve_output)
    - [Monitoring: IP protocol layer](#monitoring-ip-protocol-layer)
      - [`/proc/net/snmp`](#procnetsnmp-1)
      - [`/proc/net/netstat`](#procnetnetstat)
  - [Linux netdevice subsystem](#linux-netdevice-subsystem)
    - [Linux traffic control](#linux-traffic-control)
    - [`dev_queue_xmit` and `__dev_queue_xmit`](#dev_queue_xmit-and-__dev_queue_xmit)
      - [`netdev_pick_tx`](#netdev_pick_tx)
      - [`__netdev_pick_tx`](#__netdev_pick_tx)
        - [Transmit Packet Steering (XPS)](#transmit-packet-steering-xps)
        - [`skb_tx_hash`](#skb_tx_hash)
    - [Resuming `__dev_queue_xmit`](#resuming-__dev_queue_xmit)
    - [`__dev_xmit_skb`](#__dev_xmit_skb)
    - [Tuning: Transmit Packet Steering (XPS)](#tuning-transmit-packet-steering-xps)
  - [Queuing disciplines!](#queuing-disciplines)
    - [`qdisc_run_begin` and `qdisc_run_end`](#qdisc_run_begin-and-qdisc_run_end)
    - [`__qdisc_run`](#__qdisc_run)
    - [`qdisc_restart`](#qdisc_restart)
      - [`dequeue_skb`](#dequeue_skb)
      - [`sch_direct_xmit`](#sch_direct_xmit)
      - [`handle_dev_cpu_collision`](#handle_dev_cpu_collision)
      - [`dev_requeue_skb`](#dev_requeue_skb)
    - [Reminder, while loop in `__qdisc_run`](#reminder-while-loop-in-__qdisc_run)
      - [`__netif_schedule`](#__netif_schedule)
      - [`net_tx_action`](#net_tx_action)
        - [`net_tx_action` completion queue](#net_tx_action-completion-queue)
        - [`net_tx_action` output queue](#net_tx_action-output-queue)
    - [Finally time to meet our friend `dev_hard_start_xmit`](#finally-time-to-meet-our-friend-dev_hard_start_xmit)
    - [Monitoring qdiscs](#monitoring-qdiscs)
      - [Using the `tc` command line tool](#using-the-tc-command-line-tool)
    - [Tuning qdiscs](#tuning-qdiscs)
      - [Increasing the processing weight of `__qdisc_run`](#increasing-the-processing-weight-of-__qdisc_run)
      - [Increasing the transmit queue length](#increasing-the-transmit-queue-length)
  - [Network Device Driver](#network-device-driver)
    - [Driver operations registration](#driver-operations-registration)
    - [Transmit data with `ndo_start_xmit`](#transmit-data-with-ndo_start_xmit)
    - [`igb_tx_map`](#igb_tx_map)
      - [Dynamic Queue Limits (DQL)](#dynamic-queue-limits-dql)
    - [Transmit completions](#transmit-completions)
      - [Transmit completion IRQ](#transmit-completion-irq)
      - [`igb_poll`](#igb_poll)
      - [`igb_clean_tx_irq`](#igb_clean_tx_irq)
      - [`igb_poll` return value](#igb_poll-return-value)
    - [Monitoring network devices](#monitoring-network-devices)
      - [Using `ethtool -S`](#using-ethtool--s)
      - [Using sysfs](#using-sysfs)
      - [Using `/proc/net/dev`](#using-procnetdev)
    - [Monitoring dynamic queue limits](#monitoring-dynamic-queue-limits)
    - [Tuning network devices](#tuning-network-devices)
      - [Check the number of TX queues being used](#check-the-number-of-tx-queues-being-used)
      - [Adjust the number of TX queues used](#adjust-the-number-of-tx-queues-used)
      - [Adjust the size of the TX queues](#adjust-the-size-of-the-tx-queues)
  - [The End](#the-end)
- [Extras](#extras)
  - [Reducing ARP traffic (`MSG_CONFIRM`)](#reducing-arp-traffic-msg_confirm)
  - [UDP Corking](#udp-corking-1)
  - [Timestamping](#timestamping)
- [Conclusion](#conclusion)
- [Help with Linux networking or other systems](#help-with-linux-networking-or-other-systems)
- [Related posts](#related-posts)

[原文链接](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/)

This blog post explains how computers running the Linux kernel send packets, as well as how to monitor and tune each component of the networking stack as packets flow from user programs to network hardware.

This post forms a pair with our previous post [Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/).

It is impossible to tune or monitor the Linux networking stack without reading the source code of the kernel and having a deep understanding of what exactly is happening.

This blog post will hopefully serve as a reference to anyone looking to do this.

# General advice on monitoring and tuning the Linux networking stack

As mentioned in our previous article, the Linux network stack is complex and there is no one size fits all solution for monitoring or tuning. If you truly want to tune the network stack, you will have no choice but to invest a considerable amount of time, effort, and money into understanding how the various parts of networking system interact.

Many of the example settings provided in this blog post are used solely for illustrative purposes and are not a recommendation for or against a certain configuration or default setting. Before adjusting any setting, you should develop a frame of reference around what you need to be monitoring to notice a meaningful change.

Adjusting networking settings while connected to the machine over a network is dangerous; you could very easily lock yourself out or completely take out your networking. Do not adjust these settings on production machines; instead, make adjustments on new machines and rotate them into production, if possible.

# Overview

For reference, you may want to have a copy of the device data sheet handy. This post will examine the Intel I350 Ethernet controller, controlled by the `igb` device driver. You can find that data sheet (warning: LARGE PDF) [here for your reference](http://www.intel.com/content/dam/www/public/us/en/documents/datasheets/ethernet-controller-i350-datasheet.pdf).

The high-level path network data takes from a user program to a network device is as follows:

1.  Data is written using a system call (like `sendto`, `sendmsg`, et. al.).
2.  Data passes through the socket subsystem on to the socket’s protocol family’s system (in our case, `AF_INET`).
3.  The protocol family passes data through the protocol layers which (in many cases) arrange the data into packets.
4.  The data passes through the routing layer, populating the destination and neighbour caches along the way (if they are cold). This can generate ARP traffic if an ethernet address needs to be looked up.
5.  After passing through the protocol layers, packets reach the device agnostic layer.
6.  The output queue is chosen using XPS (if enabled) or a hash function.
7.  The device driver’s transmit function is called.
8.  The data is then passed on to the queue discipline (qdisc) attached to the output device.
9.  The qdisc will either transmit the data directly if it can, or queue it up to be sent during the `NET_TX` softirq.
10.  Eventually the data is handed down to the driver from the qdisc.
11.  The driver creates the needed DMA mappings so the device can read the data from RAM.
12.  The driver signals the device that the data is ready to be transmit.
13.  The device fetches the data from RAM and transmits it.
14.  Once transmission is complete, the device raises an interrupt to signal transmit completion.
15.  The driver’s registered IRQ handler for transmit completion runs. For many devices, this handler simply triggers the NAPI poll loop to start running via the `NET_RX` softirq.
16.  The poll function runs via a softIRQ and calls down into the driver to unmap DMA regions and free packet data.

This entire flow will be examined in detail in the following sections.

The protocol layers examined below are the IP and UDP protocol layers. Much of the information presented will serve as a reference for other protocol layers, as well.

# Detailed Look

This blog post will be examining the Linux kernel version 3.13.0 with links to code on GitHub and code snippets throughout this post, much like [the companion post](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/).

Let’s begin by examining how protocol families are registered in the kernel and used by the socket subsystem, then we can proceed to receiving data.

## Protocol family registration

What happens when you run a piece of code like this in a user program to create a UDP socket?

```c
sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP)
```

In short, the Linux kernel looks up a set of functions exported by the UDP protocol stack that deal with many things including sending and receiving network data. To understand exactly how this work, we have to look into the `AF_INET` address family code.

The Linux kernel executes the `inet_init` function early during kernel initialization. This function registers the `AF_INET` protocol family, the individual protocol stacks within that family (TCP, UDP, ICMP, and RAW), and calls initialization routines to get protocol stacks ready to process network data. You can find the code for `inet_init` in [./net/ipv4/af_inet.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/af_inet.c#L1678-L1804).

The `AF_INET` protocol family exports a structure that has a `create` function. This function is called by the kernel when a socket is created from a user program:

```c
static const struct net_proto_family inet_family_ops = {
        .family = PF_INET,
        .create = inet_create,
        .owner  = THIS_MODULE,
};
```

The `inet_create` function takes the arguments passed to the socket system call and searches the registered protocols to find a set of operations to link to the socket. Take a look:

```c
        /* Look for the requested type/protocol pair. */
lookup_protocol:
        err = -ESOCKTNOSUPPORT;
        rcu_read_lock();
        list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {

                err = 0;
                /* Check the non-wild match. */
                if (protocol == answer->protocol) {
                        if (protocol != IPPROTO_IP)
                                break;
                } else {
                        /* Check for the two wild cases. */
                        if (IPPROTO_IP == protocol) {
                                protocol = answer->protocol;
                                break;
                        }
                        if (IPPROTO_IP == answer->protocol)
                                break;
                }
                err = -EPROTONOSUPPORT;
        }
```

Later, `answer` which holds a reference to a particular protocol stack has its `ops` fields copied into the socket structure:

```c
sock->ops = answer->ops;
```

You can find the structure definitions for all of the protocol stacks in `af_inet.c`. Let’s take a look at the [TCP and UDP protocol structures](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/af_inet.c#L998-L1020):

```c
/* Upon startup we insert all the elements in inetsw_array[] into
 * the linked list inetsw.
 */
static struct inet_protosw inetsw_array[] =
{
        {
                .type =       SOCK_STREAM,
                .protocol =   IPPROTO_TCP,
                .prot =       &tcp_prot,
                .ops =        &inet_stream_ops,
                .no_check =   0,
                .flags =      INET_PROTOSW_PERMANENT |
                              INET_PROTOSW_ICSK,
        },

        {
                .type =       SOCK_DGRAM,
                .protocol =   IPPROTO_UDP,
                .prot =       &udp_prot,
                .ops =        &inet_dgram_ops,
                .no_check =   UDP_CSUM_DEFAULT,
                .flags =      INET_PROTOSW_PERMANENT,
       },

            /* .... more protocols ... */
```

In the case of `IPPROTO_UDP`, an `ops` structure is linked into place [which contains functions for various things](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/af_inet.c#L935-L960), including sending and receiving data:

```c
const struct proto_ops inet_dgram_ops = {
  .family           = PF_INET,
  .owner           = THIS_MODULE,

  /* ... */

  .sendmsg       = inet_sendmsg,
  .recvmsg       = inet_recvmsg,

  /* ... */
};
EXPORT_SYMBOL(inet_dgram_ops);
```

and a protocol-specific structure `prot`, which contains function pointers to all the internal UDP protocol stack function. For the UDP protocol, this structure is called `udp_prot` and is exported by [./net/ipv4/udp.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/udp.c#L2171-L2203):

```c
struct proto udp_prot = {
  .name           = "UDP",
  .owner           = THIS_MODULE,

  /* ... */

  .sendmsg       = udp_sendmsg,
  .recvmsg       = udp_recvmsg,

  /* ... */
};
EXPORT_SYMBOL(udp_prot);
```

Now, let’s turn to a user program that sends UDP data to see how `udp_sendmsg` is called in the kernel!

## Sending network data via a socket

A user program wants to send UDP network data and so it uses the `sendto` system call, maybe like this:

```c
ret = sendto(socket, buffer, buflen, 0, &dest, sizeof(dest));
```

This system call passes through the [Linux system call](https://blog.packagecloud.io/eng/2016/04/05/the-definitive-guide-to-linux-system-calls/) layer and lands [in this function](https://github.com/torvalds/linux/blob/v3.13/net/socket.c#L1756-L1803) in `./net/socket.c`:

```c
/*
 *      Send a datagram to a given address. We move the address into kernel
 *      space and check the user space data area is readable before invoking
 *      the protocol.
 */

SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
                unsigned int, flags, struct sockaddr __user *, addr,
                int, addr_len)
{
    /*  ... code ... */

    err = sock_sendmsg(sock, &msg, len);

    /* ... code  ... */
}
```

The `SYSCALL_DEFINE6` macro unfolds [into a pile of macros](https://github.com/torvalds/linux/blob/v3.13/include/linux/syscalls.h#L170-L198), which in turn, set up the infrastructure needed to create a system call with 6 arguments (hence `DEFINE6`). One of the results of this is that inside the kernel, system call function names have `sys_` prepended to them.

The system call code for `sendto` calls `sock_sendmsg` after arranging the data in a way that the lower layers will be able to handle. In particular, it takes the destination address passed into `sendto` and arranges it into a structure, let’s take a look:

```c
  iov.iov_base = buff;
  iov.iov_len = len;
  msg.msg_name = NULL;
  msg.msg_iov = &iov;
  msg.msg_iovlen = 1;
  msg.msg_control = NULL;
  msg.msg_controllen = 0;
  msg.msg_namelen = 0;
  if (addr) {
          err = move_addr_to_kernel(addr, addr_len, &address);
          if (err < 0)
                  goto out_put;
          msg.msg_name = (struct sockaddr *)&address;
          msg.msg_namelen = addr_len;
  }
```

This code is copying `addr`, passed in via the user program into the kernel data structure `address`, which is then embedded into a `struct msghdr` structure as `msg_name`. This is similar to what a userland program would do if it were calling `sendmsg` instead of `sendto`. The kernel provides this mutation because both `sendto` and `sendmsg` do call down to `sock_sendmsg`.

### `sock_sendmsg`, `__sock_sendmsg`, and `__sock_sendmsg_nosec`

`sock_sendmsg` performs some error checking before calling `__sock_sendmsg` does its own error checking before calling `__sock_sendmsg_nosec`. `__sock_sendmsg_nosec` passes the data deeper into the socket subsystem:

```c
static inline int __sock_sendmsg_nosec(struct kiocb *iocb, struct socket *sock,
                                       struct msghdr *msg, size_t size)
{
        struct sock_iocb *si =  ....

                /* other code ... */

        return sock->ops->sendmsg(iocb, sock, msg, size);
}
```

As seen in the previous section explaining socket creation, the `sendmsg` function registered to this socket ops structure is `inet_sendmsg`.

### `inet_sendmsg`

As you may have guessed from the name, this is a generic function provided by the `AF_INET` protocol family. This function starts by calling `sock_rps_record_flow` to record the last CPU that the flow was processed on; this is used by [Receive Packet Steering](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#receive-packet-steering-rps). Next, this function looks up the `sendmsg` function on the socket’s internal protocol operations structure and calls it:

```c
int inet_sendmsg(struct kiocb *iocb, struct socket *sock, struct msghdr *msg,
                 size_t size)
{
  struct sock *sk = sock->sk;

  sock_rps_record_flow(sk);

  /* We may need to bind the socket. */
  if (!inet_sk(sk)->inet_num && !sk->sk_prot->no_autobind &&
      inet_autobind(sk))
          return -EAGAIN;

  return sk->sk_prot->sendmsg(iocb, sk, msg, size);
}
EXPORT_SYMBOL(inet_sendmsg);
```

When dealing with UDP, `sk->sk_prot->sendmsg` above is `udp_sendmsg` as exported by the UDP protocol layer, via the `udp_prot` structure we saw earlier. This function call transitions from the generic `AF_INET` protocol family on to the UDP protocol stack.

## UDP protocol layer

### `udp_sendmsg`

The `udp_sendmsg` function can be found in [./net/ipv4/udp.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/udp.c#L845-L1088). The entire function is quite long, so we’ll examine pieces of it below. Follow the previous link if you’d like to read it in its entirety.

#### UDP corking

After variable declarations and some basic error checking, one of the first things `udp_sendmsg` does is check if the socket is “corked”. UDP corking is a feature that allows a user program request that the kernel accumulate data from multiple calls to `send` into a single datagram before sending. There are two ways to enable this option in your user program:

1.  Use the `setsockopt` system call and pass `UDP_CORK` as the socket option.
2.  Pass `MSG_MORE` as one of the `flags` when calling `send`, `sendto`, or `sendmsg` from your program.

These options are documented in the [UDP man page](http://man7.org/linux/man-pages/man7/udp.7.html) and the [send / sendto / sendmsg man page](http://man7.org/linux/man-pages/man2/send.2.html), respectively.

The code from `udp_sendmsg` checks `up->pending` to determine if the socket is currently corked, and if so, it proceeds directly to appending data. We’ll see how data is appended later.

```c
int udp_sendmsg(struct kiocb *iocb, struct sock *sk, struct msghdr *msg,
                size_t len)
{

    /* variables and error checking ... */

  fl4 = &inet->cork.fl.u.ip4;
  if (up->pending) {
          /*
           * There are pending frames.
           * The socket lock must be held while it's corked.
           */
          lock_sock(sk);
          if (likely(up->pending)) {
                  if (unlikely(up->pending != AF_INET)) {
                          release_sock(sk);
                          return -EINVAL;
                  }
                  goto do_append_data;
          }
          release_sock(sk);
  }
```

#### Get the UDP destination address and port

Next, the destination address and port are determined from one of two possible sources:

1.  The socket itself has the destination address stored because the socket was connected at some point.
2.  The address is passed in via an auxiliary structure, as we saw in the kernel code for `sendto`.

Here’s how the kernel deals with this:

```c
  /*
   *      Get and verify the address.
   */
  if (msg->msg_name) {
          struct sockaddr_in *usin = (struct sockaddr_in *)msg->msg_name;
          if (msg->msg_namelen < sizeof(*usin))
                  return -EINVAL;
          if (usin->sin_family != AF_INET) {
                  if (usin->sin_family != AF_UNSPEC)
                          return -EAFNOSUPPORT;
          }

          daddr = usin->sin_addr.s_addr;
          dport = usin->sin_port;
          if (dport == 0)
                  return -EINVAL;
  } else {
          if (sk->sk_state != TCP_ESTABLISHED)
                  return -EDESTADDRREQ;
          daddr = inet->inet_daddr;
          dport = inet->inet_dport;
          /* Open fast path for connected socket.
             Route will not be used, if at least one option is set.
           */
          connected = 1;
  }
```

Yes, that is a `TCP_ESTABLISHED` in the UDP protocol layer! The socket states for better or worse use TCP state descriptions.

Recall earlier that we saw how the kernel arranges a `struct msghdr` structure on behalf of the user when the user program calls `sendto`. The code above shows how the kernel parses that data back out in order to set `daddr` and `dport`.

If the `udp_sendmsg` function was reached by kernel function which did _not_ arrange a `struct msghdr` structure, the destination address and port are retrieved from the socket itself and the socket is marked as “connected.”

In either case `daddr` and `dport` will be set to the destination address and port.

#### Socket transmit bookkeeping and timestamping
Next, the source address, device index, and any timestamping options which were set on the socket (like `SOCK_TIMESTAMPING_TX_HARDWARE`, `SOCK_TIMESTAMPING_TX_SOFTWARE`, `SOCK_WIFI_STATUS`) are retrieved and stored:

```c
ipc.addr = inet->inet_saddr;

ipc.oif = sk->sk_bound_dev_if;

sock_tx_timestamp(sk, &ipc.tx_flags);
```

#### Ancillary messages, via sendmsg

The `sendmsg` and `recvmsg` system calls allow the user to set or request ancillary data in addition to sending or receiving packets. User programs can make use of this ancillary data by crafting a `struct msghdr` with the request embedded in it. Many of the ancillary data types are documented in the [man page for IP](http://man7.org/linux/man-pages/man7/ip.7.html).

One popular example of ancillary data is `IP_PKTINFO`. In the case of `sendmsg` this data type allows the program to set a `struct in_pktinfo` to be used when sending data. The program can specify the source address to be used on the packet by filling in fields in the `struct in_pktinfo` structure. This is a useful option if the program is a server program listening on multiple IP addresses. In this case, the server program may want to reply to the client with the same IP address that the client used to contact the server. `IP_PKTINFO` enables precisely this use case.

Similarly, the `IP_TTL` and `IP_TOS` ancillary messages allow the user to set the IP packet [TTL](https://en.wikipedia.org/wiki/Time_to_live#IP_packets) and [TOS](https://en.wikipedia.org/wiki/Type_of_service) values on a per-packet basis, when passed with data to `sendmsg` from the user program. Note that both `IP_TTL` and `IP_TOS` may be set at the socket level for all outgoing packets by using `setsockopt`, instead of on a per-packet basis if desired. The Linux kernel translates the TOS value specified to a priority [using an array](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/route.c#L179-L197). The priority affects how and when a packet is transmit from a queuing discipline. We’ll see more about what this means later.

We can see how the kernel handles ancillary messages for `sendmsg` on UDP sockets:

```c
if (msg->msg_controllen) {
        err = ip_cmsg_send(sock_net(sk), msg, &ipc,
                           sk->sk_family == AF_INET6);
        if (err)
                return err;
        if (ipc.opt)
                free = 1;
        connected = 0;
}
```

The internals of parsing the ancillary messages is handled by `ip_cmsg_send` from [./net/ipv4/ip_sockglue.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/ip_sockglue.c#L190-L241). Note that simply providing any ancillary data marks this socket as not connected.

#### Setting custom IP options

Next, `sendmsg` will check to see if the user specified any custom IP options with ancillary messages. If options were set, they will be used. If not, the options already in use by this socket will be used:

```c
if (!ipc.opt) {
        struct ip_options_rcu *inet_opt;

        rcu_read_lock();
        inet_opt = rcu_dereference(inet->inet_opt);
        if (inet_opt) {
                memcpy(&opt_copy, inet_opt,
                       sizeof(*inet_opt) + inet_opt->opt.optlen);
                ipc.opt = &opt_copy.opt;
        }
        rcu_read_unlock();
}
```

Next up, the function checks to see if the source record route (SRR) IP option is set. There are two types of source record routing: [loose and strict source record routing](https://en.wikipedia.org/wiki/Loose_Source_Routing). If this option was set, the first hop address is recorded and stored as `faddr` and the socket is marked as “not connected”. This will be used later:

```c
ipc.addr = faddr = daddr;

if (ipc.opt && ipc.opt->opt.srr) {
        if (!daddr)
                return -EINVAL;
        faddr = ipc.opt->opt.faddr;
        connected = 0;
}
```

After the SRR option is handled, the TOS IP flag is retrieved either from the value the user set via an ancillary message or the value currently in use by the socket. Followed by a check to determine if:

*   `SO_DONTROUTE` was set on the socket (with `setsockopt`), or
*   `MSG_DONTROUTE` was specified as a flag when calling `sendto` or `sendmsg`, or
*   `is_strictroute` was set, indicating that strict [source record routing](http://www.networksorcery.com/enp/protocol/ip/option009.htm) is desired

Then, the `tos` has `0x1` (`RTO_ONLINK`) added to its bit set and the socket is considered not “connected”:

```c
tos = get_rttos(&ipc, inet);
if (sock_flag(sk, SOCK_LOCALROUTE) ||
    (msg->msg_flags & MSG_DONTROUTE) ||
    (ipc.opt && ipc.opt->opt.is_strictroute)) {
        tos |= RTO_ONLINK;
        connected = 0;
}
```

#### Multicast or unicast?

Next, the code attempts to deal with multicast. This is a bit tricky, as the user could specify an alternate source address or device index of where to send the packet from by sending an ancillary `IP_PKTINFO` message, as explained earlier.

If the destination address is a multicast address:

1.  The device index of where to write the packet will be set to the multicast device index, and
2.  The source address on the packet will be set to the multicast source address.

Unless, the user has not overridden the device index by sending the `IP_PKTINFO` ancillary message. Let’s take a look:

```c
if (ipv4_is_multicast(daddr)) {
        if (!ipc.oif)
                ipc.oif = inet->mc_index;
        if (!saddr)
                saddr = inet->mc_addr;
        connected = 0;
} else if (!ipc.oif)
        ipc.oif = inet->uc_index;
```

If the destination address is not a multicast address, the device index is set unless it was overridden by the user with `IP_PKTINFO`.

#### Routing

Now it’s time for routing!

The code in the UDP layer that deals with routing begins with a fast path. If the socket is connected try to get the routing structure:

```c
if (connected)
        rt = (struct rtable *)sk_dst_check(sk, 0);
```

If the socket was not connected, or if it was but the routing helper `sk_dst_check` decided the route was obsolete the code moves into the slow path to generate a routing structure. This begins by calling `flowi4_init_output` to construct a structure describing this UDP flow:

```c
if (rt == NULL) {
        struct net *net = sock_net(sk);

        fl4 = &fl4_stack;
        flowi4_init_output(fl4, ipc.oif, sk->sk_mark, tos,
                           RT_SCOPE_UNIVERSE, sk->sk_protocol,
                           inet_sk_flowi_flags(sk)|FLOWI_FLAG_CAN_SLEEP,
                           faddr, saddr, dport, inet->inet_sport);
```

Once this flow structure has been constructed, the socket and its flow structure are passed along to the security subsystem so that systems like [SELinux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux) or [SMACK](https://en.wikipedia.org/wiki/Smack_(software)) can set a security id value on the flow structure. Next, `ip_route_output_flow` will call into the IP routing code to generate a routing structure for this flow:

```c
security_sk_classify_flow(sk, flowi4_to_flowi(fl4));
rt = ip_route_output_flow(net, fl4, sk);
```

If a routing structure could not be generated and the error was `ENETUNREACH`, the `OUTNOROUTES` statistic counter is incremented.

```c
if (IS_ERR(rt)) {
  err = PTR_ERR(rt);
  rt = NULL;
  if (err == -ENETUNREACH)
    IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
  goto out;
}
```

The location of the file holding these statistics counter and the other available counters and their meanings will be discussed below in the UDP monitoring section.

Next, if the route is for broadcast, but the socket option `SOCK_BROADCAST` was not set on the socket the code terminates. If the socket is considered “connected” (as described throughout this function), the routing structure is cached on the socket:

```c
err = -EACCES;
if ((rt->rt_flags & RTCF_BROADCAST) &&
    !sock_flag(sk, SOCK_BROADCAST))
        goto out;
if (connected)
        sk_dst_set(sk, dst_clone(&rt->dst));
```

#### Prevent the ARP cache from going stale with `MSG_CONFIRM`

If the user specified the `MSG_CONFIRM` flag when calling `send`, `sendto`, or `sendmsg`, the UDP protocol layer will now handle that:

```c
  if (msg->msg_flags&MSG_CONFIRM)
          goto do_confirm;
back_from_confirm:
```

This flag indicates to the system to confirm that the ARP cache entry is still valid and prevents it from being garbage collected. The `dst_confirm` function simply sets a flag on destination cache entry which will be checked much later when the neighbour cache has been queried and an entry has been found. We’ll see this again later. This feature is commonly used in UDP networking applications to reduce unnecessary ARP traffic. The `do_confirm` label is found near the end of this function, but it is straightforward:

```c
do_confirm:
        dst_confirm(&rt->dst);
        if (!(msg->msg_flags&MSG_PROBE) || len)
                goto back_from_confirm;
        err = 0;
        goto out;
```

This code confirms the cache entry and jumps back to `back_from_confirm`, if this was not a probe.

Once the `do_confirm` code jumps back to `back_from_confirm` (or no jump happened to `do_confirm` in the first place), the code will attempt to deal with both the UDP cork and uncorked cases next.

#### Fast path for uncorked UDP sockets: Prepare data for transmit

If UDP corking is not requested, the data can be packed into a `struct sk_buff` and passed on to `udp_send_skb` to move down the stack and closer to the IP protocol layer. This is done by calling `ip_make_skb`. Note that the routing structure generated earlier by calling `ip_route_output_flow` is passed in as well. It will be affixed to the skb and used later in the IP protocol layer.

```c
/* Lockless fast path for the non-corking case. */
if (!corkreq) {
        skb = ip_make_skb(sk, fl4, getfrag, msg->msg_iov, ulen,
                          sizeof(struct udphdr), &ipc, &rt,
                          msg->msg_flags);
        err = PTR_ERR(skb);
        if (!IS_ERR_OR_NULL(skb))
                err = udp_send_skb(skb, fl4);
        goto out;
}
```

The `ip_make_skb` function will attempt to construct an skb taking into consideration a wide range of things, like:

*   The [MTU](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/Maximum_transmission_unit).
*   UDP corking (if enabled).
*   UDP Fragmentation Offloading ([UFO](https://wiki.linuxfoundation.org/networking/ufo)).
*   Fragmentation, if UFO is unsupported and the size of the data to transmit is larger than the MTU.

Most network device drivers do not support UFO because the network hardware itself does not support this feature. Let’s take a look through this code, keeping in mind that corking is disabled. We’ll look at the corking enabled path next.

##### `ip_make_skb`

The `ip_make_skb` function can be found in [./net/ipv4/ip_output.c](https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/LINK). This function is a bit tricky. The lower level code that `ip_make_skb` needs to use in order to build an skb requires a corking structure and queue where the skb will be queued to be passed in. In the case where the socket is not corked, a faux corking structure and empty queue are passed in as dummies.

Let’s take a look at how the faux corking structure and queue are setup:

```c
struct sk_buff *ip_make_skb(struct sock *sk, /* more args */)
{
        struct inet_cork cork;
        struct sk_buff_head queue;
        int err;

        if (flags & MSG_PROBE)
                return NULL;

        __skb_queue_head_init(&queue);

        cork.flags = 0;
        cork.addr = 0;
        cork.opt = NULL;
        err = ip_setup_cork(sk, &cork, /* more args */);
        if (err)
                return ERR_PTR(err);
```

As seen above, both the corking structure (`cork`) and the queue (`queue`) are stack-allocated; neither are needed by the time `ip_make_skb` has completed. The faux corking structure is setup with a call to `ip_setup_cork` which allocates memory and initializes the structure. Next, `__ip_append_data` is called and the queue and corking structure are passed in:

```c
err = __ip_append_data(sk, fl4, &queue, &cork,
                       &current->task_frag, getfrag,
                       from, length, transhdrlen, flags);
```

We’ll see how this function works later, as it is used in both cases whether the socket is corked or not. For now, all we need to know is that `__ip_append_data` will create an skb, append data to it, and add that skb to the queue passed in. If appending the data failed, `__ip_flush_pending_frame` is called to drop the data on the floor and the error code is passed back upward:

```c
if (err) {
        __ip_flush_pending_frames(sk, &queue, &cork);
        return ERR_PTR(err);
}
```

Finally, if no error occurred, `__ip_make_skb` will dequeue the queued skb, add the IP options, and return an skb that is ready to be passed on to lower layers for sending:

```c
return __ip_make_skb(sk, fl4, &queue, &cork);
```

##### Transmit the data!

If no errors occurred, the skb is handed to `udp_send_skb` which will pass the skb to the next layer of the networking stack, the IP protocol stack:

```c
err = PTR_ERR(skb);
if (!IS_ERR_OR_NULL(skb))
        err = udp_send_skb(skb, fl4);
goto out;
```

If there was an error, it will be accounted later. See the “Error Accounting” section below the UDP corking case for more information.

#### Slow path for corked UDP sockets with no preexisting corked data

If UDP corking is being used, but no preexisting data is corked, the slow path commences:

1.  Lock the socket.
2.  Check for an application bug: a corked socket that is being “re-corked”.
3.  The flow structure for this UDP flow is prepared for corking.
4.  The data to be sent is appended to existing data.

You can see this in the next piece of code, continuing down `udp_sendmsg`:

```c
  lock_sock(sk);
  if (unlikely(up->pending)) {
          /* The socket is already corked while preparing it. */
          /* ... which is an evident application bug. --ANK */
          release_sock(sk);

          LIMIT_NETDEBUG(KERN_DEBUG pr_fmt("cork app bug 2\n"));
          err = -EINVAL;
          goto out;
  }
  /*
   *      Now cork the socket to pend data.
   */
  fl4 = &inet->cork.fl.u.ip4;
  fl4->daddr = daddr;
  fl4->saddr = saddr;
  fl4->fl4_dport = dport;
  fl4->fl4_sport = inet->inet_sport;
  up->pending = AF_INET;

do_append_data:
  up->len += ulen;
  err = ip_append_data(sk, fl4, getfrag, msg->msg_iov, ulen,
                       sizeof(struct udphdr), &ipc, &rt,
                       corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);
```

##### `ip_append_data`

The `ip_append_data` is a small wrapper function which does two major things prior to calling down to `__ip__append_data`:

1.  Checks if the `MSG_PROBE` flag was passed in from the user. This flag indicates that the user does not want to really send data. The path should be probed (for example to determine the [PMTU](https://en.wikipedia.org/wiki/Path_MTU_Discovery)).
2.  Checks if the socket’s send queue is empty. If so, this means that there is no corked data pending, so `ip_setup_cork` is called to setup corking.

Once the above conditions are dealt with the `__ip_append_data` function is called which contains the bulk of the logic for processing data into packets.

##### `__ip_append_data`

This function is called in either from `ip_append_data` if the socket is corked or from `ip_make_skb` if the socket is not corked. In either case, this function will either allocate a new buffer to store the data passed in or will append the data with existing data.

The way this work centers around the socket’s send queue. Existing data waiting to be sent (for example, if the socket is corked) will have an entry in the queue where additional data can be appended.

This function is complex; it performs several rounds of calculations to determine how to construct the skb that will be passed to the lower level networking layers and examining the buffer allocation process in detail is not strictly necessary for understanding how network data is transmit.

The important highlights of this function include:

1.  Handling UDP fragmentation offloading (UFO), if supported by the hardware. The vast majority of network hardware does not support UFO. If your network card’s driver does support it, it will set the feature flag `NETIF_F_UFO`.
2.  Handling network cards that support [scatter/gather IO](https://en.wikipedia.org/wiki/Vectored_I/O). Many cards support this and it is advertised with the `NETIF_F_SG` feature flag. The availability of this feature indicates that a network card can deal with transmitting a packet where the data has been split amongst a set of buffers; the kernel does not need to spend time coalescing multiple buffers into a single buffer. Avoiding this additional copying is desired and most network cards support this.
3.  Tracking the size of the send queue via calls to `sock_wmalloc`. When a new skb is allocated, the size of the skb is charged to the socket which owns it and the allocated bytes for a socket’s send queue are incremented. If there was not sufficient space in the send queue, the skb is not allocated and an error is returned and tracked. We’ll see how to set the socket send queue size in the tuning section below.
4.  Incrementing error statistics. Any error in this function increments “discard”. We’ll see how to read this value in the monitoring section below.

Upon successful completion of this function, `0` is returned and the data to be transmit will be assembled into an skb that is appropriate for the network device and is waiting on the send queue.

In the uncorked case, the queue holding the skb is passed to `__ip_make_skb` described above where it is dequeued and prepared to be sent to the lower layers via `udp_send_skb`.

In the corked case, the return value of `__ip_append_data` is passed upward. The data sits on the send queue until `udp_sendmsg` determines it is time to call `udp_push_pending_frames` which will finalize the skb and call `udp_send_skb`.

##### Flushing corked sockets

Now, `udp_sendmsg` will move on to check the return value (`err` below) from `__ip_append_skb`:

```c
if (err)
        udp_flush_pending_frames(sk);
else if (!corkreq)
        err = udp_push_pending_frames(sk);
else if (unlikely(skb_queue_empty(&sk->sk_write_queue)))
        up->pending = 0;
release_sock(sk);
```

Let’s take a look at each of these cases:

1.  If there is an error (`err` is non-zero), then `udp_flush_pending_frames` is called, which cancels corking and drops all data from the socket’s send queue.
2.  If this data was sent without `MSG_MORE` specified, called `udp_push_pending_frames` which will attempt to deliver the data to the lower networking layers.
3.  If the send queue is empty, mark the socket as no longer corking.

If the append operation completed successfully and there is more data to cork coming, the code continues by cleaning up and returning the length of the data appended:

```c
ip_rt_put(rt);
if (free)
        kfree(ipc.opt);
if (!err)
        return len;
```

That is how the kernel deals with corked UDP sockets.

#### Error accounting

If:

1.  The non-corking fast path failed to make an skb or `udp_send_skb` reports an error, or
2.  `ip_append_data` fails to append data to a corked UDP socket, or
3.  `udp_push_pending_frames` returns an error received from `udp_send_skb` when trying to transmit a corked skb

the `SNDBUFERRORS` statistic will be incremented only if the error received was `ENOBUFS` (no kernel memory available) or the socket has `SOCK_NOSPACE` set (the send queue is full):

```c
/*
 * ENOBUFS = no kernel mem, SOCK_NOSPACE = no sndbuf space.  Reporting
 * ENOBUFS might not be good (it's not tunable per se), but otherwise
 * we don't have a good statistic (IpOutDiscards but it can be too many
 * things).  We could add another new stat but at least for now that
 * seems like overkill.
 */
if (err == -ENOBUFS || test_bit(SOCK_NOSPACE, &sk->sk_socket->flags)) {
        UDP_INC_STATS_USER(sock_net(sk),
                        UDP_MIB_SNDBUFERRORS, is_udplite);
}
return err;
```

We’ll see how to read these counters in the monitoring section below.

### `udp_send_skb`

The [`udp_send_skb` function](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/udp.c#L765-L819) is how `udp_sendmsg` will eventually push an skb down to the next layer of the networking stack, in this case the IP protocol layer. This function does a few important things:

1.  Adds a UDP header to the skb.
2.  Deals with checksums: software checksums, hardware checksums, or no checksum (if disabled).
3.  Attempts to send the skb to the IP protocol layer by calling `ip_send_skb`.
4.  Increments statistics counters for successful or failed transmissions.

Let’s take a look. First, a UDP header is created:

```c
static int udp_send_skb(struct sk_buff *skb, struct flowi4 *fl4)
{
        /* useful variables ... */

        /*
         * Create a UDP header
         */
        uh = udp_hdr(skb);
        uh->source = inet->inet_sport;
        uh->dest = fl4->fl4_dport;
        uh->len = htons(len);
        uh->check = 0;
```

Next, checksumming is handled. There’s a few cases:

1.  [UDP-Lite](https://en.wikipedia.org/wiki/UDP-Lite) checksums are handled first.
2.  Next, if the socket is set to not generate checksums at all (via `setsockopt` with `SO_NO_CHECK`), it will be marked as such.
3.  Next, if the hardware supports UDP checksums, `udp4_hwcsum` will be called to set that up. Note that the kernel will generate checksums in software if the packet is fragmented. You can see this in the [source for `udp4_hwcsum`](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/udp.c#L720-L763).
4.  Lastly, a software checksum is generated with a call to `udp_csum`.

```c
if (is_udplite)                                  /*     UDP-Lite      */
        csum = udplite_csum(skb);

else if (sk->sk_no_check == UDP_CSUM_NOXMIT) {   /* UDP csum disabled */

        skb->ip_summed = CHECKSUM_NONE;
        goto send;

} else if (skb->ip_summed == CHECKSUM_PARTIAL) { /* UDP hardware csum */

        udp4_hwcsum(skb, fl4->saddr, fl4->daddr);
        goto send;

} else
        csum = udp_csum(skb);
```

Next, the [psuedo header](https://en.wikipedia.org/wiki/User_Datagram_Protocol#IPv4_Pseudo_Header) is added:

```c
uh->check = csum_tcpudp_magic(fl4->saddr, fl4->daddr, len,
                              sk->sk_protocol, csum);
if (uh->check == 0)
        uh->check = CSUM_MANGLED_0;
```

If the checksum is 0, the equivalent in one’s complement is set as the checksum, per [RFC 768](https://tools.ietf.org/html/rfc768). Finally, the skb is passed to the IP protocol stack and statistics are incremented:

```c
send:
  err = ip_send_skb(sock_net(sk), skb);
  if (err) {
          if (err == -ENOBUFS && !inet->recverr) {
                  UDP_INC_STATS_USER(sock_net(sk),
                                     UDP_MIB_SNDBUFERRORS, is_udplite);
                  err = 0;
          }
  } else
          UDP_INC_STATS_USER(sock_net(sk),
                             UDP_MIB_OUTDATAGRAMS, is_udplite);
  return err;
```

If `ip_send_skb` completes successfully, the `OUTDATAGRAMS` statistic is incremented. If the IP protocol layer reports an error, `SNDBUFERRORS` is incremented, but only if the error is `ENOBUFS` (lack of kernel memory) and there is no error queue enabled.

Before moving on to the IP protocol layer, let’s take a look at how to monitor and tune the UDP protocol layer in the Linux kernel.

### Monitoring: UDP protocol layer statistics
Two very useful files for getting UDP protocol statistics are:

*   `/proc/net/snmp`
*   `/proc/net/udp`

#### `/proc/net/snmp`

Monitor detailed UDP protocol statistics by reading `/proc/net/snmp`.

```sh
$ cat /proc/net/snmp | grep Udp\:
Udp: InDatagrams NoPorts InErrors OutDatagrams RcvbufErrors SndbufErrors
Udp: 16314 0 0 17161 0 0
```

In order to understand precisely where these statistics are incremented, you will need to carefully read the kernel source. There are a few cases where some errors are counted in more than one statistic.

*   `InDatagrams`: Incremented when `recvmsg` was used by a userland program to read datagram. Also incremented when a UDP packet is encapsulated and sent back for processing.
*   `NoPorts`: Incremented when UDP packets arrive destined for a port where no program is listening.
*   `InErrors`: Incremented in several cases: no memory in the receive queue, when a bad checksum is seen, and if `sk_add_backlog` fails to add the datagram.
*   `OutDatagrams`: Incremented when a UDP packet is handed down without error to the IP protocol layer to be sent.
*   `RcvbufErrors`: Incremented when `sock_queue_rcv_skb` reports that no memory is available; this happens if `sk->sk_rmem_alloc` is greater than or equal to `sk->sk_rcvbuf`.
*   `SndbufErrors`: Incremented if the IP protocol layer reported an error when trying to send the packet and no error queue has been setup. Also incremented if no send queue space or kernel memory are available.
*   `InCsumErrors`: Incremented when a UDP checksum failure is detected. Note that in all cases I could find, `InCsumErrors` is incremented at the same time as `InErrors`. Thus, `InErrors` - `InCsumErros` should yield the count of memory related errors on the receive side.

Note that some errors discovered by the UDP protocol layer are reported in the statistics files for other protocol layers. One example of this: routing errors. A routing error discovered by `udp_sendmsg` will cause an increment to the IP protocol layer’s `OutNoRoutes` statistic.

#### `/proc/net/udp`

Monitor UDP socket statistics by reading `/proc/net/udp`

```sh
$ cat /proc/net/udp
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode ref pointer drops
  515: 00000000:B346 00000000:0000 07 00000000:00000000 00:00000000 00000000   104        0 7518 2 0000000000000000 0
  558: 00000000:0371 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 7408 2 0000000000000000 0
  588: 0100007F:038F 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 7511 2 0000000000000000 0
  769: 00000000:0044 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 7673 2 0000000000000000 0
  812: 00000000:006F 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 7407 2 0000000000000000 0
```

The first line describes each of the fields in the lines following:

*   `sl`: Kernel hash slot for the socket
*   `local_address`: Hexadecimal local address of the socket and port number, separated by `:`.
*   `rem_address`: Hexadecimal remote address of the socket and port number, separated by `:`.
*   `st`: The state of the socket. Oddly enough, the UDP protocol layer seems to use some TCP socket states. In the example above, `7` is `TCP_CLOSE`.
*   `tx_queue`: The amount of memory allocated in the kernel for outgoing UDP datagrams.
*   `rx_queue`: The amount of memory allocated in the kernel for incoming UDP datagrams.
*   `tr`, `tm->when`, `retrnsmt`: These fields are unused by the UDP protocol layer.
*   `uid`: The effective user id of the user who created this socket.
*   `timeout`: Unused by the UDP protocol layer.
*   `inode`: The inode number corresponding to this socket. You can use this to help you determine which user process has this socket open. Check `/proc/[pid]/fd`, which will contain symlinks to `socket[:inode]`.
*   `ref`: The current reference count for the socket.
*   `pointer`: The memory address in the kernel of the `struct sock`.
*   `drops`: The number of datagram drops associated with this socket. Note that this does not include any drops related to sending datagrams (on corked UDP sockets or otherwise); this is only incremented in receive paths as of the kernel version examined by this blog post.

The code which outputs this can [be found in `net/ipv4/udp.c`](https://github.com/torvalds/linux/blob/master/net/ipv4/udp.c#L2396-L2431).

### Tuning: Socket send queue memory

The maximum size of the send queue (also called the write queue) can be adjusted by setting the `net.core.wmem_max` sysctl.

Increase the maximum send buffer size by setting a `sysctl`.

`$ sudo sysctl -w net.core.wmem_max=8388608`

`sk->sk_write_queue` starts at the `net.core.wmem_default` value, which can also be adjusted by setting a sysctl, like so:

Adjust the default initial send buffer size by setting a `sysctl`.

`$ sudo sysctl -w net.core.wmem_default=8388608`

You can also set the `sk->sk_write_queue` size by calling [`setsockopt`](http://www.manpagez.com/man/2/setsockopt/) from your application and passed `SO_SNDBUF`. The maximum you can set with `setsockopt` is `net.core.wmem_max`.

However, you can override the `net.core.wmem_max` limit by calling `setsockopt` and passing `SO_SNDBUFFORCE`, but the user running the application need the `CAP_NET_ADMIN` capability.

The `sk->sk_wmem_alloc` is incremented each time an skb is allocated by calls to `__ip_append_data`. As we’ll see, UDP datagrams are transmit quickly and typically don’t spend much time in the send queue.

## IP protocol layer

The UDP protocol layer hands skbs down to the IP protocol by simply calling `ip_send_skb`, so let’s start there and map out the IP protocol layer!

### `ip_send_skb`

The `ip_send_skb` function is found in [./net/ipv4/ip_output.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/ip_output.c#L1367-L1380) and is very short. It simply calls down to `ip_local_out` and bumps an error statistic if `ip_local_out` returns an error of some sort. Let’s take a look:

```c
int ip_send_skb(struct net *net, struct sk_buff *skb)
{
        int err;

        err = ip_local_out(skb);
        if (err) {
                if (err > 0)
                        err = net_xmit_errno(err);
                if (err)
                        IP_INC_STATS(net, IPSTATS_MIB_OUTDISCARDS);
        }

        return err;
}
```

As seen above, `ip_local_out` is called and the return value is dealt with after that. The call to `net_xmit_errno` helps to “translate” any errors from lower levels into an error that is understood by the IP and UDP protocol layers. If any error happens, the IP protocol statistic “OutDiscards” is incremented. We’ll see later which files to read to obtain this statistic. For now, let’s continue down the rabbit hole and see where `ip_local_out` takes us.

### `ip_local_out` and `__ip_local_out`

Luckily for us, both `ip_local_out` and `__ip_local_out` are simple. `ip_local_out` simply calls down to `__ip_local_out` and based on the return value, will call into the routing layer to output the packet:

```c
int ip_local_out(struct sk_buff *skb)
{
        int err;

        err = __ip_local_out(skb);
        if (likely(err == 1))
                err = dst_output(skb);

        return err;
}
```

We can see from the source to `__ip_local_out` that the function does two important things first:

1.  Sets the length of the IP packet
2.  Calls `ip_send_check` to compute the checksum to be written in the IP packet header. The `ip_send_check` function will call a function named `ip_fast_csum` to compute the checksum. On the x86 and x86_64 architectures, this function is implemented in assembly. You can read the [64bit implementation here](https://github.com/torvalds/linux/blob/v3.13/arch/x86/include/asm/checksum_64.h#L40-L73) and the [32bit implementation here](https://github.com/torvalds/linux/blob/v3.13/arch/x86/include/asm/checksum_32.h#L63-L98).

Next, the IP protocol layer will call down into netfilter by calling `nf_hook`. The return value of the `nf_hook` function will be passed back up to `ip_local_out`. If `nf_hook` returns `1`, this indicates that the packet was allowed to pass and that the caller should pass it along itself. As we saw above, this is precisely what happens: `ip_local_out` checks for the return value of `1` and passes the packet on by calling `dst_output` itself. Let’s take a look at the code for `__ip_local_out`:

```c
int __ip_local_out(struct sk_buff *skb)
{
        struct iphdr *iph = ip_hdr(skb);

        iph->tot_len = htons(skb->len);
        ip_send_check(iph);
        return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, skb, NULL,
                       skb_dst(skb)->dev, dst_output);
}
```

### netfilter and `nf_hook`

In the interest of brevity (and my RSI), I’ve decided to skip my deep dive into netfilter, iptables, and conntrack. You can dive into the source for netfilter by starting [here](https://github.com/torvalds/linux/blob/v3.13/include/linux/netfilter.h#L142-L147) and [here](https://github.com/torvalds/linux/blob/v3.13/net/netfilter/core.c#L168-L209).

The short version is that `nf_hook` is a wrapper which calls `nf_hook_thresh` that first checks if any filters are installed for the specified protocol family and hook type (`NFPROTO_IPV4` and `NF_INET_LOCAL_OUT` in this case, respectively) and attempt to return execution back to the IP protocol layer to avoid going deeper into netfilter and anything that hooks in below that like iptables and conntrack.

Keep in mind: if you have numerous or very complex netfilter or iptables rules, those rules will be executed in the CPU context of the user process which initiated the original `sendmsg` call. If you have CPU pinning set up to restrict execution of this process to a particular CPU (or set of CPUs), be aware that the CPU will spend system time processing outbound iptables rules. Depending on your system’s workload, you may want to carefully pin processes to CPUs or reduce the complexity of your ruleset if you measure a performance regression here.

For the purposes of our discussion, let’s assume `nf_hook` returns `1` indicating that the caller (in this case, the IP protocol layer) should pass the packet along itself.

### Destination cache
The `dst` code implements the protocol independent destination cache in the Linux kernel. To understand how `dst` entries are setup to proceed with the sending of UDP datagrams, we need to briefly examine how `dst` entries and routes are generated. The destination cache, routing, and neighbour subsystems can all be examined in extreme detail on their own. For our purposes, we can take a quick look to see how this all fits together.

The code we’ve seen above calls `dst_output(skb)`. This function simply looks up the `dst` entry attached to the `skb` and calls the output function. Let’s take a look:

```c
/* Output packet to network from transport.  */
static inline int dst_output(struct sk_buff *skb)
{
        return skb_dst(skb)->output(skb);
}
```

Seems simple enough, but how does that output function get attached to the `dst` entry in the first place?

It’s important to understand that destination cache entries are added in many different ways. One way we’ve seen so far in the code path we’ve been following is with the call to [`ip_route_output_flow`](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/route.c#L2252-L2267) from `udp_sendmsg`. The `ip_route_output_flow` function calls [`__ip_route_output_key`](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/route.c#L1990-L2173) which calls [`__mkroute_output`](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/route.c#L1868-L1988). The `__mkroute_output` function creates the route and the destination cache entry. When it does so, it determines which of the output functions is appropriate for this destination. Most of the time, this function is `ip_output`.

### `ip_output`

So, `dst_output` executes the `output` function, which in the UDP IPv4 case is `ip_output`. The `ip_output` function is straightforward:

```c
int ip_output(struct sk_buff *skb)
{
        struct net_device *dev = skb_dst(skb)->dev;

        IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUT, skb->len);

        skb->dev = dev;
        skb->protocol = htons(ETH_P_IP);

        return NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, skb, NULL, dev,
                            ip_finish_output,
                            !(IPCB(skb)->flags & IPSKB_REROUTED));
}
```

First, a statistics counter is updated `IPSTATS_MIB_OUT`. The `IP_UPD_PO_STATS` macro will increment both the number of bytes and number packets. We’ll see in a later section how to obtain the IP protocol layer statistics and what each of them mean. Next, the device for this `skb` to be transmit on is set, as is the protocol.

Finally, control is passed off to netfilter with a call to `NF_HOOK_COND`. Looking at the function prototype for `NF_HOOK_COND` will help make the explanation of how it works a bit clearer. From [./include/linux/netfilter.h](https://github.com/torvalds/linux/blob/v3.13/include/linux/netfilter.h#L177-L188):

```c
static inline int
NF_HOOK_COND(uint8_t pf, unsigned int hook, struct sk_buff *skb,
             struct net_device *in, struct net_device *out,
             int (*okfn)(struct sk_buff *), bool cond)
```

`NF_HOOK_COND` works by checking the conditional, which is passed in. In this case, that conditional is `!(IPCB(skb)->flags & IPSKB_REROUTED`. If this conditional is true, then the `skb` will be passed on to netfilter. If netfilter allows the packet to pass, the `okfn` is called. In this case, the `okfn` is `ip_finish_output`.

### `ip_finish_output`

The `ip_finish_output` function is also short and clear. Let’s take a look:

```c
static int ip_finish_output(struct sk_buff *skb)
{
#if defined(CONFIG_NETFILTER) && defined(CONFIG_XFRM)
        /* Policy lookup after SNAT yielded a new policy */
        if (skb_dst(skb)->xfrm != NULL) {
                IPCB(skb)->flags |= IPSKB_REROUTED;
                return dst_output(skb);
        }
#endif
        if (skb->len > ip_skb_dst_mtu(skb) && !skb_is_gso(skb))
                return ip_fragment(skb, ip_finish_output2);
        else
                return ip_finish_output2(skb);
}
```

If netfilter and packet transformation are enabled in this kernel, the `skb`’s flags are updated and it is sent back through `dst_output`. The two more common cases are:

1.  If packet’s length is larger than the MTU and the packet’s segmentation will not be offloaded to the device, `ip_fragment` is called to help fragment the packet prior to transmission.
2.  Otherwise, the packet is passed straight through to `ip_finish_output2`.

Let’s take a short detour to talk about Path MTU Discovery before continuing our way through the kernel.

#### Path MTU Discovery

Linux provides a feature I’ve avoided mentioning until now: [Path MTU Discovery](https://en.wikipedia.org/wiki/Path_MTU_Discovery). This feature allows the kernel to automatically determine the largest [MTU](https://en.wikipedia.org/wiki/Maximum_transmission_unit) for a particular route. Determining this value and sending packets that are less than or equal to the MTU for the route means that IP fragmentation can be avoided. This is the preferred setting because fragmenting packets consumes system resources and is seemingly easy to avoid: simply send small enough packets and fragmentation is unnecessary.

You can adjust the Path MTU Discovery settings on a per-socket basis by calling `setsockopt` in your application with the `SOL_IP` level and `IP_MTU_DISCOVER` optname. The optval can be one of the several values described in the [IP protocol man page](http://man7.org/linux/man-pages/man7/ip.7.html). The value you’ll likely want to set is: `IP_PMTUDISC_DO` which means “Always do Path MTU Discovery.” More advanced network applications or diagnostic tools may choose to implement [RFC 4821](https://www.ietf.org/rfc/rfc4821.txt) themselves to determine the PMTU at application start for a particular route or routes. In this case, you can use the `IP_PMTUDISC_PROBE` option which tells the kernel to set the “Don’t Fragment” bit, but allows you to send data larger than the PMTU.

Your application can retrieve the PMTU by calling `getsockopt`, with the `SOL_IP` and `IP_MTU` optname. You can use this to help guide the size of the UDP datagrams your application will construct prior to attempting transmissions.

If you have enabled PTMU discovery, any attempt to send UDP data larger than the PMTU will result in the application receiving the error code `EMSGSIZE`. The application can then retry, but with less data.

Enabling PTMU discovery is strongly encouraged, so I’ll avoid describing the IP fragmentation code path in detail. When we take a look at the IP protocol layer statistics, I’ll explain all the statistics including the fragmentation related statistics. Many of them are incremented in `ip_fragment`. In both the fragment or non-fragment case `ip_finish_output2` is called, so let’s continue there.

### `ip_finish_output2`

The `ip_finish_output2` is called after IP fragmentation and also directly from `ip_finish_output`. This function handles bumping various statistics counters prior to handing the packet down to the neighbour cache. Let’s see how this works:

```c
static inline int ip_finish_output2(struct sk_buff *skb)
{

                /* variable declarations */

        if (rt->rt_type == RTN_MULTICAST) {
                IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTMCAST, skb->len);
        } else if (rt->rt_type == RTN_BROADCAST)
                IP_UPD_PO_STATS(dev_net(dev), IPSTATS_MIB_OUTBCAST, skb->len);

        /* Be paranoid, rather than too clever. */
        if (unlikely(skb_headroom(skb) < hh_len && dev->header_ops)) {
                struct sk_buff *skb2;

                skb2 = skb_realloc_headroom(skb, LL_RESERVED_SPACE(dev));
                if (skb2 == NULL) {
                        kfree_skb(skb);
                        return -ENOMEM;
                }
                if (skb->sk)
                        skb_set_owner_w(skb2, skb->sk);
                consume_skb(skb);
                skb = skb2;
        }
```

If the routing structure associated with this packet is of type multicast, both the `OutMcastPkts` and `OutMcastOctets` counters are bumped by using the `IP_UPD_PO_STATS` macro. Otherwise, if the route type is broadcast the `OutBcastPkts` and `OutBcastOctets` counters are bumped.

Next, a check is performed to ensure that the skb structure has enough room for any link layer headers that need to be added. If not, additional room is allocated with a call to `skb_realloc_headroom` and the cost of the new skb is charged to the associated socket.

```c
        rcu_read_lock_bh();
        nexthop = (__force u32) rt_nexthop(rt, ip_hdr(skb)->daddr);
        neigh = __ipv4_neigh_lookup_noref(dev, nexthop);
        if (unlikely(!neigh))
                neigh = __neigh_create(&arp_tbl, &nexthop, dev, false);
```

Continuing on, we can see that the next hop is computed by querying the routing layer followed by a lookup against the neighbour cache. If the neighbour is not found, one is created by calling `__neigh_create`. This could be the case, for example, the first time data is sent to another host. Note that this function is called with `arp_tbl` (defined in [./net/ipv4/arp.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/arp.c#L160-L187)) to create the neighbour entry in the ARP table. Other systems (like IPv6 or [DECnet](https://en.wikipedia.org/wiki/DECnet)) maintain their own ARP tables and would pass a different structure into `__neigh_create`. This post does not aim to cover the neighbour cache in full detail, but it is worth nothing that if the neighbour has to be created it is possible that this creation can cause the cache to grow. This post will cover some more details about the neighbour cache in the sections below. At any rate, the neighbour cache exports its own set of statistics so that this growth can be measured. See the monitoring sections below for more information.

```c
        if (!IS_ERR(neigh)) {
                int res = dst_neigh_output(dst, neigh, skb);

                rcu_read_unlock_bh();
                return res;
        }
        rcu_read_unlock_bh();

        net_dbg_ratelimited("%s: No header cache and no neighbour!\n",
                            __func__);
        kfree_skb(skb);
        return -EINVAL;
}
```

Finally, if no error is returned, `dst_neigh_output` is called to pass the skb along on its journey to be output. Otherwise, the skb is freed and EINVAL is returned. An error here will ripple back and cause `OutDiscards` to be incremented way back up in `ip_send_skb`. Let’s continue on in `dst_neigh_output` and continue approaching the Linux kernel’s netdevice subsystem.

### `dst_neigh_output`
The `dst_neigh_output` function does two important things for us. First, recall from earlier in this blog post we saw that if a user specified `MSG_CONFIRM` via an ancillary message to `sendmsg` the function, a flag is flipped to indicate that the destination cache entry for the remote host is still valid and should not be garbage collected. That check happens here and the `confirmed` field on the neighbour is set to the current jiffies count.

```c
static inline int dst_neigh_output(struct dst_entry *dst, struct neighbour *n,
                                   struct sk_buff *skb)
{
        const struct hh_cache *hh;

        if (dst->pending_confirm) {
                unsigned long now = jiffies;

                dst->pending_confirm = 0;
                /* avoid dirtying neighbour */
                if (n->confirmed != now)
                        n->confirmed = now;
        }
```

Second, the neighbour’s state is checked and the appropriate output function is called. Let’s take a look at the conditional and try to understand what’s going on:

```c
        hh = &n->hh;
        if ((n->nud_state & NUD_CONNECTED) && hh->hh_len)
                return neigh_hh_output(hh, skb);
        else
                return n->output(n, skb);
}
```

If a neighbour is considered `NUD_CONNECTED`, meaning it is one or more of:

*   `NUD_PERMANENT`: A static route.
*   `NUD_NOARP`: Does not require an ARP request (for example, the destination is a multicast or broadcast address, or a loopback device).
*   `NUD_REACHABLE`: The neighbour is “reachable.” A destination is marked as reachable whenever an ARP request for it [is successfully processed](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/arp.c#L905-L923).

_and_ the “hardware header” (`hh`) is cached (because we’ve sent data before and have previously generated it), call `neigh_hh_output`. Otherwise, call the `output` function. Both code paths end with `dev_queue_xmit` which pass the skb down to the Linux net device subsystem where it will be processed a bit more before hitting the device driver layer. Let’s follow both the `neigh_hh_output` and `n->output` code paths until we reach `dev_queue_xmit`.

### `neigh_hh_output`

If the destination is `NUD_CONNECTED` and the hardware header has been cached, `neigh_hh_output` will be called, which does a small bit of processing before handing the skb over to `dev_queue_xmit`. Let’s take a look, from [./include/net/neighbour.h](https://github.com/torvalds/linux/blob/v3.13/include/net/neighbour.h#L336-L356):

```c
static inline int neigh_hh_output(const struct hh_cache *hh, struct sk_buff *skb)
{
        unsigned int seq;
        int hh_len;

        do {
                seq = read_seqbegin(&hh->hh_lock);
                hh_len = hh->hh_len;
                if (likely(hh_len <= HH_DATA_MOD)) {
                        /* this is inlined by gcc */
                        memcpy(skb->data - HH_DATA_MOD, hh->hh_data, HH_DATA_MOD);
                 } else {
                         int hh_alen = HH_DATA_ALIGN(hh_len);

                         memcpy(skb->data - hh_alen, hh->hh_data, hh_alen);
                 }
         } while (read_seqretry(&hh->hh_lock, seq));

         skb_push(skb, hh_len);
         return dev_queue_xmit(skb);
}
```

This function is a bit tricky to understand, partially due to the locking primitive used to synchronize reading/writing on the cached hardware header. This code uses something called a [seqlock](https://en.wikipedia.org/wiki/Seqlock). You can imagine the `do { } while()` loop above as a simple retry mechanism which will attempt to perform the operations in the loop until it can be performed successfully.

The loop itself is attempted to determine if the hardware header’s length needs to be aligned prior to being copied. This is required because some hardware headers (like the [IEEE 802.11](https://github.com/torvalds/linux/blob/v3.13/include/linux/ieee80211.h#L210-L218) header) is larger than `HH_DATA_MOD` (16 bytes).

Once the data is copied to the skb and the skb’s internal pointers tracking the data are updated with `skb_push`, the skb is passed to `dev_queue_xmit` to enter the Linux net device subsystem.

### `n->output`

If the destination is not `NUD_CONNECTED` or the hardware header has not been cached the code proceeds down the `n->output` path. What is attached to the `output` function pointer on the neigbour structure? Well, it depends. To understand how this is setup, we’ll need to understand a bit more about how the neighbour cache works.

A `struct neighbour` contains several important fields. The `nud_state` field as we saw above, an `output` function, and an `ops` structure. Recall how earlier we saw that `__neigh_create` is called from `ip_finish_output2` if no existing entry was found in the cache. When `__neigh_creaet` is called a neighbour is allocated with its `output` [function initially set](https://github.com/torvalds/linux/blob/v3.13/net/core/neighbour.c#L294) to `neigh_blackhole`. As the `__neigh_create` code progresses, it will adjust the value of `output` to point to appropriate `output` functions based on the state of the neighbour.

For example, `neigh_connect` will be used to set the `output` pointer to `neigh->ops->connected_output` when the code determines the neighbour to be connected. Alternatively, `neigh_suspect` will be used to set the `output` pointer to `neigh->ops->output` when the code suspects that the neighbour may be down (for example if has been more than `/proc/sys/net/ipv4/neigh/default/delay_first_probe_time` seconds since a probe was sent).

In other words: `neigh->output` is set to another pointer, either `neigh->ops_connected_output` or `neigh->ops->output` depending on it’s state. Where does `neigh->ops` come from?

After the neighbour is allocated, `arp_constructor` (from [./net/ipv4/arp.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/arp.c#L220-L313)) is called to set some of the fields of the `struct neighbour`. In particular, this function checks the device associated with this neighbour and if the device exposes a `header_ops` structure that contains a `cache` function ([ethernet devices do](https://github.com/torvalds/linux/blob/v3.13/net/ethernet/eth.c#L342-L348)) `neigh->ops` is set to the following structure defined in [./net/ipv4/arp.c](https://github.com/torvalds/linux/blob/v3.13/net/ipv4/arp.c#L138-L144):

```c
static const struct neigh_ops arp_hh_ops = {
        .family =               AF_INET,
        .solicit =              arp_solicit,
        .error_report =         arp_error_report,
        .output =               neigh_resolve_output,
        .connected_output =     neigh_resolve_output,
};
```

So, regardless of whether or not the neighbour is considered “connected” or “suspect” by the neighbour cache code, the `neigh_resolve_output` function will be attached to `neigh->output` and will be called when `n->output` is called above.

#### `neigh_resolve_output`

This function’s purpose is to attempt to resolve a neighbour that is not connected or one which is connected, but has no cached hardware header. Let’s take a look at how this function works:

```c
/* Slow and careful. */

int neigh_resolve_output(struct neighbour *neigh, struct sk_buff *skb)
{
        struct dst_entry *dst = skb_dst(skb);
        int rc = 0;

        if (!dst)
                goto discard;

        if (!neigh_event_send(neigh, skb)) {
                int err;
                struct net_device *dev = neigh->dev;
                unsigned int seq;
```

The code starts by doing some basic checks and proceeds to calling `neigh_event_send`. The `neigh_event_send` function is short wrapper around `__neigh_event_send` which will do the heavy lifting to resolve the neighbour. You can read the source for `__neigh_event_send` in [./net/core/neighbour.c](https://github.com/torvalds/linux/blob/v3.13/net/core/neighbour.c#L964-L1028), but the high-level takeaway from the code is that there are three cases users will most interested in:

1.  Neighbours in state `NUD_NONE` (the default state when allocated) will cause an immediate ARP request to be sent assuming the values set in `/proc/sys/net/ipv4/neigh/default/app_solicit` and `/proc/sys/net/ipv4/neigh/default/mcast_solicit` allow probes to be sent (if not, the state is marked as `NUD_FAILED`). The neighbour state will be updated and set to `NUD_INCOMPLETE`.
2.  Neighbours in state `NUD_STALE` will be updated to `NUD_DELAYED` and a timer will be set to probe them later (later is the time now + `/proc/sys/net/ipv4/neigh/default/delay_first_probe_time` seconds).
3.  Any neighbours in `NUD_INCOMPLETE` (including things from case 1 above) will be checked to ensure that the number of queued packets for an unresolved neighbour is less than or equal to `/proc/sys/net/ipv4/neigh/default/unres_qlen`. If there are more, packets are dequeued and dropped until the length is below or equal to the value in proc. A statistics counter in the neighbour cache stats is bumped for all occurrences of this.

If an immediate ARP probe is needed it will be sent. `__neigh_event_send` will return either `0` indicating that the neighbour is considered “connected” or “delayed” or `1` otherwise. The return value of `0` allows `neigh_resolve_output` to continue:

```c
                if (dev->header_ops->cache && !neigh->hh.hh_len)
                        neigh_hh_init(neigh, dst);
```

If the device’s protocol implementation (ethernet in our case) associated with the neighbour supports caching the hardware header and it is currently not cached, the call to `neigh_hh_init` will cache it.

```c
                do {
                        __skb_pull(skb, skb_network_offset(skb));
                        seq = read_seqbegin(&neigh->ha_lock);
                        err = dev_hard_header(skb, dev, ntohs(skb->protocol),
                                              neigh->ha, NULL, skb->len);
                } while (read_seqretry(&neigh->ha_lock, seq));
```

Next, a [seqlock](https://en.wikipedia.org/wiki/Seqlock) is used to synchronize access to the neighbour structure’s hardware address which will be read by `dev_hard_header` when attempting to create the ethernet header for the skb. Once the seqlock has allowed execution to continue, error checking takes place:

```c
                if (err >= 0)
                        rc = dev_queue_xmit(skb);
                else
                        goto out_kfree_skb;
        }
```

If the ethernet header was written without returning an error, the skb is handed down to `dev_queue_xmit` to pass through the Linux network device subsystem for transmit. If there was an error, a `goto` will drop the skb, set the return code and return the error:

```c
out:
        return rc;
discard:
        neigh_dbg(1, "%s: dst=%p neigh=%p\n", __func__, dst, neigh);
out_kfree_skb:
        rc = -EINVAL;
        kfree_skb(skb);
        goto out;
}
EXPORT_SYMBOL(neigh_resolve_output);
```

Before we proceed into the Linux network device subsystem, let’s take a look at some files for monitoring and turning the IP protocol layer.

### Monitoring: IP protocol layer

#### `/proc/net/snmp`

Monitor detailed IP protocol statistics by reading `/proc/net/snmp`.

```sh
$ cat /proc/net/snmp
Ip: Forwarding DefaultTTL InReceives InHdrErrors InAddrErrors ForwDatagrams InUnknownProtos InDiscards InDelivers OutRequests OutDiscards OutNoRoutes ReasmTimeout ReasmReqds ReasmOKs ReasmFails FragOKs FragFails FragCreates
Ip: 1 64 25922988125 0 0 15771700 0 0 25898327616 22789396404 12987882 51 1 10129840 2196520 1 0 0 0
...
```

This file contains statistics for several protocol layers. The IP protocol layer appears first. The first line contains space separate names for each of the corresponding values in the next line.

In the IP protocol layer, you will find statistics counters being bumped. Those counters are referenced by a C enum. All of the valid enum values and the field names they correspond to in `/proc/net/snmp` can be found in [include/uapi/linux/snmp.h](https://github.com/torvalds/linux/blob/v3.13/include/uapi/linux/snmp.h#L10-L59):

```c
enum
{
  IPSTATS_MIB_NUM = 0,
/* frequently written fields in fast path, kept in same cache line */
  IPSTATS_MIB_INPKTS,     /* InReceives */
  IPSTATS_MIB_INOCTETS,     /* InOctets */
  IPSTATS_MIB_INDELIVERS,     /* InDelivers */
  IPSTATS_MIB_OUTFORWDATAGRAMS,   /* OutForwDatagrams */
  IPSTATS_MIB_OUTPKTS,      /* OutRequests */
  IPSTATS_MIB_OUTOCTETS,      /* OutOctets */

  /* ... */
```

Some interesting statistics:

*   `OutRequests`: Incremented each time an IP packet is attempted to be sent. It appears that this is incremented for every send, successful or not.
*   `OutDiscards`: Incremented each time an IP packet is discarded. This can happen if appending data to the skb (for corked sockets) fails, or if the layers below IP return an error.
*   `OutNoRoute`: Incremented in several places, for example in the UDP protocol layer (`udp_sendmsg`) if no route can be generated for a given destination. Also incremented when an application calls “connect” on a UDP socket but no route can be found.
*   `FragOKs`: Incremented once per packet that is fragmented. For example, a packet split into 3 fragments will cause this counter to be incremented once.
*   `FragCreates`: Incremented once per fragment that is created. For example, a packet split into 3 fragments will cause this counter to be incremented thrice.
*   `FragFails`: Incremented if fragmentation was attempted, but is not permitted (because the “Don’t Fragment” bit is set). Also incremented if outputting the fragment fails.

Other statistics are documented in [the receive side blog post](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#monitoring-ip-protocol-layer-statistics).

#### `/proc/net/netstat`

Monitor extended IP protocol statistics by reading `/proc/net/netstat`.

```sh
$ cat /proc/net/netstat | grep IpExt
IpExt: InNoRoutes InTruncatedPkts InMcastPkts OutMcastPkts InBcastPkts OutBcastPkts InOctets OutOctets InMcastOctets OutMcastOctets InBcastOctets OutBcastOctets InCsumErrors InNoECTPkts InECT0Pktsu InCEPkts
IpExt: 0 0 0 0 277959 0 14568040307695 32991309088496 0 0 58649349 0 0 0 0 0
```

The format is similar to `/proc/net/snmp`, except the lines are prefixed with `IpExt`.

Some interesting statistics:

*   `OutMcastPkts`: Incremented each time a packet destined for a multicast address is sent.
*   `OutBcastPkts`: Incremented each time a packet destined for a broadcast address is sent.
*   `OutOctects`: The number of packet bytes output.
*   `OutMcastOctets`: The number of multicast packet bytes output.
*   `OutBcastOctets`: The number of broadcast packet bytes output.

Other statistics are documented in [the receive side blog post](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#monitoring-ip-protocol-layer-statistics).

Note that each of these is incremented in really specific locations in the IP layer. Code gets moved around from time to time and double counting errors or other accounting bugs can sneak in. If these statistics are important to you, you are strongly encouraged to read the IP protocol layer source code for the metrics that are important to you so you understand when they are (and are not) being incremented.

## Linux netdevice subsystem

Before we pick up on the packet transmit path with `dev_queue_xmit`, let’s take a moment to talk about some important concepts which will appear in the coming sections.

### Linux traffic control

Linux supports a feature called [traffic control](http://tldp.org/HOWTO/Traffic-Control-HOWTO/intro.html). This feature allows system administrators to control how packets are transmit from a machine. This blog post will not dive into the details of every aspect of Linux traffic control. [This document](http://tldp.org/HOWTO/Traffic-Control-HOWTO/) provides a great in-depth examination of the system, its control, and its features. There a few concepts that are worth mentioning to make the code seen next easier to understand.

The traffic control system contains several different sets of queuing systems that provide different features for controlling traffic flow. Individual queuing systems are commonly called `qdisc` and also known as queuing disciplines. You can think of qdiscs as schedulers; qdiscs decide when and how packets are transmit.

On Linux every interface has a default qdisc associated with it. For network hardware that supports only a single transmit queue, the default qdisc `pfifo_fast` is used. Network hardware that supports multiple transmit queues uses the default qdisc of `mq`. You can check your system by running `tc qdisc`.

It is also important to note that some devices support traffic control in hardware which can allow an administrator to offload traffic control to the network hardware and conserve CPU resources on the system.

Now that those ideas have been introduced, let’s proceed down `dev_queue_xmit` from [./net/core/dev.c](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L2890-L2894).

### `dev_queue_xmit` and `__dev_queue_xmit`

`dev_queue_xmit` is a simple wrapper around `__dev_queue_xmit`:

```c
int dev_queue_xmit(struct sk_buff *skb)
{
        return __dev_queue_xmit(skb, NULL);
}
EXPORT_SYMBOL(dev_queue_xmit);
```

Following that, `__dev_queue_xmit` is where the heavy lifting gets done. Let’s take a look and step through this code piece by piece. [Follow along](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L2808-L2825):

```c
static int __dev_queue_xmit(struct sk_buff *skb, void *accel_priv)
{
        struct net_device *dev = skb->dev;
        struct netdev_queue *txq;
        struct Qdisc *q;
        int rc = -ENOMEM;

        skb_reset_mac_header(skb);

        /* Disable soft irqs for various locks below. Also
         * stops preemption for RCU.
         */
        rcu_read_lock_bh();

        skb_update_prio(skb);
```

The code above starts out by:

1.  Declaring variables.
2.  Preparing the skb to be processed by calling `skb_reset_mac_header`. This resets the skb’s internal pointers so that the ethernet header can be accessed.
3.  `rcu_read_lock_bh` is called to prepare for reading RCU protected data structures in the code below. [Read more about safely using RCU](https://www.kernel.org/doc/Documentation/RCU/checklist.txt).
4.  `skb_update_prio` is called to set the skb’s priority, if [the network priority cgroup is being used](https://github.com/torvalds/linux/blob/v3.13/Documentation/cgroups/net_prio.txt).

Now, we’ll get to the more complicated parts of transmitting data ;)

```c
        txq = netdev_pick_tx(dev, skb, accel_priv);
```

Here the code attempts to determine which transmit queue to use. As you’ll see later in this post, some network devices expose multiple transmit queues for transmitting data. Let’s see how this works in detail.

#### `netdev_pick_tx`

The `netdev_pick_tx` code lives in [./net/core/flow_dissector.c](https://github.com/torvalds/linux/blob/v3.13/net/core/flow_dissector.c#L397-L417). Let’s take a look:

```c
struct netdev_queue *netdev_pick_tx(struct net_device *dev,
                                    struct sk_buff *skb,
                                    void *accel_priv)
{
        int queue_index = 0;

        if (dev->real_num_tx_queues != 1) {
                const struct net_device_ops *ops = dev->netdev_ops;
                if (ops->ndo_select_queue)
                        queue_index = ops->ndo_select_queue(dev, skb,
                                                            accel_priv);
                else
                        queue_index = __netdev_pick_tx(dev, skb);

                if (!accel_priv)
                        queue_index = dev_cap_txqueue(dev, queue_index);
        }

        skb_set_queue_mapping(skb, queue_index);
        return netdev_get_tx_queue(dev, queue_index);
}
```

As you can see above, if the network device supports only a single TX queue, the more complex code is skipped and that single TX queue is returned. Most devices used on higher end servers will have multiple TX queues. There are two cases for devices with multiple TX queues:

1.  The driver implements `ndo_select_queue`, which can be used to choose a TX queue more intelligently in a hardware or feature specific way, or
2.  The driver does not implement `ndo_select_queue, so the kernel should pick the device itself.

As of the 3.13 kernel, not many drivers implement `ndo_select_queue`. The bnx2x and ixgbe drivers implement this function, but it is only used for [fibre channel over ethernet (FCoE)](https://en.wikipedia.org/wiki/Fibre_Channel_over_Ethernet). In light of this, let’s assume that the network device does not implement `ndo_select_queue` and/or that FCoE is not being used. In that case, the kernel will choose the tx queue with `__netdev_pick_tx`.

Once `__netdev_pick_tx` determines what the queue is index, `skb_set_queue_mapping` will cache that value (it will be used later in the traffic control code) and `netdev_get_tx_queue` will look up and return a pointer to that queue. Let’s take a look at how `__netdev_pick_tx` works before going back up to `__dev_queue_xmit`.

#### `__netdev_pick_tx`

Let’s take a look at how the kernel chooses the TX queue to use for transmitting data. From [./net/core/flow_dissector.c](https://github.com/torvalds/linux/blob/v3.13/net/core/flow_dissector.c#L375-L395):

```c
u16 __netdev_pick_tx(struct net_device *dev, struct sk_buff *skb)
{
        struct sock *sk = skb->sk;
        int queue_index = sk_tx_queue_get(sk);

        if (queue_index < 0 || skb->ooo_okay ||
            queue_index >= dev->real_num_tx_queues) {
                int new_index = get_xps_queue(dev, skb);
                if (new_index < 0)
                        new_index = skb_tx_hash(dev, skb);

                if (queue_index != new_index && sk &&
                    rcu_access_pointer(sk->sk_dst_cache))
                        sk_tx_queue_set(sk, new_index);

                queue_index = new_index;
        }

        return queue_index;
}
```

The code begins first by checking if the transmit queue has already been cached on the socket by calling `sk_tx_queue_get`, If it hasn’t been cached, `-1` is returned.

The next if-statement checks if any of the following are true:

*   The queue_index is < 0\. This will happen if the queue hasn’t been set yet.
*   If the `ooo_okay` flag is set. If this flag is set, this means that out of order packets are allowed now. The protocol layers must set this flag appropriately. The TCP protocol layer sets this flag when all outstanding packets for a flow have been acknowledged. When this happens, the kernel can choose a different TX queue for this packet. The UDP protocol layer does not set this flag – so UDP packets will never have `ooo_okay` set to a non-zero value.
*   If the queue index is larger than the number of queues. This can happen if the user has recently changed the queue count on the device via `ethtool`. More on this later.

In any of those cases, the code descends into the slow path to get the transmit queue. This begins with `get_xps_queue` which attempts to use a user-configured map linking transmit queues to CPUs. This is called “Transmit Packet Steering.” We’ll look more closely at what Transmit Packet Steering (XPS) is and how it works shortly.

If `get_xps_queue` returns `-1` because this kernel does not support XPS, or XPS was not configured by the system administrator, or the mapping configured refers to an invalid queue the code will continue on to call `skb_tx_hash`.

Once the queue is selected by either XPS or by the kernel automatically with `skb_tx_hash`, the queue is cached on the socket object with `sk_tx_queue_set` and returned. Let’s see how XPS and `skb_tx_hash` work before continuing through `dev_queue_xmit`.

##### Transmit Packet Steering (XPS)

Transmit Packet Steering (XPS) is a feature that allows the system administrator to determine which CPUs can process transmit operations for each available transmit queue supported by the device. The aim of this feature is mainly to avoid lock contention when processing transmit requests. Other benefits like reducing cache evictions and avoiding remote memory access on [NUMA machines](https://en.wikipedia.org/wiki/Non-uniform_memory_access) are also expected when using XPS.

You can read more about how XPS works by [checking the kernel documentation for XPS](https://github.com/torvalds/linux/blob/v3.13/Documentation/networking/scaling.txt#L364-L422). We’ll examine how to tune XPS for your system below, but for now, all you need to know is that to configure XPS the system administrator can define a bitmap mapping transmit queues to CPUs.

The function call in the code above to `get_xps_queue` will consult this user-specified map in order to determine which transmit queue should be used. If `get_xps_queue` returns `-1`, `skb_tx_hash` will be used instead.

##### `skb_tx_hash`

If XPS is not included in the kernel, or is not configured, or suggests a queue that is not available (because perhaps the user adjusted the queue count) `skb_tx_hash` takes over to determine which queue the data should be sent on. Understanding precisely how `skb_tx_hash` works is important depending on your transmit workload. Note that this code has been adjusted over time, so if you are using a different kernel version than this document, you should consult your kernel source directly.

Let’s take a look at how it works, from [./include/linux/netdevice.h](https://github.com/torvalds/linux/blob/v3.13/include/linux/netdevice.h#L2331-L2340):

```c
/*
 * Returns a Tx hash for the given packet when dev->real_num_tx_queues is used
 * as a distribution range limit for the returned value.
 */
static inline u16 skb_tx_hash(const struct net_device *dev,
                              const struct sk_buff *skb)
{
        return __skb_tx_hash(dev, skb, dev->real_num_tx_queues);
}
```

The code simply calls down to `__skb_tx_hash`, from [./net/core/flow_dissector.c](https://github.com/torvalds/linux/blob/v3.13/net/core/flow_dissector.c#L239-L271). There’s some interesting code in this function, so let’s take a look:

```c
/*
 * Returns a Tx hash based on the given packet descriptor a Tx queues' number
 * to be used as a distribution range.
 */
u16 __skb_tx_hash(const struct net_device *dev, const struct sk_buff *skb,
                  unsigned int num_tx_queues)
{
        u32 hash;
        u16 qoffset = 0;
        u16 qcount = num_tx_queues;

        if (skb_rx_queue_recorded(skb)) {
                hash = skb_get_rx_queue(skb);
                while (unlikely(hash >= num_tx_queues))
                        hash -= num_tx_queues;
                return hash;
        }
```

The first if stanza in this function is an interesting short circuit. The function name `skb_rx_queue_recorded` is a bit misleading. An skb has a `queue_mapping` field that is used both for rx and tx. At any rate, this if statement can be true if your system is receiving packets and forwarding them elsewhere. If that isn’t the case, the code continues.

```c
        if (dev->num_tc) {
                u8 tc = netdev_get_prio_tc_map(dev, skb->priority);
                qoffset = dev->tc_to_txq[tc].offset;
                qcount = dev->tc_to_txq[tc].count;
        }
```

To understand this piece of code, it is important to mention that a program can set the priority of data sent on a socket. This can be done by using `setsockopt` with the `SOL_SOCKET` and `SO_PRIORITY` level and optname, respectively. See the [socket(7) man page](http://man7.org/linux/man-pages/man7/socket.7.html) for more information about `SO_PRIORITY`.

Note that if you have used the `setsockopt` option `IP_TOS` to set the TOS flags on the IP packets sent on a particular socket (or on a per-packet basis if passed as an ancillary message to `sendmsg`) in your application, the kernel will translate the TOS options set by you to a priority which end up in `skb->priority`.

As was mentioned earlier, some network devices support hardware based traffic control systems. If `num_tc` is non-zero, that means this device supports hardware based traffic control.

If that number is non-zero it means that this device supports hardware based traffic control. The priority map which maps packet priority to hardware based traffic control will be consulted. The appropriate traffic class for the data’s priority will be selected based on this map.

Next, the range of appropriate transmit queues for the traffic class will be generated. They will be used to determine the transmit queue.

If `num_tc` was zero (because the network device does not support hardware based traffic control), the `qcount` and `qoffset` variables are set to the number of transmit queues and `0`, respectively.

Using `qcount` and `qoffset`, the index of the transmit queue will be calculated:

```c
        if (skb->sk && skb->sk->sk_hash)
                hash = skb->sk->sk_hash;
        else
                hash = (__force u16) skb->protocol;
        hash = __flow_hash_1word(hash);

        return (u16) (((u64) hash * qcount) >> 32) + qoffset;
}
EXPORT_SYMBOL(__skb_tx_hash);
```

Finally, the appropriate queue index is returned back up to `__netdev_pick_tx`.

### Resuming `__dev_queue_xmit`

At this point the appropriate transmit queue has been selected. `__dev_queue_xmit` can continue:

```c
        q = rcu_dereference_bh(txq->qdisc);

#ifdef CONFIG_NET_CLS_ACT
        skb->tc_verd = SET_TC_AT(skb->tc_verd, AT_EGRESS);
#endif
        trace_net_dev_queue(skb);
        if (q->enqueue) {
                rc = __dev_xmit_skb(skb, q, dev, txq);
                goto out;
        }
```

It starts by obtaining a reference to the queuing discipline associated with this queue. Recall that earlier we saw that the default for single transmit queue devices is the `pfifo_fast` qdisc, whereas for multiqueue devices it is the `mq` qdisc.

Next, the code assigns a traffic classification “verdict” to the outgoing data, if the packet classification API has been enabled in your kernel. Next, the queue discipline is checked to see if there is a way to queue data. Some queuing disciplines like the `noqueue` qdisc do not have a queue. If there is a queue, the code calls down to `__dev_xmit_skb` to continue processing the data for transmit. Afterward, execution jumps to the end of this function. We’ll take a look at `__dev_xmit_skb` shortly. For now, let’s see what happens if there is no queue, starting with a very helpful comment:

```c
        /* The device has no queue. Common case for software devices:
           loopback, all the sorts of tunnels...

           Really, it is unlikely that netif_tx_lock protection is necessary
           here.  (f.e. loopback and IP tunnels are clean ignoring statistics
           counters.)
           However, it is possible, that they rely on protection
           made by us here.

           Check this and shot the lock. It is not prone from deadlocks.
           Either shot noqueue qdisc, it is even simpler 8)
         */
        if (dev->flags & IFF_UP) {
                int cpu = smp_processor_id(); /* ok because BHs are off */
```

As the comment illustrates, the only devices that could have a qdisc with no queues are the loopback device and tunnel devices. If the device is currently up, then the current CPU is saved. It used for the next check which is a bit tricky, let’s take a look:

```c
                if (txq->xmit_lock_owner != cpu) {

                        if (__this_cpu_read(xmit_recursion) > RECURSION_LIMIT)
                                goto recursion_alert;
```

There’s two cases: the transmit lock on this device queue is owned by this CPU or not. If so, a counter variable `xmit_recursion`, which is allocated per-CPU, is checked here to determine if the count is over the `RECURSION_LIMIT`. It is possible that one program could attempt to send data and get preempted right around this place in the code. Another program could be selected by the scheduler to run. If that second program attempts to send data as well and lands here. So, the `xmit_recursion` counter is used to prevent more than `RECURSION_LIMIT` programs from racing here to transmit data. Let’s keep going:

```c
                        HARD_TX_LOCK(dev, txq, cpu);

                        if (!netif_xmit_stopped(txq)) {
                                __this_cpu_inc(xmit_recursion);
                                rc = dev_hard_start_xmit(skb, dev, txq);
                                __this_cpu_dec(xmit_recursion);
                                if (dev_xmit_complete(rc)) {
                                        HARD_TX_UNLOCK(dev, txq);
                                        goto out;
                                }
                        }
                        HARD_TX_UNLOCK(dev, txq);
                        net_crit_ratelimited("Virtual device %s asks to queue packet!\n",
                                             dev->name);
                } else {
                        /* Recursion is detected! It is possible,
                         * unfortunately
                         */
recursion_alert:
                        net_crit_ratelimited("Dead loop on virtual device %s, fix it urgently!\n",
                                             dev->name);
                }
        }
```

The remainder of the code starts by trying to take the transmit lock. The device’s transmit queue to be used is checked to see if transmit is stopped. If not, the `xmit_recursion` variable is incremented and the data is passed down closer to the device to be transmit. We’ll see `dev_hard_start_xmit` in more detail later. Once this completes, the locks are released and a warning is printed.

Alternatively, if the current CPU is transmit lock owner, or if the `RECURSION_LIMIT` is hit, no transmit is done, but a warning is printed. The remaining code in the function sets the error code and returns.

Since we are interested in real ethernet devices, let’s continue down the code path that would have been taken for those earlier via `__dev_xmit_skb`.

### `__dev_xmit_skb`

And now we descend into `__dev_xmit_skb` from [./net/core/dev.c](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L2684-L2745) armed with the queuing discipline, network device, and transmit queue reference:

```c
static inline int __dev_xmit_skb(struct sk_buff *skb, struct Qdisc *q,
                                 struct net_device *dev,
                                 struct netdev_queue *txq)
{
        spinlock_t *root_lock = qdisc_lock(q);
        bool contended;
        int rc;

        qdisc_pkt_len_init(skb);
        qdisc_calculate_pkt_len(skb, q);
        /*
         * Heuristic to force contended enqueues to serialize on a
         * separate lock before trying to get qdisc main lock.
         * This permits __QDISC_STATE_RUNNING owner to get the lock more often
         * and dequeue packets faster.
         */
        contended = qdisc_is_running(q);
        if (unlikely(contended))
                spin_lock(&q->busylock);
```

This code begins by using `qdisc_pkt_len_init` and `qdisc_calculate_pkt_len` to compute an accurate length for the data that will be used by the qdisc later. This is necessary for skbs that will pass through hardware based send offloading (such as UDP Fragmentation Offloading, as we saw earlier) as the additional headers that will be added when fragmentation occurs need to be taken into account.

Next, a lock is used to help reduce contention on the qdisc’s main lock (a second lock we’ll see later). If qdisc is currently running, then other programs attempting to transmit will contend on the qdisc’s `busylock`. This allows the running qdisc to process packets and contend with a smaller number of programs for the second, main lock. This trick increases throughput as the number of contenders is reduced. You can read the original commit message describing this [here](https://github.com/torvalds/linux/commit/79640a4ca6955e3ebdb7038508fa7a0cd7fa5527). Next the main lock is taken:

```c
        spin_lock(root_lock);
```

Now, we approach an if statement that handles 3 possible cases:

1.  The qdisc is deactivated.
2.  The qdisc allows packets to bypass the queuing system, there are no other packets to send, and the qdisc is not currently running. A qdisc allows packet bypass for “work-conserving” qdisc - in other words, a qdisc that does not delay packet transmit for traffic shaping purposes.
3.  All other cases.

Let’s take a look at what happens in each of these cases, in order starting with a deactivated qdisc:

```c
        if (unlikely(test_bit(__QDISC_STATE_DEACTIVATED, &q->state))) {
                kfree_skb(skb);
                rc = NET_XMIT_DROP;
```

This is straightforward. If the qdisc is deactivated, free the data and set the return code to `NET_XMIT_DROP`. Next, a qdisc allowing packet bypass, with no other outstanding packets, that is not currently running:

```c
        } else if ((q->flags & TCQ_F_CAN_BYPASS) && !qdisc_qlen(q) &&
                   qdisc_run_begin(q)) {
                /*
                 * This is a work-conserving queue; there are no old skbs
                 * waiting to be sent out; and the qdisc is not running -
                 * xmit the skb directly.
                 */
                if (!(dev->priv_flags & IFF_XMIT_DST_RELEASE))
                        skb_dst_force(skb);

                qdisc_bstats_update(q, skb);

                if (sch_direct_xmit(skb, q, dev, txq, root_lock)) {
                        if (unlikely(contended)) {
                                spin_unlock(&q->busylock);
                                contended = false;
                        }
                        __qdisc_run(q);
                } else
                        qdisc_run_end(q);

                rc = NET_XMIT_SUCCESS;
```

This if statement is a bit tricky. The entire statement evaluates as `true` if all of the following are true:

1.  `q->flags & TCQ_F_CAN_BYPASS`: The qdisc allows packets to bypass the queuing system. This will be true for “work-conserving” qdiscs; i.e. qdiscs that do not delay packet transmit for traffic shaping purposes are considered “work-conserving” and allow packet bypass. The `pfifo_fast` qdisc allows packets to bypass the queuing system.
2.  `!qdisc_qlen(q)`: The qdisc’s queue has no data in it that is waiting to be transmit.
3.  `qdisc_run_begin(p)`: This function call will either set the qdisc’s state as “running” and return true or return false if the qdisc was already running.

If all of the above evaluate to true, then:

*   The `IFF_XMIT_DST_RELEASE` flag is checked. If enabled, this flag indicates that the kernel is allowed to free the skb’s destination cache structure. The code in this function checks if the flag is disabled and forces a reference count on that structure.
*   `qdisc_bstats_update` is used to increment the number of bytes and packet sent by the qdisc.
*   `sch_direct_xmit` is used to attempt to transmit the packet. We’ll dive more into `sch_direct_xmit` shortly as it is used in the slower code path, too.

The return value of `sch_direct_xmit` is checked for two cases:

1.  The queue is not empty (`>0` returned). In this case, lock preventing contention from other programs is released and `__qdisc_run` is called to restart the qdisc processing.
2.  The queue was empty (`0` is returned). In this case `qdisc_run_end` is used to turn off qdisc processing.

In either case, the return value `NET_XMIT_SUCCESS` is set as the return code. That wasn’t too bad. Let’s check the last case, which is catch all:

```c
        } else {
                skb_dst_force(skb);
                rc = q->enqueue(skb, q) & NET_XMIT_MASK;
                if (qdisc_run_begin(q)) {
                        if (unlikely(contended)) {
                                spin_unlock(&q->busylock);
                                contended = false;
                        }
                        __qdisc_run(q);
                }
        }
```

In all other cases:

1.  Call `skb_dst_force` to force a reference count bump on the skb’s destination cache reference.
2.  Queue the data to the qdisc by calling the `enqueue` function of the queue disc. Store the return code.
3.  Call `qdisc_run_begin(p)` to mark the qdisc as running. If it was not already running, the `busylock` is released and `__qdisc_run(p)` is called to start qdisc processing.

The function then finishes up by releasing some locks and returning the return code:

```c
        spin_unlock(root_lock);
        if (unlikely(contended))
                spin_unlock(&q->busylock);
        return rc;
```
### Tuning: Transmit Packet Steering (XPS)

For XPS to work, it must be enabled in the kernel configuration (it is on Ubuntu for kernel 3.13.0), and a bitmask describing which CPUs should process packets for a given interface and TX queue.

These bitmasks are similar to the [RPS](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#receive-packet-steering-rps) bitmasks and you can find some [documentation](https://github.com/torvalds/linux/blob/v3.13/Documentation/networking/scaling.txt#L147-L150) about these bitmasks in the kernel documentation.

In short, the bitmasks to modify are found in:

`/sys/class/net/DEVICE_NAME/queues/QUEUE/xps_cpus`

So, for eth0 and transmit queue 0, you would modify the file: `/sys/class/net/eth0/queues/tx-0/xps_cpus` with a hexadecimal number indicating which CPUs should process transmit completions from `eth0`’s transmit queue 0\. As the [documentation points out](https://github.com/torvalds/linux/blob/v3.13/Documentation/networking/scaling.txt#L412-L422), XPS may be unnecessary in certain configurations.

## Queuing disciplines!

To follow the path of network data, we’ll need to move into the qdisc code a bit. This post does not intend to cover the specific details of each of the different transmit queue options. If you are interested in that, [check this excellent guide](http://lartc.org/howto/index.html).

For the purpose of this blog post, we’ll continue the code path by examining how the generic packet scheduler code works. In particular, we’ll explore how `qdisc_run_begin`, `qdisc_run_end`, `__qdisc_run`, and `sch_direct_xmit` work to move network data closer to the driver for transmit.

Let’s start by examining how `qdisc_run_begin` works and proceed from there.

### `qdisc_run_begin` and `qdisc_run_end`

The `qdisc_run_begin` function can be found in [./include/net/sch_generic.h](https://github.com/torvalds/linux/blob/v3.13/include/net/sch_generic.h#L101-L107):

```c
static inline bool qdisc_run_begin(struct Qdisc *qdisc)
{
        if (qdisc_is_running(qdisc))
                return false;
        qdisc->__state |= __QDISC___STATE_RUNNING;
        return true;
}
```

This function is simple: the qdisc `__state` flag is checked. If it’s already running, `false` is returned. Otherwise, `__state` is updated to enable the `__QDISC___STATE_RUNNING` bit.

Similarly, `qdisc_run_end` [is anti-climactic](https://github.com/torvalds/linux/blob/v3.13/include/net/sch_generic.h#L109-L113):

```c
static inline void qdisc_run_end(struct Qdisc *qdisc)
{
        qdisc->__state &= ~__QDISC___STATE_RUNNING;
}
```

It simply disables the `__QDISC___STATE_RUNNING` bit from the qdisc’s `__state` field. It is important to note that both of these functions simply flip bits; neither actually start or stop processing themselves. The function `__qdisc_run`, on the other hand, will actually start processing.

### `__qdisc_run`

The code for `__qdisc_run` is deceptively brief:

```c
void __qdisc_run(struct Qdisc *q)
{
        int quota = weight_p;

        while (qdisc_restart(q)) {
                /*
                 * Ordered by possible occurrence: Postpone processing if
                 * 1\. we've exceeded packet quota
                 * 2\. another process needs the CPU;
                 */
                if (--quota <= 0 || need_resched()) {
                        __netif_schedule(q);
                        break;
                }
        }

        qdisc_run_end(q);
}
```

This function begins by obtaining the `weight_p` value. This is set typically via a sysctl and is also used in the receive path. We’ll see later how to adjust this value. This loop does two things:

1.  It calls `qdisc_restart` in a busy loop until it returns false (or the break below is triggered).
2.  Determines if either the quota drops below zero or `need_resched()` returns true. If either is `true`, `__netif_schedule` is called and the loop is broken out of.

Remember: up to now the kernel is still executing on behalf of the original call to `sendmsg` by the user program; the user program is currently accumulating system time. If the user program has exhausted its time quota in the kernel, `need_resched` will return true. If there’s still available quota and the user program hasn’t used is time slice up yet, `qdisc_restart` will be called over again.

Let’s see how `qdisc_restart(q)` works and then we’ll dive into `__netif_schedule(q)`.

### `qdisc_restart`

Let’s jump into [the code for `qdisc_restart`](https://github.com/torvalds/linux/blob/v3.13/net/sched/sch_generic.c#L156-L192):

```c
/*
 * NOTE: Called under qdisc_lock(q) with locally disabled BH.
 *
 * __QDISC_STATE_RUNNING guarantees only one CPU can process
 * this qdisc at a time. qdisc_lock(q) serializes queue accesses for
 * this queue.
 *
 *  netif_tx_lock serializes accesses to device driver.
 *
 *  qdisc_lock(q) and netif_tx_lock are mutually exclusive,
 *  if one is grabbed, another must be free.
 *
 * Note, that this procedure can be called by a watchdog timer
 *
 * Returns to the caller:
 *                                0  - queue is empty or throttled.
 *                                >0 - queue is not empty.
 *
 */
static inline int qdisc_restart(struct Qdisc *q)
{
        struct netdev_queue *txq;
        struct net_device *dev;
        spinlock_t *root_lock;
        struct sk_buff *skb;

        /* Dequeue packet */
        skb = dequeue_skb(q);
        if (unlikely(!skb))
                return 0;
        WARN_ON_ONCE(skb_dst_is_noref(skb));
        root_lock = qdisc_lock(q);
        dev = qdisc_dev(q);
        txq = netdev_get_tx_queue(dev, skb_get_queue_mapping(skb));

        return sch_direct_xmit(skb, q, dev, txq, root_lock);
}
```

The `qdisc_restart` function begins with a useful comment describing some of the locking constraints for calling this function. The first operation this function performs is to attempt to dequeue an skb from the qdisc.

The function `dequeue_skb` will attempt to obtain the next packet to transmit. If the queue is empty `qdisc_restart` will return false (causing the loop in `__qdisc_run` above to bail).

Assuming there is data to transmit, the code continues by obtaining a reference to the qdisc queue lock, the qdisc’s associated device, and the transmit queue.

All of these are passed through to `sch_direct_xmit`. Let’s take a look at `dequeue_skb` and then we’ll come back `sch_direct_xmit`.

#### `dequeue_skb`

Let’s take a look at `dequeue_skb` from [./net/sched/sch_generic.c](https://github.com/torvalds/linux/blob/v3.13/net/sched/sch_generic.c#L59-L78). This function handles two major cases:

1.  Dequeuing data that was requeued because it could not be sent before, or
2.  Dequeuing new data from the qdisc to be processed.

Let’s take a look at the first case:

```c
static inline struct sk_buff *dequeue_skb(struct Qdisc *q)
{
        struct sk_buff *skb = q->gso_skb;
        const struct netdev_queue *txq = q->dev_queue;

        if (unlikely(skb)) {
                /* check the reason of requeuing without tx lock first */
                txq = netdev_get_tx_queue(txq->dev, skb_get_queue_mapping(skb));
                if (!netif_xmit_frozen_or_stopped(txq)) {
                        q->gso_skb = NULL;
                        q->q.qlen--;
                } else
                        skb = NULL;
```

Note that the code begins by taking a reference to `gso_skb` field of the qdisc. This field holds a reference to data that was requeued. If no data was requeued, this field will be `NULL`. If that field is not `NULL`, the code continues by getting the transmit queue for the data and checking if the queue is stopped. If the queue is not stopped, the `gso_skb` field is cleared and the queue length counter is decreased. If the queue is stopped, the data remains attached to `gso_skb`, but `NULL` will be returned from this function.

Let’s check the next case, where there is no data that was requeued:

```c
        } else {
                if (!(q->flags & TCQ_F_ONETXQUEUE) || !netif_xmit_frozen_or_stopped(txq))
                        skb = q->dequeue(q);
        }

        return skb;
}
```

In the case where no data was requeued, another tricky compound if statement is evaluated. If:

1.  The qdisc does not have a single transmit queue, or
2.  The transmit queue is not stopped

Then, the qdisc’s `dequeue` function will be called to obtain new data. The internal implementation of `dequeue` will vary depending on the qdisc’s implementation and features.

The function finishes by returning the data that is up for processing.

#### `sch_direct_xmit`

Now we come to `sch_direct_xmit` (in [./net/sched/sch_generic.c](https://github.com/torvalds/linux/blob/v3.13/net/sched/sch_generic.c#L109-L154)) which is an important participant in moving data down toward the network device. Let’s walk through it, piece by piece:

```c
/*
 * Transmit one skb, and handle the return status as required. Holding the
 * __QDISC_STATE_RUNNING bit guarantees that only one CPU can execute this
 * function.
 *
 * Returns to the caller:
 *                                0  - queue is empty or throttled.
 *                                >0 - queue is not empty.
 */
int sch_direct_xmit(struct sk_buff *skb, struct Qdisc *q,
                    struct net_device *dev, struct netdev_queue *txq,
                    spinlock_t *root_lock)
{
        int ret = NETDEV_TX_BUSY;

        /* And release qdisc */
        spin_unlock(root_lock);

        HARD_TX_LOCK(dev, txq, smp_processor_id());
        if (!netif_xmit_frozen_or_stopped(txq))
                ret = dev_hard_start_xmit(skb, dev, txq);

        HARD_TX_UNLOCK(dev, txq);
```

The code begins by unlocking the qdisc lock and then locking the transmit lock. Note that `HARD_TX_LOCK` is a macro:

```c
#define HARD_TX_LOCK(dev, txq, cpu) {                   \
        if ((dev->features & NETIF_F_LLTX) == 0) {      \
                __netif_tx_lock(txq, cpu);              \
        }                                               \
}
```

This macro is checking if the device has the `NETIF_F_LLTX` flag set in its feature flags. This flag is deprecated and should not be used by new device drivers. Most drivers in this kernel version do not use this flag, so this check will evaluate to to true and the lock for the transmit queue for this data will be obtained.

Next, the transmit queue is checked to ensure that it is not stopped and then `dev_hard_start_xmit` is called. As we’ll see later, `dev_hard_start_xmit` handles transitioning the network data from the Linux kernel’s network device subsystem into the device driver itself for transmission. The return code from this function is stored and will be checked next to determine if the transmit succeeded.

Once this has run (or been skipped because the queue is stopped), the queue’s transmit lock is released. Let’s continue:

```c
        spin_lock(root_lock);

        if (dev_xmit_complete(ret)) {
                /* Driver sent out skb successfully or skb was consumed */
                ret = qdisc_qlen(q);
        } else if (ret == NETDEV_TX_LOCKED) {
                /* Driver try lock failed */
                ret = handle_dev_cpu_collision(skb, txq, q);
```

Next, the lock for this qdisc is taken again and then the return value of `dev_hard_start_xmit` is examined. The first case is checked by calling `dev_xmit_complete` which simply checks the return value to determine if the data was sent successfully. If so the qdisc queue length is set as the return value.

If `dev_xmit_complete` returns false, the return value will be checked to see if `dev_hard_start_xmit` returned `NETDEV_TX_LOCKED` up from the device driver. Devices with the deprecated `NETIF_F_LLTX` feature flag can return `NETDEV_TX_LOCKED` when the driver attempts to do its own locking of the transmit queue and fails. In this case, `handle_dev_cpu_collision` is called to deal with the lock contention. We’ll take a closer look at `handle_dev_cpu_collision` shortly, but for now, let’s continue down `sch_direct_xmit` and check out the catch-all case:

```c
        } else {
                /* Driver returned NETDEV_TX_BUSY - requeue skb */
                if (unlikely(ret != NETDEV_TX_BUSY))
                        net_warn_ratelimited("BUG %s code %d qlen %d\n",
                                             dev->name, ret, q->q.qlen);

                ret = dev_requeue_skb(skb, q);
        }
```

So if the driver did not transmit the data and it was not due to the transmit lock being held, it is probably due to `NETDEV_TX_BUSY` (if not a warning is printed). `NETDEV_TX_BUSY` can be returned by a driver to indicate that either the device or the driver were “busy” and the data can not be transmit right now. In this case, `dev_requeue_skb` is used to queue the data to be retried.

The function wraps up by (possibly) adjusting the return value:

```c
        if (ret && netif_xmit_frozen_or_stopped(txq))
                ret = 0;

        return ret;
```

Let’s a take a dive into `handle_dev_cpu_collision` and `dev_requeue_skb`.

#### `handle_dev_cpu_collision`

The code for `handle_dev_cpu_collision`, from [./net/sched/sch_generic.c](https://github.com/torvalds/linux/blob/v3.13/net/sched/sch_generic.c#L80-L107) handles two cases:

1.  The transmit lock is held by the current CPU.
2.  The transmit lock is held by some other CPU.

In the first case, this is handled as a configuration problem and thus a warning is printed. In the second case a statistic counter `cpu_collision` is incremented and the data is sent through `dev_requeue_skb` to be requeued for transmission later. Recall earlier we saw code in `dequeue_skb` that dealt specifically with requeued skbs.

The code for `handle_dev_cpu_collision` is short and worth a quick read:

```c
static inline int handle_dev_cpu_collision(struct sk_buff *skb,
                                           struct netdev_queue *dev_queue,
                                           struct Qdisc *q)
{
        int ret;

        if (unlikely(dev_queue->xmit_lock_owner == smp_processor_id())) {
                /*
                 * Same CPU holding the lock. It may be a transient
                 * configuration error, when hard_start_xmit() recurses. We
                 * detect it by checking xmit owner and drop the packet when
                 * deadloop is detected. Return OK to try the next skb.
                 */
                kfree_skb(skb);
                net_warn_ratelimited("Dead loop on netdevice %s, fix it urgently!\n",
                                     dev_queue->dev->name);
                ret = qdisc_qlen(q);
        } else {
                /*
                 * Another cpu is holding lock, requeue & delay xmits for
                 * some time.
                 */
                __this_cpu_inc(softnet_data.cpu_collision);
                ret = dev_requeue_skb(skb, q);
        }

        return ret;
}
```

Let’s take a look at what `dev_requeue_skb` does, as we’ll see this function called from `sch_direct_xmit`.

#### `dev_requeue_skb`

Thankfully, the source for `dev_requeue_skb` is short and straight to the point, from [./net/sched/sch_generic.c](https://github.com/torvalds/linux/blob/v3.13/net/sched/sch_generic.c#L39-L57):

```c
/* Modifications to data participating in scheduling must be protected with
 * qdisc_lock(qdisc) spinlock.
 *
 * The idea is the following:
 * - enqueue, dequeue are serialized via qdisc root lock
 * - ingress filtering is also serialized via qdisc root lock
 * - updates to tree and tree walking are only done under the rtnl mutex.
 */

static inline int dev_requeue_skb(struct sk_buff *skb, struct Qdisc *q)
{
        skb_dst_force(skb);
        q->gso_skb = skb;
        q->qstats.requeues++;
        q->q.qlen++;        /* it's still part of the queue */
        __netif_schedule(q);

        return 0;
}
```

This function does a few things:

1.  It forces a reference count on the skb.
2.  It attaches the skb to the qdisc’s `gso_skb` field. Recall earlier we saw that this field is checked in `dequeue_skb` before data is pulled off the qdisc’s queue.
3.  A statistics counter is bumped.
4.  The size of the queue is increased.
5.  `__netif_schedule` is called.

Simple and straightforward. Let’s refresh how we got here and then examine `__netif_schedule`.

### Reminder, while loop in `__qdisc_run`

Recall that we got here by examining the function `__qdisc_run` which contained the following code:

```c
void __qdisc_run(struct Qdisc *q)
{
        int quota = weight_p;

        while (qdisc_restart(q)) {
                /*
                 * Ordered by possible occurrence: Postpone processing if
                 * 1\. we've exceeded packet quota
                 * 2\. another process needs the CPU;
                 */
                if (--quota <= 0 || need_resched()) {
                        __netif_schedule(q);
                        break;
                }
        }

        qdisc_run_end(q);
}
```

This code works by repeatedly calling `qdisc_restart` in a loop which, internally, dequeues skbs, attempts to transmit them by calling `sch_direct_xmit`, which calls `dev_hard_start_xmit` to get down to the driver to do the actual transmit. Anything that could not be transmit is requeued to be transmit in the `NET_TX` softirq.

The next step in the transmit process is examining `dev_hard_start_xmit` to see how the drivers are invoked for sending data. Before doing that, we should examine `__netif_schedule` to fully understand how both `__qdisc_run` and `dev_requeue_skb` work.

#### `__netif_schedule`

Let’s jump into `__netif_schedule` from [./net/core/dev.c](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L2127-L2146):

```c
void __netif_schedule(struct Qdisc *q)
{
        if (!test_and_set_bit(__QDISC_STATE_SCHED, &q->state))
                __netif_reschedule(q);
}
EXPORT_SYMBOL(__netif_schedule);
```

This code checks and sets the `__QDISC_STATE_SCHED` bit in the qdisc’s state. If the bit was flipped (meaning that it was not previously in the `__QDISC_STATE_SCHED` state), the code will call `__netif_reschedule`, which is not much longer but has very interesting side effects. Let’s take a look:

```c
static inline void __netif_reschedule(struct Qdisc *q)
{
        struct softnet_data *sd;
        unsigned long flags;

        local_irq_save(flags);
        sd = &__get_cpu_var(softnet_data);
        q->next_sched = NULL;
        *sd->output_queue_tailp = q;
        sd->output_queue_tailp = &q->next_sched;
        raise_softirq_irqoff(NET_TX_SOFTIRQ);
        local_irq_restore(flags);
}
```

This function does several things:

1.  Save the current local IRQ state and disable IRQs with a call to `local_irq_save`.
2.  Get the current CPUs `softnet_data` structure.
3.  Add the qdisc to the `softnet_data`’s output queue.
4.  Raise the `NET_TX_SOFTIRQ` softirq.
5.  Restore the IRQ state and re-enable interrupts.

You can read more about [the initialization of the `softnet_data` data structures](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#linux-network-device-subsystem) by reading our previous post about the receive side of the networking stack.

The important piece of code in the above function is: `raise_softirq_irqoff` which triggers the `NET_TX_SOFTIRQ` softirq. [softirqs and their registration](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#softirqs) are also covered in our previous post. Briefly, you can think of softirqs are kernel threads that execute with a very high priority and process data on behalf of the kernel. They are used for processing incoming network data and also for processing outgoing data.

As you’ll see from the previous post, the `NET_TX_SOFTIRQ` softirq has the function `net_tx_action` registered to it. This means that there is a kernel thread executing `net_tx_action`. That thread is occasionally paused and `raise_softirq_irqoff` resumes it. Let’s take a look at what `net_tx_action` does so we can understand how the kernel processes transmit requests.

#### `net_tx_action`

The `net_tx_action` function from [./net/core/dev.c](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L3297-L3353) handles two main things when it runs:

1.  The completion queue of the `softnet_data` structure for the executing CPU.
2.  The output queue of the `softnet_data` structure for the executing CPU.

In fact, the code for the function is two large if blocks. Let’s take them one at a time, remembering all the while that this code is executing in the softirq context as an independent kernel thread. The purpose of `net_tx_action` is to execute code that cannot be executed in hot paths throughout the transmit side of the network stack; work is deferred and later processed by the thread executing `net_tx_action`.

##### `net_tx_action` completion queue

The `softnet_data`’s completion queue is simply a queue of skbs that are waiting to be freed. The function `dev_kfree_skb_irq` can be used to add skbs to a queue to be freed later. This is commonly used by device drivers to defer freeing consumed skbs. The reason why a driver would want to defer freeing the skb instead of simply freeing the skb is that freeing memory can take time and there are instances (like hardirq handlers) where code needs to execute as quickly as possible and return.

Take a look at the `net_tx_action` code which deals with freeing skbs on the completion queue:

```c
        if (sd->completion_queue) {
                struct sk_buff *clist;

                local_irq_disable();
                clist = sd->completion_queue;
                sd->completion_queue = NULL;
                local_irq_enable();

                while (clist) {
                        struct sk_buff *skb = clist;
                        clist = clist->next;

                        WARN_ON(atomic_read(&skb->users));
                        trace_kfree_skb(skb, net_tx_action);
                        __kfree_skb(skb);
                }
        }
```

If the completion queue has entries, the `while` loop will walk through the linked list of skbs and call `__kfree_skb` on each of them to free their memory. Remember, this code is running in a separate “thread” called a softirq – it is not running on behalf of any user program in particular.

##### `net_tx_action` output queue

The output queue serves a different purpose entirely. As we saw earlier, data is added to the output queue by calls to `__netif_reschedule`, which is typically called from `__netif_schedule`. The `__netif_schedule` function is called in two instances we’ve seen so far:

*   `dev_requeue_skb`: As we saw, this function can be called if the driver reports back the error code `NETDEV_TX_BUSY` or if there is a CPU collision.
*   `__qdisc_run`: We saw this function earlier, as well. It also calls `__netif_schedule` once the quota has been exceeded or if the process needs to be rescheduled.

In either of those cases, the `__netif_schedule` function will be called which will add the qdisc to the `softnet_data`’s output queue for processing. I’ve split out the output queue processing code into three blocks. Let’s take a look at the first:

```c
        if (sd->output_queue) {
                struct Qdisc *head;

                local_irq_disable();
                head = sd->output_queue;
                sd->output_queue = NULL;
                sd->output_queue_tailp = &sd->output_queue;
                local_irq_enable();
```

This block simply ensures that there are qdiscs on the output queue, and if so, it sets `head` to the first entry and moves the tail pointer of the queue.

Next, the `while` loop for traversing the list of qdsics starts:

```c
                while (head) {
                        struct Qdisc *q = head;
                        spinlock_t *root_lock;

                        head = head->next_sched;

                        root_lock = qdisc_lock(q);
                        if (spin_trylock(root_lock)) {
                                smp_mb__before_clear_bit();
                                clear_bit(__QDISC_STATE_SCHED,
                                          &q->state);
                                qdisc_run(q);
                                spin_unlock(root_lock);
```

The above section of code moves the head pointer forward and obtains a reference to the qdisc lock. `spin_trylock` is used to check if the lock can be obtained; note that this call is used specifically because it does not block. If the lock is already held, `spin_trylock` will return immediately instead of waiting to obtain the lock.

If `spin_trylock` successfully obtains the lock it returns a non-zero value. In this case, the qdisc’s state field has its `__QDISC_STATE_SCHED` bit flipped and `qdisc_run` is invoked which flips the `__QDISC___STATE_RUNNING` bit and kicks begins executing `__qdisc_run`.

This is important. What’s happening here is that the processing loop we examined before which was running on behalf of the system call made by the user is now running again, but in the softirq context because the skb transmit for this qdisc was unable to transmit. This distinction is important because it affects how you monitor CPU usage of applications which send large amounts of data. Let me state this another way:

*   Your program’s system time will include time spent calling down to the driver to try to send data, regardless of whether the send completes or the driver returns an error.
*   If that send is unsuccessful at the driver layer (e.g. because the device was busy sending something else), the qdisc will be added to the output queue and processed later by a softirq thread. In this case, softirq (si) time will be spent attempting to transmit your data.

So, the total time spent sending data is a combination of both the system time of send-related system calls and the softirq time for the `NET_TX` softirq.

At any rate, the code above completes by releasing the qdisc lock. If the `spin_trylock` call above falls to obtain the lock, the following code is executed:

```c
                        } else {
                                if (!test_bit(__QDISC_STATE_DEACTIVATED,
                                              &q->state)) {
                                        __netif_reschedule(q);
                                } else {
                                        smp_mb__before_clear_bit();
                                        clear_bit(__QDISC_STATE_SCHED,
                                                  &q->state);
                                }
                        }
                }
        }
```

This code, which only executes if the qdisc lock couldn’t be obtained, handles two cases. Either:

1.  The qdisc is not deactivated, but the lock couldn’t be obtained for executing `qdisc_run`. So, call `__netif_reschedule`. Calling `__netif_reschedule` here puts the qdisc back on the queue that this function is currently dequeuing from. This allows the qdisc to be checked again later when perhaps the lock has been given up.
2.  The qdisc is marked as deactivated, ensure that the `__QDISC_STATE_SCHED` state flag is cleared as well.

### Finally time to meet our friend `dev_hard_start_xmit`
So, we’ve traversed the entire network stack down to `dev_hard_start_xmit`. Maybe you’ve arrived here directly from a `sendmsg` system call or you arrived here via a softirq thread processing network data on the qdisc. `dev_hard_start_xmit` will call down to the device driver to actually do the transmit operation.

The `dev_hard_start_xmit` function handles two major cases:

*   Network data that is ready to send, or
*   Network data that has segmentation offloading that needs to be dealt with.

We’ll see how both cases are handled, starting with the case of network data that is ready to send. Let’s take a look (follow along here: [./net/code/dev.c](https://github.com/torvalds/linux/blob/v3.13/net/core/dev.c#L2541-L2652):

```c
int dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev,
                        struct netdev_queue *txq)
{
        const struct net_device_ops *ops = dev->netdev_ops;
        int rc = NETDEV_TX_OK;
        unsigned int skb_len;

        if (likely(!skb->next)) {
                netdev_features_t features;

                /*
                 * If device doesn't need skb->dst, release it right now while
                 * its hot in this cpu cache
                 */
                if (dev->priv_flags & IFF_XMIT_DST_RELEASE)
                        skb_dst_drop(skb);

                features = netif_skb_features(skb);
```

This code starts by obtaining a reference to the device driver’s exposed operations with `ops`. This will be used later when it’s time to get the driver to do some work to transmit data. The code checks `skb->next` to ensure that this data is not part of a chain of data that is segmented ready to go and moves on to do two things:

1.  First, it checks if the `IFF_XMIT_DST_RELEASE` flag is set on the device. This flag isn’t used by any of the “real” ethernet devices in this kernel. It used by the loopback device and some other software devices, though. If this flag is enabled, the reference count on the destination cache entry can be decreased, since it won’t be needed by the driver.
2.  Next, `netif_skb_features` is used to get the feature flags from the device and modify them a bit based on the protocol for which the data is destined (`dev->protocol`). For example, if the protocol is one the device can checksum for, the skb will marked as such. The VLAN tag (if it is set) will also cause additional feature flags to be flipped.

Next, the vlan tag will be checked and if the device can’t offload VLAN tagging, `__vlan_put_tag` will be used to do this in software:

```c
                if (vlan_tx_tag_present(skb) &&
                    !vlan_hw_offload_capable(features, skb->vlan_proto)) {
                        skb = __vlan_put_tag(skb, skb->vlan_proto,
                                             vlan_tx_tag_get(skb));
                        if (unlikely(!skb))
                                goto out;

                        skb->vlan_tci = 0;
                }
```

Following that, the data will be checked if it’s an encapsulation offload request, perhaps for [GRE](https://en.wikipedia.org/wiki/Generic_Routing_Encapsulation), for example. In this case, the feature flags will be updated to include any device-specific hardware encapsulation features that are available:

```c
                /* If encapsulation offload request, verify we are testing
                 * hardware encapsulation features instead of standard
                 * features for the netdev
                 */
                if (skb->encapsulation)
                        features &= dev->hw_enc_features;
```

Next, `netif_needs_gso` is used to determine whether or not an skb itself needs segmentation at all. If the skb needs segmentation, but the device does not support it, then `netif_needs_gso` will return `true` indicating that segmentation should occur in software. In this case, `dev_gso_segment` is called to do the segmentation and the code will jump down to `gso` to transmit the packets. We’ll see the GSO path later.

```c
                if (netif_needs_gso(skb, features)) {
                        if (unlikely(dev_gso_segment(skb, features)))
                                goto out_kfree_skb;
                        if (skb->next)
                                goto gso;
                }
```

If the data does not need segmentation, a few other cases are handled. First: does the data need to be linearized? That is, can the device support sending network data if the data is spread out across multiple buffers, or does it all need to be combined into a single linear buffer first? The vast majority of network cards do not required the data to be linearized before transmit, so in almost all cases this will evaluated to false and will be skipped.

```c
                                     else {
                        if (skb_needs_linearize(skb, features) &&
                            __skb_linearize(skb))
                                goto out_kfree_skb;
```

A helpful comment is provided next, explaining the next case. The packet will be checked to determine if it still needs a checksum. If the device does not support checksumming, a checksum will be generated in software now:

```c
                        /* If packet is not checksummed and device does not
                         * support checksumming for this protocol, complete
                         * checksumming here.
                         */
                        if (skb->ip_summed == CHECKSUM_PARTIAL) {
                                if (skb->encapsulation)
                                        skb_set_inner_transport_header(skb,
                                                skb_checksum_start_offset(skb));
                                else
                                        skb_set_transport_header(skb,
                                                skb_checksum_start_offset(skb));
                                if (!(features & NETIF_F_ALL_CSUM) &&
                                     skb_checksum_help(skb))
                                        goto out_kfree_skb;
                        }
                }
```

Now we move on to packet taps! Recall in the [receive side blog post](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#netifreceiveskbcore-special-box-delivers-data-to-packet-taps-and-protocol-layers), we saw how packets were passed off to packet taps (like [PCAP](http://www.tcpdump.org/manpages/pcap.3pcap.html)). The next chunk of code in this function hands packets which are about to be transmit over to the packet taps (if there are any).

```c
                if (!list_empty(&ptype_all))
                        dev_queue_xmit_nit(skb, dev);
```

Finally, the driver’s `ops` are used to pass the data down to the device by calling `ndo_start_xmit`:

```c
                skb_len = skb->len;
                rc = ops->ndo_start_xmit(skb, dev);

                trace_net_dev_xmit(skb, rc, dev, skb_len);
                if (rc == NETDEV_TX_OK)
                        txq_trans_update(txq);
                return rc;
        }
```

The return value of `ndo_start_xmit` is returned indicating whether the packet was transmit or not. We saw how this return value will affect the upper layers: the data will likely be requeued by the qdisc above this function so it can be transmit again later.

Let’s take a look at the GSO case. This code will run if the skb was already separated into a chain of packets due to segmentation which happened in this function or a packet that was previously segmented, but failed to send and was queued to be sent again.

```c
gso:
        do {
                struct sk_buff *nskb = skb->next;

                skb->next = nskb->next;
                nskb->next = NULL;

                if (!list_empty(&ptype_all))
                        dev_queue_xmit_nit(nskb, dev);

                skb_len = nskb->len;
                rc = ops->ndo_start_xmit(nskb, dev);
                trace_net_dev_xmit(nskb, rc, dev, skb_len);
                if (unlikely(rc != NETDEV_TX_OK)) {
                        if (rc & ~NETDEV_TX_MASK)
                                goto out_kfree_gso_skb;
                        nskb->next = skb->next;
                        skb->next = nskb;
                        return rc;
                }
                txq_trans_update(txq);
                if (unlikely(netif_xmit_stopped(txq) && skb->next))
                        return NETDEV_TX_BUSY;
        } while (skb->next);
```

As you may have guessed, this code is a while loop that iterates over the list of skbs that were generated when the data was segmented.

Each packet is:

*   Passed through the packet taps (if there are any).
*   Passed through to the driver via `ndo_start_xmit` to be transmit.

Any error in transmitting a packet is dealt with by adjusting the list of skbs that need to be sent. The error will be returned up the stack and the unsent skbs may be requeued to be sent again later.

The last piece of this function handles cleaning up and potentially freeing data in the event of any errors hit above:

```c
out_kfree_gso_skb:
        if (likely(skb->next == NULL)) {
                skb->destructor = DEV_GSO_CB(skb)->destructor;
                consume_skb(skb);
                return rc;
        }
out_kfree_skb:
        kfree_skb(skb);
out:
        return rc;
}
EXPORT_SYMBOL_GPL(dev_hard_start_xmit);
```

Before continuing into the device driver, let’s take a look at some monitoring and tuning that can be done for the code that we just walked through.

### Monitoring qdiscs
#### Using the `tc` command line tool

Monitor your qdisc statistics by using `tc`

```sh
$ tc -s qdisc show dev eth1
qdisc mq 0: root
 Sent 31973946891907 bytes 2298757402 pkt (dropped 0, overlimits 0 requeues 1776429)
 backlog 0b 0p requeues 1776429
```

In order to monitor the packet transmit health of your system, it is vital to examine the statistics of the queue discipline(s) attached to your network device(s). You can check the status by running the command line tool `tc`. The example above shows how to check the statistics for the `eth1` interface.

*   `bytes`: The number of bytes that were pushed down to the driver for transmit.
*   `pkt`: The number of packets that were pushed down to the driver for transmit.
*   `dropped`: The number of packets that were dropped by the qdisc. This can happen if transmit queue length is not large enough to fit the data being queued to it.
*   `overlimits`: Depends on the queuing discipline, but can be either the number of packets that could not be enqueued due to a limit being hit, and/or the number of packets which triggered a throttling event when dequeued.
*   `requeues`: Number of times `dev_requeue_skb` has been called to requeue an skb. Note that an skb which is requeued multiple times will bump this counter each time it is requeued.
*   `backlog`: Number of bytes currently on the qdisc’s queue. This number is usually bumped each time a packet is enqueued.

Some qdsics may export additional statistics. Each qdisc is different and may bump these counters at different times. You may want to study the source for the qdisc you are using to understand precisely when these values can be incremented on your system to help understand what the consequences are for you.

### Tuning qdiscs

#### Increasing the processing weight of `__qdisc_run`

You can adjust the weight of `__qdisc_run` loop seen earlier (the `quota` variable seen above) which will cause more calls to `__netif_schedule` to be executed. The result will be the current qdisc added to the `output_queue` list for the current CPU more times, which should result in additional processing of transmit packets.

Example: increase the `__qdisc_run` quota for all qdiscs with `sysctl`.

`$ sudo sysctl -w net.core.dev_weight=600`

#### Increasing the transmit queue length

Each network device has a `txqueuelen` tuning knob that can be modified. Most qdisc’s will check if the device has sufficient `txqueuelen` bytes when enqueuing data that should eventually be transmit by the qdisc. You can adjust his parameter to increase the number of bytes that may be queued by a qdisc.

Example: increase the `txqueuelen` of `eth0` to `10000`.

`$ sudo ifconfig eth0 txqueuelen 10000`

The default value for ethernet devices is `1000`. You can check the `txqueuelen` for network devices by reading the output of `ifconfig`.

## Network Device Driver

We’re nearing the end of our journey. There’s an important concept to understand about packet transmit. Most devices and drivers deal with packet transmit as a two-step process:

1.  Data is arranged properly and the device is triggered to DMA the data from RAM and write it to the network
2.  After the transmit completes, the device will raise an interrupt so the driver can unmap buffers, free memory, or otherwise clean its state.

The second phase of this is commonly called the “transmit completion” phase. We’re going to examine both, but we’ll start with the first phase: the transmit phase.

We saw that `dev_hard_start_xmit` calls the `ndo_start_xmit` (with a lock held) to transmit data, so let’s start by examining how a driver registers an `ndo_start_xmit` and then we’ll dive into how that function works.

As in [the previous blog post](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/) we’ll be examining the `igb` driver.

### Driver operations registration

Drivers implement a series of functions for a variety of operations, like:

*   Sending data (`ndo_start_xmit`)
*   Getting statistical information (`ndo_get_stats64`)
*   Handling device `ioctls` (`ndo_do_ioctl`)
*   And more.

The functions are exported as a series of function pointers arranged in a structure. Let’s take a look at the structure defintion for these operations in the `igb` [driver source](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L1905-L1928):

```c
static const struct net_device_ops igb_netdev_ops = {
        .ndo_open               = igb_open,
        .ndo_stop               = igb_close,
        .ndo_start_xmit         = igb_xmit_frame,
        .ndo_get_stats64        = igb_get_stats64,

                /* ... more fields ... */
};
```

This structure is registered in the [`igb_probe`](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L2090) function:

```c
static int igb_probe(struct pci_dev *pdev, const struct pci_device_id *ent)
{
                /* ... lots of other stuff ... */

        netdev->netdev_ops = &igb_netdev_ops;

                /* ... more code ... */
}
```

As we saw in the previous section, higher layers of code will obtain a refernece to a device’s `netdev_ops` structure and call the appropriate function. If you are curious to learn more about how exactly PCI devices are brought up and when/where `igb_probe` is called, check out [the driver initialization](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#initialization) section from our other blog post.

### Transmit data with `ndo_start_xmit`

The higher layers of the networking stack use the `net_device_ops` structure to call into a driver to perform various operations. As we saw earlier, the qdisc code calls `ndo_start_xmit` to pass data down to the driver for transmit. The `ndo_start_xmit` function is called while a lock is held, for most hardware devices, as we saw above.

In the `igb` device driver, the function registered to `ndo_start_xmit` is called `igb_xmit_frame`, so let’s start at `igb_xmit_frame` and learn how this driver transmits data. Follow along in [./drivers/net/ethernet/intel/igb/igb_main.c](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L4664-L4741) and keep in mind that a lock is beind held the entire time the following code is executing:

```c
netdev_tx_t igb_xmit_frame_ring(struct sk_buff *skb,
                                struct igb_ring *tx_ring)
{
        struct igb_tx_buffer *first;
        int tso;
        u32 tx_flags = 0;
        u16 count = TXD_USE_COUNT(skb_headlen(skb));
        __be16 protocol = vlan_get_protocol(skb);
        u8 hdr_len = 0;

        /* need: 1 descriptor per page * PAGE_SIZE/IGB_MAX_DATA_PER_TXD,
         *       + 1 desc for skb_headlen/IGB_MAX_DATA_PER_TXD,
         *       + 2 desc gap to keep tail from touching head,
         *       + 1 desc for context descriptor,
         * otherwise try next time
         */
        if (NETDEV_FRAG_PAGE_MAX_SIZE > IGB_MAX_DATA_PER_TXD) {
                unsigned short f;
                for (f = 0; f < skb_shinfo(skb)->nr_frags; f++)
                        count += TXD_USE_COUNT(skb_shinfo(skb)->frags[f].size);
        } else {
                count += skb_shinfo(skb)->nr_frags;
        }
```

The function starts out be determining use the `TXD_USER_COUNT` macro to determine how many transmit descriptors will be needed to transmit the data passed in. The `count` value initialized at the the number of descriptors to fit the skb. It is then adjusted to account for any additional fragments that need to be transmit.

```c
        if (igb_maybe_stop_tx(tx_ring, count + 3)) {
                /* this is a hard error */
                return NETDEV_TX_BUSY;
        }
```

The driver then calls an internal function `igb_maybe_stop_tx` which will check the number of descriptors needed to ensure that the transmit queue has enough available. If not, `NETDEV_TX_BUSY` is returned here. As we saw earlier in the qdisc code, this will cause the qdisc to requeue the data to be retried later.

```c
        /* record the location of the first descriptor for this packet */
        first = &tx_ring->tx_buffer_info[tx_ring->next_to_use];
        first->skb = skb;
        first->bytecount = skb->len;
        first->gso_segs = 1;
```

The code then obtains a regerence to the next available buffer info in the transmit queue. This structure will track the information needed for setting up a buffer descriptor later. A reference to the packet and its size are copied into the buffer info structure.

```c
        skb_tx_timestamp(skb);
```

The code above starts by calling `skb_tx_timestamp` which is used to obtain a software based transmit timestamp. An application can use the transmit timestamp to determine the amount of time it takes for a packet to travel through the transmit path of the network stack.

Some devices also support generating timestamps for packets transmit in hardware. This allows the system to offload timestamping to the device and it allows the programmer to obtain a more accurate timestamp, as it will be taken much closer to when the actual transmit by the hardware occurs. We’ll see the code for this now:

```c
        if (unlikely(skb_shinfo(skb)->tx_flags & SKBTX_HW_TSTAMP)) {
                struct igb_adapter *adapter = netdev_priv(tx_ring->netdev);

                if (!(adapter->ptp_tx_skb)) {
                        skb_shinfo(skb)->tx_flags |= SKBTX_IN_PROGRESS;
                        tx_flags |= IGB_TX_FLAGS_TSTAMP;

                        adapter->ptp_tx_skb = skb_get(skb);
                        adapter->ptp_tx_start = jiffies;
                        if (adapter->hw.mac.type == e1000_82576)
                                schedule_work(&adapter->ptp_tx_work);
                }
        }
```

Some network devices can timestamp packets in hardware using the [Precision Time Protocol](https://events.linuxfoundation.org/sites/events/files/slides/lcjp14_ichikawa_0.pdf). The driver code handles that here when a user requests hardware timestampping.

The `if` statement above checks for the `SKBTX_HW_TSTAMP` flag. This flag indicates that the user requested hardware timestamping. If the user requested hardware timestamping, the code will next check if `ptp_tx_skb` is set. One packet can be timestampped at a time, so a reference to the packet being timestampped is taken here and the `SKBTX_IN_PROGRESS` flag is set on the skb. The `tx_flags` are updated to mark the `IGB_TX_FLAGS_TSTAMP` flag. The `tx_flags` variable will be copied into the buffer info structure later.

A reference is taken to the skb, the current jiffies count is copied to `ptp_tx_start`. This value will be used by other code in the driver to ensure that the TX hardware timestampping is not hanging. Finally, the `schedule_work` function is used to kick the [workqueue](http://www.makelinux.net/ldd3/chp-7-sect-6) if this is an `82576` ethernet hardware adapter.

```c
        if (vlan_tx_tag_present(skb)) {
                tx_flags |= IGB_TX_FLAGS_VLAN;
                tx_flags |= (vlan_tx_tag_get(skb) << IGB_TX_FLAGS_VLAN_SHIFT);
        }
```

The code above will check if the `vlan_tci` field of the skb was set. If it is set, then the `IGB_TX_FLAGS_VLAN` flag is enabled and the vlan ID is stored.

```c
        /* record initial flags and protocol */
        first->tx_flags = tx_flags;
        first->protocol = protocol;
```

The flags and protocol are recorded to the buffer info structure.

```c
        tso = igb_tso(tx_ring, first, &hdr_len);
        if (tso < 0)
                goto out_drop;
        else if (!tso)
                igb_tx_csum(tx_ring, first);
```

Next, the driver calls its internal function `igb_tso`. This function will determine if an skb needs fragmentation. If so, the buffer info reference (`first`) will have its flags updated to indicate to the hardware that [TSO](https://en.wikipedia.org/wiki/Large_segment_offload) is required.

`igb_tso` will return `0` is TSO is unncessary, otherwise `1` is returned. If `0` is returned, `igb_tx_csum` will be called to deal with enabling checksum offloading if needed and if supported for this protocol. The `igb_tx_csum` function will check the properties of the skb and flip some flag bits in the buffer info `first` to signal that checksum offloading is needed.

```c
        igb_tx_map(tx_ring, first, hdr_len);
```

The `igb_tx_map` function is called to prepare the data to be consumed by the device for transmit. We’ll examine this function in detail next.

```c
        /* Make sure there is space in the ring for the next send. */
        igb_maybe_stop_tx(tx_ring, DESC_NEEDED);

        return NETDEV_TX_OK;
```

Once the the transmit is complete, the driver checks to ensure that there is sufficient space available for another transmit. If not, the queue is shutdown. In either case `NETDEV_TX_OK` is returned to the higher layer (the qdisc code).

```c
out_drop:
        igb_unmap_and_free_tx_resource(tx_ring, first);

        return NETDEV_TX_OK;
}
```

Finally, some error handling code. This code is only hit if `igb_tso` hits an error of some kind. The `igb_unmap_and_free_tx_resource` is used to clean up data. `NETDEV_TX_OK` is returned in this case as well. The transmit was not successful, but the driver freed the resources associated and there is nothing left to do. Note that this driver does not increment packet drops in this case, but it probably should.

### `igb_tx_map`
The `igb_tx_map` function handles the details of mapping skb data to DMA-able regions of RAM. It also updates the transmit queue’s tail pointer on the device, which is what triggers the device to “wake up”, fetch the data from RAM, and begin transmitting the data.

Let’s take a look, briefly, at how [this function](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L4501-L4627) works:

```c
static void igb_tx_map(struct igb_ring *tx_ring,
                       struct igb_tx_buffer *first,
                       const u8 hdr_len)
{
        struct sk_buff *skb = first->skb;

                /* ... other variables ... */

        u32 tx_flags = first->tx_flags;
        u32 cmd_type = igb_tx_cmd_type(skb, tx_flags);
        u16 i = tx_ring->next_to_use;

        tx_desc = IGB_TX_DESC(tx_ring, i);

        igb_tx_olinfo_status(tx_ring, tx_desc, tx_flags, skb->len - hdr_len);

        size = skb_headlen(skb);
        data_len = skb->data_len;

        dma = dma_map_single(tx_ring->dev, skb->data, size, DMA_TO_DEVICE);
```

The code above does a few things:

1.  Declares a set of variables and initializes them.
2.  Uses the `IGB_TX_DESC` macro to determine obtain a reference to the next available descriptor.
3.  `igb_tx_olinfo_status` will update the `tx_flags` and copy them into the descriptor (`tx_desc`).
4.  The size and data length are captured so they can be used later.
5.  `dma_map_single` is used to construct any memory mapping necessary to obtain a DMA-able address for `skb->data`. This is done so that the device can read the packet data from memory.

What follows next is a very dense loop in the driver to deal with generating a valid mapping for each fragment of a skb. The details of how exactly this happens are not particularly important, but are worth mentioning:

*   The driver iterates across the set of packet fragments.
*   The current descriptor has the DMA address of the data filled in.
*   If the size of the fragment is larger than what a single IGB descriptor can transmit, multiple descriptors are constructed to point to chunks of the DMA-able region until the entire fragment is pointed to by descriptors.
*   The descriptor iterator is bumped.
*   The remaining length is reduced.
*   The loop terminates when either: no fragments are remaining or the entire data length has been consumed.

The code for the loop is provided below for reference with the above description. This should illustrate further to readers that avoiding fragmentation, if at all possible, is a good idea. Lots of additional code needs to run to deal with it at every layer of the stack, including the driver.

```c
        tx_buffer = first;

        for (frag = &skb_shinfo(skb)->frags[0];; frag++) {
                if (dma_mapping_error(tx_ring->dev, dma))
                        goto dma_error;

                /* record length, and DMA address */
                dma_unmap_len_set(tx_buffer, len, size);
                dma_unmap_addr_set(tx_buffer, dma, dma);

                tx_desc->read.buffer_addr = cpu_to_le64(dma);

                while (unlikely(size > IGB_MAX_DATA_PER_TXD)) {
                        tx_desc->read.cmd_type_len =
                                cpu_to_le32(cmd_type ^ IGB_MAX_DATA_PER_TXD);

                        i++;
                        tx_desc++;
                        if (i == tx_ring->count) {
                                tx_desc = IGB_TX_DESC(tx_ring, 0);
                                i = 0;
                        }
                        tx_desc->read.olinfo_status = 0;

                        dma += IGB_MAX_DATA_PER_TXD;
                        size -= IGB_MAX_DATA_PER_TXD;

                        tx_desc->read.buffer_addr = cpu_to_le64(dma);
                }

                if (likely(!data_len))
                        break;

                tx_desc->read.cmd_type_len = cpu_to_le32(cmd_type ^ size);

                i++;
                tx_desc++;
                if (i == tx_ring->count) {
                        tx_desc = IGB_TX_DESC(tx_ring, 0);
                        i = 0;
                }
                tx_desc->read.olinfo_status = 0;

                size = skb_frag_size(frag);
                data_len -= size;

                dma = skb_frag_dma_map(tx_ring->dev, frag, 0,
                                       size, DMA_TO_DEVICE);

                tx_buffer = &tx_ring->tx_buffer_info[i];
        }
```

Once all the necessary descriptors have been constructed and all of the skb’s data has been mapped to DMA-able addresses, the driver proceeds to its final steps to trigger a transmit:

```c
        /* write last descriptor with RS and EOP bits */
        cmd_type |= size | IGB_TXD_DCMD;
        tx_desc->read.cmd_type_len = cpu_to_le32(cmd_type);
```

A terminating descriptor is written to indicate to the device that it is the last descriptor.

```c
        netdev_tx_sent_queue(txring_txq(tx_ring), first->bytecount);

        /* set the timestamp */
        first->time_stamp = jiffies;
```

The `netdev_tx_sent_queue` function is called with the number of bytes being added to this transmit queue. This function is part of the byte query limit feature that we’ll see shortly in more detail. The current jiffies are stored in the first buffer info structure.

Next, something a bit tricky:

```c
        /* Force memory writes to complete before letting h/w know there
         * are new descriptors to fetch.  (Only applicable for weak-ordered
         * memory model archs, such as IA-64).
         *
         * We also need this memory barrier to make certain all of the
         * status bits have been updated before next_to_watch is written.
         */
        wmb();

        /* set next_to_watch value indicating a packet is present */
        first->next_to_watch = tx_desc;

        i++;
        if (i == tx_ring->count)
                i = 0;

        tx_ring->next_to_use = i;

        writel(i, tx_ring->tail);

        /* we need this if more than one processor can write to our tail
         * at a time, it synchronizes IO on IA64/Altix systems
         */
        mmiowb();

        return;
```

The code above is doing a few important things:

1.Start by using the `wmb` function is called to force memory writes to complete. This executed as a special instruction appropriate for the CPU platform and is commonly referred to as a “write barrier.” This is important on certain CPU architectures because if we trigger the device to start DMA without ensuring that all memory writes to update internal state have completed, the device may read data from RAM that is not in a consistent state. [This article](http://preshing.com/20120930/weak-vs-strong-memory-models/) and this [lecture](http://www.cs.utexas.edu/~pingali/CS378/2012fa/lectures/consistency.pdf) dive into the details on memory ordering.

1.  The `next_to_watch` field is set. It will be used later during the completion phase.
2.  Counters are bumped, and the `next_to_use` field of the transmit queue is updated to the next available descriptor.
3.  The transmit queue’s tail is updated with a `writel` function. `writel` writes a “long” to a [memory mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) address. In this case, the address is `tx_ring->tail` (which is a hardware address) and the value to be written is `i`. This write to the device triggers the device to let it know that additional data is ready to be DMA’d from RAM and written to the network.
4.  Finally, call the `mmiowb` function. This function will execute the appropriate instruction for the CPU architecture causing memory mapped write operations to be ordered. It is also a write barrier, but for memory mapped I/O writes.

You can read some excellent [documentation about memory barriers](https://github.com/torvalds/linux/blob/v3.13/Documentation/memory-barriers.txt) included with the Linux kernel, if you are curious to learn more about `wmb`, `mmiowb`, and when to use them.

Finally, the code wraps up with some error handling. This code only executes if an error was returned from the DMA API when attemtping to map skb data addresses to DMA-able addresses.

```c
dma_error:
        dev_err(tx_ring->dev, "TX DMA map failed\n");

        /* clear dma mappings for failed tx_buffer_info map */
        for (;;) {
                tx_buffer = &tx_ring->tx_buffer_info[i];
                igb_unmap_and_free_tx_resource(tx_ring, tx_buffer);
                if (tx_buffer == first)
                        break;
                if (i == 0)
                        i = tx_ring->count;
                i--;
        }

        tx_ring->next_to_use = i;
```

Before moving on to the transmit completion, let’s examine something we passed over above: dynamic queue limits.

#### Dynamic Queue Limits (DQL)
As you’ve seen throughout this post, network data spends a lot of time sitting queues at various stages as it moves closer and closer to the device for transmission. As queue sizes increase, packets spend longer sitting in queues not being transmit i.e. packet transmit latency increases as queue size increases.

One way to fight this is with back pressure. The dynamic queue limit (DQL) system is a mechanism that device drivers can use to apply back pressure to the networking system to prevent too much data from being queued for transmit when the device is unable to transmit,

To use this system, network device drivers need to make a few simple API calls during their transmit and completion routines. The DQL system internally will use an algorithm to determine when sufficient data is in transmit. Once this limit is reached, the transmit queue will be temporarily disabled. This queue disabling is what produces the back pressure against the networking system. The queue will be automatically re-enabled when the DQL system determines enough data has finished transmission.

Check out this excellent set of [slides about the DQL system](https://www.linuxplumbersconf.org/2012/wp-content/uploads/2012/08/bql_slide.pdf) for some performance data and an explanation of the internal algorithm in DQL.

The function `netdev_tx_sent_queue` called in the code we just saw is part of the DQL API. This function is called when data is queued to the device for transmit. Once the transmit completes, the driver calls `netdev_tx_completed_queue`. Internally, both of these functions will call into the DQL library (found in [./lib/dynamic_queue_limits.c](https://github.com/torvalds/linux/blob/v3.13/lib/dynamic_queue_limits.c) and [./include/linux/dynamic_queue_limits.h](https://github.com/torvalds/linux/blob/v3.13/include/linux/dynamic_queue_limits.h)) to determine if the transmit queue should be disabled, re-enabled, or left as-is.

DQL exports statistics and tuning knobs in sysfs. Tuning DQL should not be necessary; the algorithm will adjust its parameters over time. Nevertheless, in the interest of completeness we’ll see later how to monitor and tune DQL.

### Transmit completions

Once the device has transmit the data, it will generate an interrupt to signal that transmission is complete. The device driver can then schedule some long running work to be completed, like unmapping memory regions and freeing data. How exactly this works is device specific. In the case of the `igb` driver (and its associated devices), the same IRQ is fired for transmit completion and packet receive. This means that for the `igb` driver the `NET_RX` is used to process _both_ transmit completions and incoming packet receives.

Let me re-state that to emphasize the importance of this: your device may raise the same interrupt for receiving packets that it raises to signal that a packet transmit has completed. If it does, the `NET_RX` softirq runs to process _both_ incoming packets and transmit completions.

Since both operations share the same IRQ, only a single IRQ handler function can be registered and it must deal with both possible cases. Recall the following flow when network data is received:

1.  Network data is received.
2.  The network device raises an IRQ.
3.  The device driver’s IRQ handler executes, clearing the IRQ and ensuring that a [softIRQ](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#softirqs) is scheduled to run (if not running already). This softIRQ that is triggered here is the `NET_RX` softIRQ.
4.  The softIRQ executes essentially as a separate kernel thread. It runs and implements the [NAPI](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#napi) poll loop.
5.  The NAPI poll loop is simply a piece of code which executes in loop harvesting packets as long as sufficient budget is available.
6.  Each time a packet is processed, the budget is decreased until there are no more packets to process, the budget reaches 0, or the time slice has expired.

Step 5 above in the `igb` driver (and the `ixgbe` driver [greetings, tyler]) processes TX completions before processing incoming data. Keep in mind that depending on the implementation of the driver, both processing functions for TX completions and incoming data may share the same processing budget. The `igb` and `ixgbe` drivers track the TX completion and incoming packet budgets separately, so processing TX completions will not necessarily exhaust the RX budget.

That said, the entire NAPI poll loop runs [within a hard coded time slice](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#netrxaction-processing-loop). This means that if you have a lot of TX completion processing to handle, TX completions may eat more of the time slice than processing incoming data does. This may be an important consideration for those running network hardware in very high load environments.

Let’s see how this happens in practice for the `igb` driver.

#### Transmit completion IRQ

Instead of restating information already covered in the [Linux kernel receive side networking blog post](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/), this post will instead list the steps in order and link to the appropriate sections in the receive side blog post until transmit completions are reached.

So, let’s start from the beginning:

1.  The network device is [brought up](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#bringing-a-network-device-up).
2.  [IRQ handlers are registered](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#register-an-interrupt-handler).
3.  The user program sends data to a network socket. The data travels the network stack until the device fetches it from memory and transmits it.
4.  The device finishes transmitting the data and raises an IRQ to signal transmit completion.
5.  The driver’s [IRQ handler executes to handle the interrupt](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#interrupt-handler).
6.  The IRQ handler calls `napi_schedule` in response to the IRQ.
7.  The [NAPI code](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#napi-and-napischedule) triggers the `NET_RX` softirq to execute.
8.  The `NET_RX` sofitrq function, `net_rx_action` [begins to execute](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#network-data-processing-begins).
9.  The `net_rx_action` function calls the [driver’s registered NAPI poll function](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#napi-poll-function-and-weight).
10.  The NAPI poll function, `igb_poll`, [is executed](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#igbpoll).

The poll function `igb_poll` is where the code splits off and processes both incoming packets and transmit completions. Let’s dive into the code for this function and see where that happens.

#### `igb_poll`

Let’s take a look at the code for `igb_poll` (from [./drivers/net/ethernet/intel/igb/igb_main.c](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L5987-L6018)):

```c
/**
 *  igb_poll - NAPI Rx polling callback
 *  @napi: napi polling structure
 *  @budget: count of how many packets we should handle
 **/
static int igb_poll(struct napi_struct *napi, int budget)
{
        struct igb_q_vector *q_vector = container_of(napi,
                                                     struct igb_q_vector,
                                                     napi);
        bool clean_complete = true;

#ifdef CONFIG_IGB_DCA
        if (q_vector->adapter->flags & IGB_FLAG_DCA_ENABLED)
                igb_update_dca(q_vector);
#endif
        if (q_vector->tx.ring)
                clean_complete = igb_clean_tx_irq(q_vector);

        if (q_vector->rx.ring)
                clean_complete &= igb_clean_rx_irq(q_vector, budget);

        /* If all work not completed, return budget and keep polling */
        if (!clean_complete)
                return budget;

        /* If not enough Rx work done, exit the polling mode */
        napi_complete(napi);
        igb_ring_irq_enable(q_vector);

        return 0;
}
```

This function performs a few operations, in order:

1.  If [Direct Cache Access (DCA)](https://lwn.net/Articles/247493/) support is enabled in the kernel, the CPU cache is warmed so that accesses to the RX ring will hit CPU cache. You can [read more about DCA in the Extras section](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#direct-cache-access-dca) of the receive side networking post.
2.  `igb_clean_tx_irq` is called which performs the transmit completion operations.
3.  `igb_clean_rx_irq` is called next which performs the incoming packet processing.
4.  Finally, `clean_complete` is checked to determine if there was still more work that could have been done. If so, the `budget` is returned. If this happens, `net_rx_action` will move this NAPI structure to the end of the poll list to be processed again later.

To learn more about how `igb_clean_rx_irq` works, read [this section](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#igbcleanrxirq) of the previous blog post.

This blog post is concerned primarily with the transmit side, so we’ll continue by examining how `igb_clean_tx_irq` above works.

#### `igb_clean_tx_irq`

Take a look at the source for this function in [./drivers/net/ethernet/intel/igb/igb_main.c](https://github.com/torvalds/linux/blob/v3.13/drivers/net/ethernet/intel/igb/igb_main.c#L6020-L6189).

It’s a bit long, so we’ll break it into chunks and go through it:

```c
static bool igb_clean_tx_irq(struct igb_q_vector *q_vector)
{
        struct igb_adapter *adapter = q_vector->adapter;
        struct igb_ring *tx_ring = q_vector->tx.ring;
        struct igb_tx_buffer *tx_buffer;
        union e1000_adv_tx_desc *tx_desc;
        unsigned int total_bytes = 0, total_packets = 0;
        unsigned int budget = q_vector->tx.work_limit;
        unsigned int i = tx_ring->next_to_clean;

        if (test_bit(__IGB_DOWN, &adapter->state))
                return true;
```

The function begins by initializing some useful variables. One important one to take a look at is `budget`. As you can see above `budget` is initialized to this queue’s `tx.work_limit`. In the `igb` driver, `tx.work_limit` is initialized to a hardcoded value `IGB_DEFAULT_TX_WORK` (128).

It is important to note that while the TX completion code we are looking at now runs in the same `NET_RX` softirq as receive processing does, the TX and RX functions do not share a processing budget with each other in the `igb` driver. Since the entire poll function runs within the same [time slice](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#netrxaction-processing-loop), it is not possible for a single run of the `igb_poll` function to starve incoming packet processing or transmit completions. As long as `igb_poll` is called, both will be handled.

Moving on, the snippet of code above finishes by checking if the network device is down. If so, it returns `true` and exits `igb_clean_tx_irq`.

```c
        tx_buffer = &tx_ring->tx_buffer_info[i];
        tx_desc = IGB_TX_DESC(tx_ring, i);
        i -= tx_ring->count;
```

1.  The `tx_buffer` variable is initialized to transmit buffer info structure at location `tx_ring->next_to_clean` (which itself is initialized to `0`).
2.  A reference to the associated descriptor is obtained and stored in `tx_desc`.
3.  The counter `i` is decreased by the size of the transmit queue. This value can be adjusted (as we’ll see in the tuning section), but is initialized to `IGB_DEFAULT_TXD` (256).

Next, a loop begins. It includes some helpful comments to explain what is happening at each step:

```c
        do {
                union e1000_adv_tx_desc *eop_desc = tx_buffer->next_to_watch;

                /* if next_to_watch is not set then there is no work pending */
                if (!eop_desc)
                        break;

                /* prevent any other reads prior to eop_desc */
                read_barrier_depends();

                /* if DD is not set pending work has not been completed */
                if (!(eop_desc->wb.status & cpu_to_le32(E1000_TXD_STAT_DD)))
                        break;

                /* clear next_to_watch to prevent false hangs */
                tx_buffer->next_to_watch = NULL;

                /* update the statistics for this packet */
                total_bytes += tx_buffer->bytecount;
                total_packets += tx_buffer->gso_segs;

                /* free the skb */
                dev_kfree_skb_any(tx_buffer->skb);

                /* unmap skb header data */
                dma_unmap_single(tx_ring->dev,
                                 dma_unmap_addr(tx_buffer, dma),
                                 dma_unmap_len(tx_buffer, len),
                                 DMA_TO_DEVICE);

                /* clear tx_buffer data */
                tx_buffer->skb = NULL;
                dma_unmap_len_set(tx_buffer, len, 0);
```

1.  First `eop_desc` is set to the buffer’s `next_to_watch` field. This was set in the transmit code we saw earlier.
2.  If `eop_desc` (eop = end of packet) is `NULL`, then there is no work pending.
3.  The `read_barrier_depends` function is called, which will execute the appropriate CPU instruction for this CPU architecture to prevent reads from being reordered over this barrier.
4.  Next, a status bit is checked in the end of packet descriptor `eop_desc`. If the `E1000_TXD_STAT_DD` bit is not set, then the transmit has not completed yet, so break from the loop.
5.  Clear the `tx_buffer->next_to_watch`. A watchdog timer in the driver will be watching this field to determine if a transmit was hung. Clearing this field will prevent the watchdog from triggering.
6.  Statistics counters are updated for total bytes and packets sent. These will be copied into the statistics counters that the driver reads once all descriptors have been processed.
7.  The skb is freed.
8.  `dma_unmap_single` is used to unmap the skb data region.
9.  The `tx_buffer->skb` is set to `NULL` and the `tx_buffer` is unmapped.

Next, another loop is started inside of the loop above:

```c
                /* clear last DMA location and unmap remaining buffers */
                while (tx_desc != eop_desc) {
                        tx_buffer++;
                        tx_desc++;
                        i++;
                        if (unlikely(!i)) {
                                i -= tx_ring->count;
                                tx_buffer = tx_ring->tx_buffer_info;
                                tx_desc = IGB_TX_DESC(tx_ring, 0);
                        }

                        /* unmap any remaining paged data */
                        if (dma_unmap_len(tx_buffer, len)) {
                                dma_unmap_page(tx_ring->dev,
                                               dma_unmap_addr(tx_buffer, dma),
                                               dma_unmap_len(tx_buffer, len),
                                               DMA_TO_DEVICE);
                                dma_unmap_len_set(tx_buffer, len, 0);
                        }
                }
```

This inner loop will loop over each transmit descriptor until `tx_desc` arrives at the `eop_desc`. This code unmaps data referenced by any of the additional descriptors.

The outer loop continues:

```c
                /* move us one more past the eop_desc for start of next pkt */
                tx_buffer++;
                tx_desc++;
                i++;
                if (unlikely(!i)) {
                        i -= tx_ring->count;
                        tx_buffer = tx_ring->tx_buffer_info;
                        tx_desc = IGB_TX_DESC(tx_ring, 0);
                }

                /* issue prefetch for next Tx descriptor */
                prefetch(tx_desc);

                /* update budget accounting */
                budget--;
        } while (likely(budget));
```

The outer loop increments iterators and reduces the `budget` value. The loop invariant is checked to determine if the loop should continue.

```c
        netdev_tx_completed_queue(txring_txq(tx_ring),
                                  total_packets, total_bytes);
        i += tx_ring->count;
        tx_ring->next_to_clean = i;
        u64_stats_update_begin(&tx_ring->tx_syncp);
        tx_ring->tx_stats.bytes += total_bytes;
        tx_ring->tx_stats.packets += total_packets;
        u64_stats_update_end(&tx_ring->tx_syncp);
        q_vector->tx.total_bytes += total_bytes;
        q_vector->tx.total_packets += total_packets;
```

This code:

1.  Calls `netdev_tx_completed_queue`, which is part of the DQL API explained above. This will potentially re-enable a transmit queue if enough completions were processed.
2.  Statistics are added to their appropriate places so that they can be accessed by the user as we’ll see later.

The code continues by first checking if the `IGB_RING_FLAG_TX_DETECT_HANG` flag is set. A watchdog timer sets this flag each time the timer callback is run, to enforce periodic checking of the transmit queue. If that flag happens to be on now, the code will continue and check if the transmit queue is hung:

```c
        if (test_bit(IGB_RING_FLAG_TX_DETECT_HANG, &tx_ring->flags)) {
                struct e1000_hw *hw = &adapter->hw;

                /* Detect a transmit hang in hardware, this serializes the
                 * check with the clearing of time_stamp and movement of i
                 */
                clear_bit(IGB_RING_FLAG_TX_DETECT_HANG, &tx_ring->flags);
                if (tx_buffer->next_to_watch &&
                    time_after(jiffies, tx_buffer->time_stamp +
                               (adapter->tx_timeout_factor * HZ)) &&
                    !(rd32(E1000_STATUS) & E1000_STATUS_TXOFF)) {

                        /* detected Tx unit hang */
                        dev_err(tx_ring->dev,
                                "Detected Tx Unit Hang\n"
                                "  Tx Queue             <%d>\n"
                                "  TDH                  <%x>\n"
                                "  TDT                  <%x>\n"
                                "  next_to_use          <%x>\n"
                                "  next_to_clean        <%x>\n"
                                "buffer_info[next_to_clean]\n"
                                "  time_stamp           <%lx>\n"
                                "  next_to_watch        <%p>\n"
                                "  jiffies              <%lx>\n"
                                "  desc.status          <%x>\n",
                                tx_ring->queue_index,
                                rd32(E1000_TDH(tx_ring->reg_idx)),
                                readl(tx_ring->tail),
                                tx_ring->next_to_use,
                                tx_ring->next_to_clean,
                                tx_buffer->time_stamp,
                                tx_buffer->next_to_watch,
                                jiffies,
                                tx_buffer->next_to_watch->wb.status);
                        netif_stop_subqueue(tx_ring->netdev,
                                            tx_ring->queue_index);

                        /* we are about to reset, no point in enabling stuff */
                        return true;
                }
```

The `if` statement above checks:

*   `tx_buffer->next_to_watch` is set, and
*   That the current `jiffies` is greater than the `time_stamp` recorded on the transmit path to the `tx_buffer` with a timeout factor added, and
*   The device’s transmit status register is not set to `E1000_STATUS_TXOFF`.

If those three tests are all true, then an error is printed that a hang has been detected. `netif_stop_subqueue` is used to turn off the queue and `true` is returned.

Let’s continue reading the code to see what happens if there was no transmit hang check, or if there was, but no hang was detected:

```c
#define TX_WAKE_THRESHOLD (DESC_NEEDED * 2)
        if (unlikely(total_packets &&
            netif_carrier_ok(tx_ring->netdev) &&
            igb_desc_unused(tx_ring) >= TX_WAKE_THRESHOLD)) {
                /* Make sure that anybody stopping the queue after this
                 * sees the new next_to_clean.
                 */
                smp_mb();
                if (__netif_subqueue_stopped(tx_ring->netdev,
                                             tx_ring->queue_index) &&
                    !(test_bit(__IGB_DOWN, &adapter->state))) {
                        netif_wake_subqueue(tx_ring->netdev,
                                            tx_ring->queue_index);

                        u64_stats_update_begin(&tx_ring->tx_syncp);
                        tx_ring->tx_stats.restart_queue++;
                        u64_stats_update_end(&tx_ring->tx_syncp);
                }
        }

        return !!budget;
```

In the above code the driver will restart the transmit queue if it was previously disabled. It first checks if:

*   Some packets were processed for completions (`total_packets` is non-zero), and
*   `netif_carrier_ok` to ensure the device has not been brought down, and
*   The number of unused descriptors in the transmit queue is greater than or equal to `TX_WAKE_THRESHOLD`. This threshold value appears to be `42` on my x86_64 system.

If all conditions are satisfied, a write barrier is used (`smp_mb`). Next another set of conditions are checked:

*   If the queue is stopped, and
*   The device is not down

Then `netif_wake_subqueue` called to wake up the transmit queue and signal to the higher layers that they may queue data again. The `restart_queue` statistics counter is incremented. We’ll see how to read this value next.

Finally, a boolean value is returned. If there was any remaining un-used budget `true` is returned, otherwise `false`. This value is checked in `igb_poll` to determine what to return back to `net_rx_action`.

#### `igb_poll` return value

The `igb_poll` function has this code to determine what to return to `net_rx_action`:

```c
        if (q_vector->tx.ring)
                clean_complete = igb_clean_tx_irq(q_vector);

        if (q_vector->rx.ring)
                clean_complete &= igb_clean_rx_irq(q_vector, budget);

        /* If all work not completed, return budget and keep polling */
        if (!clean_complete)
                return budget;
```

In other words, if:

*   `igb_clean_tx_irq` cleared all transmit completions without exhausting its transmit completion budget, and
*   `igb_clean_rx_irq` cleared all incoming packets without exhausting its packet processing budget

Then, the entire budget amount (which is hardcoded to `64` for most drivers including `igb`) will be returned. If either of RX or TX processing could not complete (because there was more work to do), then NAPI is disabled with a call to `napi_complete` and `0` is returned:

```c
        /* If not enough Rx work done, exit the polling mode */
        napi_complete(napi);
        igb_ring_irq_enable(q_vector);

        return 0;
}
```

### Monitoring network devices
There are several different ways to monitor your network devices offering different levels of granularity and complexity. Let’s start with most granular and move to least granular.

#### Using `ethtool -S`

You can install `ethtool` on an Ubuntu system by running: `sudo apt-get install ethtool`.

Once it is installed, you can access the statistics by passing the `-S` flag along with the name of the network device you want statistics about.

Monitor detailed NIC device statistics (e.g., transmit errors) with `ethtool -S`.
```sh
$ sudo ethtool -S eth0
NIC statistics:
     rx_packets: 597028087
     tx_packets: 5924278060
     rx_bytes: 112643393747
     tx_bytes: 990080156714
     rx_broadcast: 96
     tx_broadcast: 116
     rx_multicast: 20294528
     ....
```

Monitoring this data can be difficult. It is easy to obtain, but there is no standardization of the field values. Different drivers, or even different versions of the _same_ driver might produce different field names that have the same meaning.

You should look for values with “drop”, “buffer”, “miss”, “errors” etc in the label. Next, you will have to read your driver source. You’ll be able to determine which values are accounted for totally in software (e.g., incremented when there is no memory) and which values come directly from hardware via a register read. In the case of a register value, you should consult the data sheet for your hardware to determine what the meaning of the counter really is; many of the labels given via `ethtool` can be misleading.

#### Using sysfs

sysfs also provides a lot of statistics values, but they are slightly higher level than the direct NIC level stats provided.

You can find the number of dropped incoming network data frames for, e.g. eth0 by using `cat` on a file.

Monitor higher level NIC statistics with sysfs.

```sh
$ cat /sys/class/net/eth0/statistics/tx_aborted_errors
2
```

The counter values will be split into files like `tx_aborted_errors`, `tx_carrier_errors`, `tx_compressed`, `tx_dropped`, etc.

Unfortunately, it is up to the drivers to decide what the meaning of each field is, and thus, when to increment them and where the values come from. You may notice that some drivers count a certain type of error condition as a drop, but other drivers may count the same as a miss.

If these values are critical to you, you will need to read your driver source and device data sheet to understand exactly what your driver thinks each of these values means.

#### Using `/proc/net/dev`

An even higher level file is `/proc/net/dev` which provides high-level summary-esque information for each network adapter on the system.

Monitor high level NIC statistics by reading `/proc/net/dev`.
```sh
$ cat /proc/net/dev
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
  eth0: 110346752214 597737500    0    2    0     0          0  20963860 990024805984 6066582604    0    0    0     0       0          0
    lo: 428349463836 1579868535    0    0    0     0          0         0 428349463836 1579868535    0    0    0     0       0          0
```

This file shows a subset of the values you’ll find in the sysfs files mentioned above, but it may serve as a useful general reference.

The caveat mentioned above applies here, as well: if these values are important to you, you will still need to read your driver source to understand exactly when, where, and why they are incremented to ensure your understanding of an error, drop, or fifo are the same as your driver.

### Monitoring dynamic queue limits

You can monitor dynamic queue limits for a network device by reading the files located under: `/sys/class/net/NIC/queues/tx-QUEUE_NUMBER/byte_queue_limits/`.

Replacing `NIC` with your device name (`eth0`, `eth1`, etc) and `tx-QUEUE_NUMBER` with the transmit queue number (`tx-0`, `tx-1`, `tx-2`, etc).

Some of those files are:

*   `hold_time`: Initialized to `HZ` (a single hertz). If the queue has been full for `hold_time`, then the maximum size is decreased.
*   `inflight`: This value is equal to (number of packets queued - number of packets completed). It is the current number of packets being transmit for which a completion has not been processed.
*   `limit_max`: A hardcoded value, set to `DQL_MAX_LIMIT` (`1879048192` on my x86_64 system).
*   `limit_min`: A hardcoded value, set to `0`.
*   `limit`: A value between `limit_min` and `limit_max` which represents the current maximum number of objects which can be queued.

Before modifying any of these values, it is strongly recommended to [read these presentation slides](https://www.linuxplumbersconf.org/2012/wp-content/uploads/2012/08/bql_slide.pdf) for an in-depth explanation of the algorithm.

Monitor packet transmits in flight by reading `/sys/class/net/eth0/queues/tx-0/byte_queue_limits/inflight`.

```sh
$ cat /sys/class/net/eth0/queues/tx-0/byte_queue_limits/inflight
350
```

### Tuning network devices

#### Check the number of TX queues being used

If your NIC and the device driver loaded on your system support multiple transmit queues, you can usually adjust the number of TX queues (also called TX channels), by using `ethtool`.

Check the number of NIC transmit queues with `ethtool`

```sh
$ sudo ethtool -l eth0
Channel parameters for eth0:
Pre-set maximums:
RX:   0
TX:   0
Other:    0
Combined: 8
Current hardware settings:
RX:   0
TX:   0
Other:    0
Combined: 4
```

This output is displaying the pre-set maximums (enforced by the driver and the hardware) and the current settings.

**Note:** not all device drivers will have support for this operation.

Error seen if your NIC doesn't support this operation.

```sh
$ sudo ethtool -l eth0
Channel parameters for eth0:
Cannot get device channel parameters
: Operation not supported
```

This means that your driver has not implemented the ethtool `get_channels` operation. This could be because the NIC doesn’t support adjusting the number of queues, doesn’t support multiple transmit queues, or your driver has not been updated to handle this feature.

#### Adjust the number of TX queues used

Once you’ve found the current and maximum queue count, you can adjust the values by using `sudo ethtool -L`.

**Note:** some devices and their drivers only support combined queues that are paired for transmit and receive, as in the example in the above section.

Set combined NIC transmit and receive queues to 8 with `ethtool -L`

`$ sudo ethtool -L eth0 combined 8`

If your device and driver support individual settings for RX and TX and you’d like to change only the TX queue count to 8, you would run:

Set the number of NIC transmit queues to 8 with `ethtool -L`.

`$ sudo ethtool -L eth0 tx 8`

**Note:** making these changes will, for most drivers, take the interface down and then bring it back up; connections to this interface will be interrupted. This may not matter much for a one-time change, though.

#### Adjust the size of the TX queues
Some NICs and their drivers also support adjusting the size of the TX queue. Exactly how this works is hardware specific, but luckily `ethtool` provides a generic way for users to adjust the size. Increasing the size of the TX may not make a drastic difference because DQL is used to prevent higher layer networking code from queueing more data at times. Nevertheless, you may want to increase the TX queues to the maximum size and let DQL sort everything else out for you:

Check current NIC queue sizes with `ethtool -g`

```sh
$ sudo ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:   4096
RX Mini:  0
RX Jumbo: 0
TX:   4096
Current hardware settings:
RX:   512
RX Mini:  0
RX Jumbo: 0
TX:   512
```

the above output indicates that the hardware supports up to 4096 receive and transmit descriptors, but it is currently only using 512.

Increase size of each TX queue to 4096 with `ethtool -G`

`$ sudo ethtool -G eth0 tx 4096`

**Note:** making these changes will, for most drivers, take the interface down and then bring it back up; connections to this interface will be interrupted. This may not matter much for a one-time change, though.

## The End

The end! Now you know everything about how packet transmit works on Linux: from the user program to the device driver and back.

# Extras

There are a few extra things worth mentioning that are worth mentioning which didn’t seem quite right anywhere else.

## Reducing ARP traffic (`MSG_CONFIRM`)

The `send`, `sendto`, and and `sendmsg` system calls all take a `flags` parameter. If you pass the `MSG_CONFIRM` flag to these system calls from your application, it will cause the `dst_neigh_output` function in the kernel on the send path to update the timestamp of the neighbour structure. The consequence of this is that the neighbour structure will not be garbage collected. This prevents additional ARP traffic from being generated as the neighbour cache entry will stay warmer, longer.

## UDP Corking

We examined UDP corking extensively throughout the UDP protocol stack. If you want to use it in your application, you can enable UDP corking by calling `setsockopt` with level set to `IPPROTO_UDP`, optname set to `UDP_CORK`, and `optval` set to `1`.

## Timestamping

As mentioned in the above blog post, the networking stack can collect timestamps of outgoing data. See the above network stack walkthrough to see where transmit timestamping happens in software. Some NICs even support timestamping in hardware, too.

This is a useful feature if you’d like to try to determine how much latency the kernel network stack is adding to sending packets.

The [kernel documentation about timestamping](https://github.com/torvalds/linux/blob/v3.13/Documentation/networking/timestamping.txt) is excellent and there is even an included sample program and Makefile [you can check out!](https://github.com/torvalds/linux/tree/v3.13/Documentation/networking/timestamping).

Determine which timestamp modes your driver and device support with `ethtool -T`.

```sh
$ sudo ethtool -T eth0
Time stamping parameters for eth0:
Capabilities:
  software-transmit     (SOF_TIMESTAMPING_TX_SOFTWARE)
  software-receive      (SOF_TIMESTAMPING_RX_SOFTWARE)
  software-system-clock (SOF_TIMESTAMPING_SOFTWARE)
PTP Hardware Clock: none
Hardware Transmit Timestamp Modes: none
Hardware Receive Filter Modes: none
```

This NIC, unfortunately, does not support hardware transmit timestamping, but software timestamping can still be used on this system to help me determine how much latency the kernel is adding to my packet transmit path.

# Conclusion

The Linux networking stack is complicated.

As we saw above, even something as simple as the `NET_RX` can’t be guaranteed to work as we expect it to. Even though `RX` is in the name, transmit completions are still processed in this softIRQ.

This highlights what I believe to be the core of the issue: optimizing and monitoring the network stack is impossible unless you carefully read and understand how it works. You cannot monitor code you don’t understand at a deep level.

# Help with Linux networking or other systems

Need some extra help navigating the network stack? Have questions about anything in this post or related things not covered? Send us an [email](mailto:support@packagecloud.io) and let us know how we can help.

# Related posts
If you enjoyed this post, you may enjoy some of our other low-level technical posts:

*   [Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/)
*   [Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/)
*   [The Definitive Guide to Linux System Calls](https://blog.packagecloud.io/eng/2016/04/05/the-definitive-guide-to-linux-system-calls/)
*   [How does `strace` work?](https://blog.packagecloud.io/eng/2016/02/29/how-does-strace-work/)
*   [How does `ltrace` work?](https://blog.packagecloud.io/eng/2016/03/14/how-does-ltrace-work/)
*   [APT Hash sum mismatch](https://blog.packagecloud.io/eng/2016/03/21/apt-hash-sum-mismatch/)
*   [HOWTO: GPG sign and verify deb packages and APT repositories](https://blog.packagecloud.io/eng/2014/10/28/howto-gpg-sign-verify-deb-packages-apt-repositories/)
*   [HOWTO: GPG sign and verify RPM packages and yum repositories](https://blog.packagecloud.io/eng/2014/11/24/howto-gpg-sign-verify-rpm-packages-yum-repositories/)