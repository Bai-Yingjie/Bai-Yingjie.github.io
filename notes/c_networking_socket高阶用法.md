- [Advanced Socket Topics](#advanced-socket-topics)
  - [Out-of-Band Data](#out-of-band-data)
        - [Example 8–10 Flushing Terminal I/O on Receipt of Out-of-Band Data](#example-810-flushing-terminal-io-on-receipt-of-out-of-band-data)
  - [Nonblocking Sockets](#nonblocking-sockets)
        - [Example 8–11 Set Nonblocking Socket](#example-811-set-nonblocking-socket)
  - [Asynchronous Socket I/O](#asynchronous-socket-io)
        - [Example 8–12 Making a Socket Asynchronous](#example-812-making-a-socket-asynchronous)
  - [Interrupt-Driven Socket I/O](#interrupt-driven-socket-io)
        - [Example 8–13 Asynchronous Notification of I/O Requests](#example-813-asynchronous-notification-of-io-requests)
  - [Signals and Process Group ID](#signals-and-process-group-id)
  - [Selecting Specific Protocols](#selecting-specific-protocols)
  - [Address Binding](#address-binding)
  - [Socket Options](#socket-options)
  - [<kbd>inetd</kbd> Daemon](#inetd-daemon)
  - [Broadcasting and Determining Network Configuration](#broadcasting-and-determining-network-configuration)
        - [Example 8–14 <tt>net/if.h</tt> Header File](#example-814-ttnetifhtt-header-file)
        - [Example 8–15 Obtaining Interface Flags](#example-815-obtaining-interface-flags)
        - [Example 8–16 Broadcast Address of an Interface](#example-816-broadcast-address-of-an-interface)
- [参考](#参考)

# Advanced Socket Topics

For most programmers, the mechanisms already described are enough to build distributed applications. This section describes additional features.

## Out-of-Band Data

The stream socket abstraction includes out-of-band data. Out-of-band data is a logically independent transmission channel between a pair of connected stream sockets. Out-of-band data is delivered independent of normal data. The out-of-band data facilities must support the reliable delivery of at least one out-of-band message at a time. This message can contain at least one byte of data. At least one message can be pending delivery at any time.

With in-band signaling, urgent data is delivered in sequence with normal data, and the message is extracted from the normal data stream. The extracted message is stored separately. Users can choose between receiving the urgent data in order and receiving the data out of sequence, without having to buffer the intervening data.

Using <tt>MSG_PEEK</tt>, you can peek at out-of-band data. If the socket has a process group, a <tt>SIGURG</tt> signal is generated when the protocol is notified of its existence. A process can set the process group or process ID to deliver <tt>SIGURG</tt> to with the appropriate [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html) call, as described in [Interrupt-Driven Socket I/O](https://docs.oracle.com/cd/E19120-01/open.solaris/817-4415/sockets-82255/index.html) for <tt>SIGIO</tt>. If multiple sockets have out-of-band data waiting for delivery, a [select(3C)](https://docs.oracle.com/docs/cd/E19253-01/816-5168/select-3c/index.html) call for exceptional conditions can determine which sockets have such data pending.

A logical mark is placed in the data stream at the point at which the out-of-band data was sent. The remote login and remote shell applications use this facility to propagate signals between client and server processes. When a signal is received, all data up to the mark in the data stream is discarded.

To send an out-of-band message, apply the <tt>MSG_OOB</tt> flag to [send(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/send-3socket/index.html) or [sendto(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/sendto-3socket/index.html). To receive out-of-band data, specify <tt>MSG_OOB</tt> to [recvfrom(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/recvfrom-3socket/index.html) or [recv(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/recv-3socket/index.html). If out-of-band data is taken in line the <tt>MSG_OOB</tt> flag is not needed. The <tt>SIOCATMARK</tt> [ioctl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/ioctl-2/index.html) indicates whether the read pointer currently points at the mark in the data stream:
```c
int yes;
ioctl(s, SIOCATMARK, &yes);
```

If <var>yes</var> is <tt>1</tt> on return, the next read returns data after the mark. Otherwise, assuming out-of-band data has arrived, the next read provides data sent by the client before sending the out-of-band signal. The routine in the remote login process that flushes output on receipt of an interrupt or quit signal is shown in the following example. This code reads the normal data up to the mark to discard the normal data, then reads the out-of-band byte.

A process can also read or peek at the out-of-band data without first reading up to the mark. Accessing this data when the underlying protocol delivers the urgent data in-band with the normal data, and sends notification of its presence only ahead of time, is more difficult. An example of this type of protocol is TCP, the protocol used to provide socket streams in the Internet family. With such protocols, the out-of-band byte might not yet have arrived when [recv(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/recv-3socket/index.html) is called with the <tt>MSG_OOB</tt> flag. In that case, the call returns the error of <tt>EWOULDBLOCK</tt>. Also, the amount of in-band data in the input buffer might cause normal flow control to prevent the peer from sending the urgent data until the buffer is cleared. The process must then read enough of the queued data to clear the input buffer before the peer can send the urgent data.


* * *

##### Example 8–10 Flushing Terminal I/O on Receipt of Out-of-Band Data

```c
#include <sys/ioctl.h>
#include <sys/file.h>
...
oob()
{
    int out = FWRITE;
    char waste[BUFSIZ];
    int mark = 0;

    /* flush local terminal output */
    ioctl(1, TIOCFLUSH, (char *) &out);
    while(1) {
        if (ioctl(rem, SIOCATMARK, &mark) == -1) {
            perror("ioctl");
            break;
        }
        if (mark)
            break;
        (void) read(rem, waste, sizeof waste);
    }
    if (recv(rem, &mark, 1, MSG_OOB) == -1) {
        perror("recv");
        ...
    }
    ...
}
```

* * *

A facility to retain the position of urgent in-line data in the socket stream is available as a socket-level option, <tt>SO_OOBINLINE</tt>. See [getsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getsockopt-3socket/index.html) for usage. With this socket-level option, the position of urgent data remains. However, the urgent data immediately following the mark in the normal data stream is returned without the <tt>MSG_OOB</tt> flag. Reception of multiple urgent indications moves the mark, but does not lose any out-of-band data.

## Nonblocking Sockets

Some applications require sockets that do not block. For example, a server would return an error code, not executing a request that cannot complete immediately. This error could cause the process to be suspended, awaiting completion. After creating and connecting a socket, issuing a [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html) call, as shown in the following example, makes the socket nonblocking.

* * *

##### Example 8–11 Set Nonblocking Socket

```c
#include <fcntl.h>
#include <sys/file.h>
...
int fileflags;
int s;
...
s = socket(AF_INET6, SOCK_STREAM, 0);
...
if (fileflags = fcntl(s, F_GETFL, 0) == -1)
    perror("fcntl F_GETFL");
    exit(1);
}
if (fcntl(s, F_SETFL, fileflags | FNDELAY) == -1)
    perror("fcntl F_SETFL, FNDELAY");
    exit(1);
}
```

* * *

When performing I/O on a nonblocking socket, check for the error <samp>EWOULDBLOCK</samp> in <tt>errno.h</tt>, which occurs when an operation would normally block. [accept(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/accept-3socket/index.html), [connect(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/connect-3socket/index.html), [send(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/send-3socket/index.html), [recv(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/recv-3socket/index.html), [read(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/read-2/index.html), and [write(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/write-2/index.html) can all return <tt>EWOULDBLOCK</tt>. If an operation such as a [send(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/send-3socket/index.html) cannot be done in its entirety but partial writes work, as when using a stream socket, all available data is processed. The return value is the amount of data actually sent.

## Asynchronous Socket I/O

Asynchronous communication between processes is required in applications that simultaneously handle multiple requests. Asynchronous sockets must be of the <tt>SOCK_STREAM</tt> type. To make a socket asynchronous, you issue a [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html) call, as shown in the following example.

* * *

##### Example 8–12 Making a Socket Asynchronous
```c
#include <fcntl.h>
#include <sys/file.h>
...
int fileflags;
int s;
...
s = socket(AF_INET6, SOCK_STREAM, 0);
...
if (fileflags = fcntl(s, F_GETFL ) == -1)
    perror("fcntl F_GETFL");
    exit(1);
}
if (fcntl(s, F_SETFL, fileflags | FNDELAY | FASYNC) == -1)
    perror("fcntl F_SETFL, FNDELAY | FASYNC");
    exit(1);
}
```

* * *

After sockets are initialized, connected, and made nonblocking and asynchronous, communication is similar to reading and writing a file asynchronously. Initiate a data transfer by using [send(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/send-3socket/index.html), [write(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/write-2/index.html), [recv(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/recv-3socket/index.html), or [read(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/read-2/index.html). A signal-driven I/O routine completes a data transfer, as described in the next section.

## Interrupt-Driven Socket I/O

1.  Set up a <samp>SIGIO</samp> signal handler with the [signal(3C)](https://docs.oracle.com/docs/cd/E19253-01/816-5168/signal-3c/index.html) or [sigvec(3UCB)](https://docs.oracle.com/docs/cd/E19253-01/816-5168/sigvec-3ucb/index.html) calls.

2.  Use [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html) to set the process ID or process group ID to route the signal to its own process ID or process group ID. The default process group of a socket is group <tt>0</tt>.

3.  Convert the socket to asynchronous, as shown in [Asynchronous Socket I/O](https://docs.oracle.com/cd/E19120-01/open.solaris/817-4415/sockets-40/index.html).

The following sample code enables receipt of information on pending requests as the requests occur for a socket by a given process. With the addition of a handler for <tt>SIGURG</tt>, this code can also be used to prepare for receipt of <tt>SIGURG</tt> signals.

* * *

##### Example 8–13 Asynchronous Notification of I/O Requests

```c
#include <fcntl.h>
#include <sys/file.h>
 ...
signal(SIGIO, io_handler);
/* Set the process receiving SIGIO/SIGURG signals to us. */
if (fcntl(s, F_SETOWN, getpid()) < 0) {
    perror("fcntl F_SETOWN");
    exit(1);
}

```
* * *

## Signals and Process Group ID

For <tt>SIGURG</tt> and <tt>SIGIO</tt>, each socket has a process number and a process group ID. These values are initialized to zero, but can be redefined at a later time with the <tt>F_SETOWN</tt> [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html) command, as in the previous example. A positive third argument to [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html) sets the socket's process ID. A negative third argument to [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html) sets the socket's process group ID. The only allowed recipient of <tt>SIGURG</tt> and <tt>SIGIO</tt> signals is the calling process. A similar [fcntl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fcntl-2/index.html), <tt>F_GETOWN</tt>, returns the process number of a socket.

You can also enable reception of <samp>SIGURG</samp> and <samp>SIGIO</samp> by using [ioctl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/ioctl-2/index.html) to assign the socket to the user's process group.

```c
/* oobdata is the out-of-band data handling routine */
sigset(SIGURG, oobdata);
int pid = -getpid();
if (ioctl(client, SIOCSPGRP, (char *) &pid) < 0) {
    perror("ioctl: SIOCSPGRP");
}
```

## Selecting Specific Protocols

If the third argument of the [socket(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/socket-3socket/index.html) call is <tt>0</tt>, [socket(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/socket-3socket/index.html) selects a default protocol to use with the returned socket of the type requested. The default protocol is usually correct, and alternate choices are not usually available. When using raw sockets to communicate directly with lower-level protocols or lower-level hardware interfaces, set up de-multiplexing with the protocol argument.

Using raw sockets in the Internet family to implement a new protocol on IP ensures that the socket only receives packets for the specified protocol. To obtain a particular protocol, determine the protocol number as defined in the protocol family. For the Internet family, use one of the library routines that are discussed in [Standard Routines](https://docs.oracle.com/cd/E19120-01/open.solaris/817-4415/sockets-86445/index.html), such as [getprotobyname(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getprotobyname-3socket/index.html).

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netdb.h>
...
pp = getprotobyname("newtcp");
s = socket(AF_INET6, SOCK_STREAM, pp->p_proto);
```

Using <kbd>getprotobyname</kbd> results in a socket <kbd>**s**</kbd> by using a stream-based connection, but with a protocol type of <tt>newtcp</tt> instead of the default <tt>tcp</tt>.

## Address Binding
For addressing, TCP and UDP use a 4-tuple of:

*   Local IP address
*   Local port number
*   Foreign IP address
*   Foreign port number

TCP requires these 4-tuples to be unique. UDP does not. User programs do not always know proper values to use for the local address and local port, because a host can reside on multiple networks. The set of allocated port numbers is not directly accessible to a user. To avoid these problems, leave parts of the address unspecified and let the system assign the parts appropriately when needed. Various portions of these tuples can be specified by various parts of the sockets API:

[bind(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/bind-3socket/index.html)

Local address or local port or both

[connect(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/connect-3socket/index.html)

Foreign address and foreign port

A call to [accept(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/accept-3socket/index.html) retrieves connection information from a foreign client. This causes the local address and port to be specified to the system even though the caller of [accept(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/accept-3socket/index.html) did not specify anything. The foreign address and foreign port are returned.

A call to [listen(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/listen-3socket/index.html) can cause a local port to be chosen. If no explicit [bind(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/bind-3socket/index.html) has been done to assign local information, [listen(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/listen-3socket/index.html) assigns an ephemeral port number.

A service that resides at a particular port can [bind(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/bind-3socket/index.html) to that port. Such a service can leave the local address unspecified if the service does not require local address information. The local address is set to <tt>in6addr_any</tt>, a variable with a constant value in <tt><netinet/in.h></tt>. If the local port does not need to be fixed, a call to [listen(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/listen-3socket/index.html) causes a port to be chosen. Specifying an address of <tt>in6addr_any</tt> or a port number of 0 is known as _wildcarding_. For <tt>AF_INET</tt>, <tt>INADDR_ANY</tt> is used in place of <tt>in6addr_any</tt>.

The wildcard address simplifies local address binding in the Internet family. The following sample code binds a specific port number that was returned by a call to [getaddrinfo(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getaddrinfo-3socket/index.html) to a socket and leaves the local address unspecified:

```c
#include <sys/types.h>
#include <netinet/in.h>
...
struct addrinfo  *aip;
...
if (bind(sock, aip->ai_addr, aip->ai_addrlen) == -1) {
    perror("bind");
    (void) close(sock);
    return (-1);
}
```

Each network interface on a host typically has a unique IP address. Sockets with wildcard local addresses can receive messages that are directed to the specified port number. Messages that are sent to any of the possible addresses that are assigned to a host are also received by sockets with wildcard local addresses. To allow only hosts on a specific network to connect to the server, a server binds the address of the interface on the appropriate network.

Similarly, a local port number can be left unspecified, in which case the system selects a port number. For example, to bind a specific local address to a socket, but to leave the local port number unspecified, you could use <kbd>bind</kbd> as follows:

```c
bzero (&sin, sizeof (sin));
(void) inet_pton (AF_INET6, "::ffff:127.0.0.1", sin.sin6_addr.s6_addr);
sin.sin6_family = AF_INET6;
sin.sin6_port = htons(0);
bind(s, (struct sockaddr *) &sin, sizeof sin);
```

The system uses two criteria to select the local port number:

*   Internet port numbers less than 1024 (<tt>IPPORT_RESERVED</tt>) are reserved for privileged users. Nonprivileged users can use any Internet port number that is greater than 1024\. The largest Internet port number is 65535.

*   The port number is not currently bound to some other socket.

The port number and IP address of the client are found through either [accept(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/accept-3socket/index.html) or [getpeername(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getpeername-3socket/index.html).

In certain cases, the algorithm used by the system to select port numbers is unsuitable for an application due to the two-step creation process for associations. For example, the Internet file transfer protocol specifies that data connections must always originate from the same local port. However, duplicate associations are avoided by connecting to different foreign ports. In this situation, the system would disallow binding the same local address and local port number to a socket if a previous data connection's socket still existed.

To override the default port selection algorithm, you must perform an option call before address binding:

```c
int on = 1;
...
setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof on);
bind(s, (struct sockaddr *) &sin, sizeof sin);
```

With this call, local addresses already in use can be bound. This binding does not violate the uniqueness requirement. The system still verifies at connect time that any other sockets with the same local address and local port do not have the same foreign address and foreign port. If the association already exists, the error <samp>EADDRINUSE</samp> is returned.

## Socket Options

You can set and get several options on sockets through [setsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/setsockopt-3socket/index.html) and [getsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getsockopt-3socket/index.html). For example, you can change the send or receive buffer space. The general forms of the calls are in the following list:

`setsockopt(s, level, optname, optval, optlen);`

and

`getsockopt(s, level, optname, optval, optlen);`

The operating system can adjust the values appropriately at any time.

The arguments of [setsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/setsockopt-3socket/index.html) and [getsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getsockopt-3socket/index.html) calls are in the following list:

s: Socket on which the option is to be applied

level: Specifies the protocol level, such as socket level, indicated by the symbolic constant <tt>SOL_SOCKET</tt> in <tt>sys/socket.h</tt>

optname: Symbolic constant defined in <tt>sys/socket.h</tt> that specifies the option

optval: Points to the value of the option

optlen: Points to the length of the value of the option


For [getsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getsockopt-3socket/index.html), <var>optlen</var> is a value-result argument. This argument is initially set to the size of the storage area pointed to by <var>optval</var>. On return, the argument's value is set to the length of storage used.

When a program needs to determine an existing socket's type, the program should invoke [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) by using the <tt>SO_TYPE</tt> socket option and the [getsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getsockopt-3socket/index.html) call:

```c
#include <sys/types.h>
#include <sys/socket.h>

int type, size;

size = sizeof (int);
if (getsockopt(s, SOL_SOCKET, SO_TYPE, (char *) &type, &size) < 0) {
    ...
}
```

After [getsockopt(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getsockopt-3socket/index.html), <tt>type</tt> is set to the value of the socket type, as defined in <tt>sys/socket.h</tt>. For a datagram socket, <tt>type</tt> would be <tt>SOCK_DGRAM</tt>.

## <kbd>inetd</kbd> Daemon

The [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) daemon is invoked at startup time and gets the services for which the daemon listens from the <tt>/etc/inet/inetd.conf</tt> file. The daemon creates one socket for each service that is listed in <tt>/etc/inet/inetd.conf</tt>, binding the appropriate port number to each socket. See the [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) man page for details.

The [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) daemon polls each socket, waiting for a connection request to the service corresponding to that socket. For <tt>SOCK_STREAM</tt> type sockets, [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) accepts ([accept(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/accept-3socket/index.html)) on the listening socket, forks ([fork(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/fork-2/index.html)), duplicates ([dup(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/dup-2/index.html)) the new socket to file descriptors <tt>0</tt> and <tt>1</tt> (<tt>stdin</tt> and <tt>stdout</tt>), closes other open file descriptors, and executes ([exec(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/exec-2/index.html)) the appropriate server.

The primary benefit of using [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) is that services not in use do not consume machine resources. A secondary benefit is that [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) does most of the work to establish a connection. The server started by [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) has the socket connected to its client on file descriptors <tt>0</tt> and <tt>1</tt>. The server can immediately read, write, send, or receive. Servers can use buffered I/O as provided by the <tt>stdio</tt> conventions, as long as the servers use [fflush(3C)](https://docs.oracle.com/docs/cd/E19253-01/816-5168/fflush-3c/index.html) when appropriate.

The [getpeername(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/getpeername-3socket/index.html) routine returns the address of the peer (process) connected to a socket. This routine is useful in servers started by [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html). For example, you could use this routine to log the Internet address such as <tt>fec0::56:a00:20ff:fe7d:3dd2</tt>, which is conventional for representing the IPv6 address of a client. An [inetd(1M)](https://docs.oracle.com/docs/cd/E19253-01/816-5166/inetd-1m/index.html) server could use the following sample code:

```c
struct sockaddr_storage name;
int namelen = sizeof (name);
char abuf[INET6_ADDRSTRLEN];
struct in6_addr addr6;
struct in_addr addr;

if (getpeername(fd, (struct sockaddr *) &name, &namelen) == -1) {
    perror("getpeername");
    exit(1);
} else {
    addr = ((struct sockaddr_in *)&name)->sin_addr;
    addr6 = ((struct sockaddr_in6 *)&name)->sin6_addr;
    if (name.ss_family == AF_INET) {
            (void) inet_ntop(AF_INET, &addr, abuf, sizeof (abuf));
    } else if (name.ss_family == AF_INET6 &&
               IN6_IS_ADDR_V4MAPPED(&addr6)) {
            /* this is a IPv4-mapped IPv6 address */
            IN6_MAPPED_TO_IN(&addr6, &addr);
            (void) inet_ntop(AF_INET, &addr, abuf, sizeof (abuf));
    } else if (name.ss_family == AF_INET6) {
            (void) inet_ntop(AF_INET6, &addr6, abuf, sizeof (abuf));

    }
    syslog("Connection from %s\n", abuf);
}
```

## Broadcasting and Determining Network Configuration

Broadcasting is not supported in IPv6\. Broadcasting is supported only in IPv4.

Messages sent by datagram sockets can be broadcast to reach all of the hosts on an attached network. The network must support broadcast because the system provides no simulation of broadcast in software. Broadcast messages can place a high load on a network because broadcast messages force every host on the network to service the broadcast messages. Broadcasting is usually used for either of two reasons:

*   To find a resource on a local network without having its address

*   For functions that require information to be sent to all accessible neighbors

To send a broadcast message, create an Internet datagram socket:

`s = socket(AF_INET, SOCK_DGRAM, 0);`

Bind a port number to the socket:

```c
sin.sin_family = AF_INET;
sin.sin_addr.s_addr = htonl(INADDR_ANY);
sin.sin_port = htons(MYPORT);
bind(s, (struct sockaddr *) &sin, sizeof sin);
```

The datagram can be broadcast on only one network by sending to the network's broadcast address. A datagram can also be broadcast on all attached networks by sending to the special address <tt>INADDR_BROADCAST</tt>, which is defined in <tt>netinet/in.h</tt>.

The system provides a mechanism to determine a number of pieces of information about the network interfaces on the system. This information includes the IP address and broadcast address. The <tt>SIOCGIFCONF</tt> [ioctl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/ioctl-2/index.html) call returns the interface configuration of a host in a single <tt>ifconf</tt> structure. This structure contains an array of <tt>ifreq</tt> structures. Every address family supported by every network interface to which the host is connected has its own <tt>ifreq</tt> structure.

The following example shows the <kbd>ifreq</kbd> structures defined in <tt>net/if.h</tt>.


* * *

##### Example 8–14 <tt>net/if.h</tt> Header File

```c
struct ifreq {
    #define IFNAMSIZ 16
    char ifr_name[IFNAMSIZ]; /* if name, e.g., "en0" */
    union {
        struct sockaddr ifru_addr;
        struct sockaddr ifru_dstaddr;
        char ifru_oname[IFNAMSIZ]; /* other if name */
        struct sockaddr ifru_broadaddr;
        short ifru_flags;
        int ifru_metric;
        char ifru_data[1]; /* interface dependent data */
        char ifru_enaddr[6];
    } ifr_ifru;
    #define ifr_addr ifr_ifru.ifru_addr
    #define ifr_dstaddr ifr_ifru.ifru_dstaddr
    #define ifr_oname ifr_ifru.ifru_oname
    #define ifr_broadaddr ifr_ifru.ifru_broadaddr
    #define ifr_flags ifr_ifru.ifru_flags
    #define ifr_metric ifr_ifru.ifru_metric
    #define ifr_data ifr_ifru.ifru_data
    #define ifr_enaddr ifr_ifru.ifru_enaddr
};
```

* * *

The call that obtains the interface configuration is:

```c
/*
 * Do SIOCGIFNUM ioctl to find the number of interfaces
 *
 * Allocate space for number of interfaces found
 *
 * Do SIOCGIFCONF with allocated buffer
 *
 */
if (ioctl(s, SIOCGIFNUM, (char *)&numifs) == -1) {
    numifs = MAXIFS;
}
bufsize = numifs * sizeof(struct ifreq);
reqbuf = (struct ifreq *)malloc(bufsize);
if (reqbuf == NULL) {
    fprintf(stderr, "out of memory\n");
    exit(1);
}
ifc.ifc_buf = (caddr_t)&reqbuf[0];
ifc.ifc_len = bufsize;
if (ioctl(s, SIOCGIFCONF, (char *)&ifc) == -1) {
    perror("ioctl(SIOCGIFCONF)");
    exit(1);
}
```

After this call, <var>buf</var> contains an array of <tt>ifreq</tt> structures. Every network to which the host connects has an associated <tt>ifreq</tt> structure. The sort order these structures appear in is:

*   Alphabetical by interface name

*   Numerical by supported address families

The value of <tt>ifc.ifc_len</tt> is set to the number of bytes used by the <tt>ifreq</tt> structures.

Each structure has a set of interface flags that indicate whether the corresponding network is up or down, point-to-point or broadcast, and so on. The following example shows [ioctl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/ioctl-2/index.html) returning the <tt>SIOCGIFFLAGS</tt> flags for an interface specified by an <kbd>ifreq</kbd> structure.

* * *

##### Example 8–15 Obtaining Interface Flags

```c
struct ifreq *ifr;
ifr = ifc.ifc_req;
for (n = ifc.ifc_len/sizeof (struct ifreq); --n >= 0; ifr++) {
    /*
     * Be careful not to use an interface devoted to an address
     * family other than those intended.
     */
    if (ifr->ifr_addr.sa_family != AF_INET)
        continue;
    if (ioctl(s, SIOCGIFFLAGS, (char *) ifr) < 0) {
        ...
    }
    /* Skip boring cases */
    if ((ifr->ifr_flags & IFF_UP) == 0 ||
            (ifr->ifr_flags & IFF_LOOPBACK) ||
            (ifr->ifr_flags & (IFF_BROADCAST | IFF_POINTOPOINT)) == 0)
        continue;
}
```

* * *

The following example uses the <tt>SIOGGIFBRDADDR</tt> [ioctl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/ioctl-2/index.html) command to obtain the broadcast address of an interface.

* * *

##### Example 8–16 Broadcast Address of an Interface
```c
if (ioctl(s, SIOCGIFBRDADDR, (char *) ifr) < 0) {
    ...
}
memcpy((char *) &dst, (char *) &ifr->ifr_broadaddr,
    sizeof ifr->ifr_broadaddr);
```

* * *

You can also use <tt>SIOGGIFBRDADDR</tt> [ioctl(2)](https://docs.oracle.com/docs/cd/E19253-01/816-5167/ioctl-2/index.html) to get the destination address of a point-to-point interface.

After the interface broadcast address is obtained, transmit the broadcast datagram with [sendto(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/sendto-3socket/index.html):


`sendto(s, buf, buflen, 0, (struct sockaddr *)&dst, sizeof dst)`

Use one [sendto(3SOCKET)](https://docs.oracle.com/docs/cd/E19253-01/816-5170/sendto-3socket/index.html) for each interface to which the host is connected, if that interface supports the broadcast or point-to-point addressing.

# 参考
鸿篇巨制: [Chapter 8 Socket Interfaces](https://docs.oracle.com/cd/E19120-01/open.solaris/817-4415/6mjum5soe/index.html)
[总目录](https://docs.oracle.com/cd/E19120-01/open.solaris/817-4415/index.html)