- [Introduction](#introduction)
  - [Preamble](#preamble)
  - [How to Use This Document and Intended Audience](#how-to-use-this-document-and-intended-audience)
  - [Requirements to Use This Document](#requirements-to-use-this-document)
  - [Obtain Example Source Code](#obtain-example-source-code)
  - [Getting More Information](#getting-more-information)
  - [Disclaimer](#disclaimer)
  - [Copyright](#copyright)
  - [Authors](#authors)
  - [Most recent version](#most-recent-version)
- [Procfs, Sysfs, and Similar Kernel Interfaces](#procfs-sysfs-and-similar-kernel-interfaces)
  - [Introduction](#introduction-1)
  - [Procfs](#procfs)
    - [Description](#description)
    - [Implementation](#implementation)
      - [1\. Legacy procfs API](#1-legacy-procfs-api)
      - [2\. Seq_file API](#2-seq_file-api)
    - [Further Reading and Resources](#further-reading-and-resources)
  - [Sysfs](#sysfs)
    - [Description](#description-1)
    - [Implementation](#implementation-1)
      - [Module Parameter API](#module-parameter-api)
      - [Standard Sysfs API](#standard-sysfs-api)
  - [Configfs](#configfs)
    - [Description](#description-2)
    - [Resources and Further Reading](#resources-and-further-reading)
  - [Debugfs](#debugfs)
    - [Description](#description-3)
    - [Implementation](#implementation-2)
  - [Sysctl](#sysctl)
    - [Description](#description-4)
    - [Implementation](#implementation-3)
    - [Resources and Further Reading](#resources-and-further-reading-1)
  - [Character Devices](#character-devices)
    - [Description](#description-5)
    - [Implementation](#implementation-4)
    - [Resources and Further Reading](#resources-and-further-reading-2)
- [Socket Based Mechanisms](#socket-based-mechanisms)
  - [Introduction](#introduction-2)
  - [UDP Sockets](#udp-sockets)
    - [Description](#description-6)
    - [Implementation](#implementation-5)
  - [Netlink Sockets](#netlink-sockets)
    - [Description](#description-7)
  - [Implementation](#implementation-6)
    - [User Space](#user-space)
    - [Kernel Module](#kernel-module)
  - [Resources and Further Reading](#resources-and-further-reading-3)
- [Ioctl](#ioctl)
  - [Introduction](#introduction-3)
  - [Implementation](#implementation-7)
  - [Resources and Further Reading](#resources-and-further-reading-4)
- [Kernel System Calls](#kernel-system-calls)
  - [Introduction](#introduction-4)
  - [Implementation](#implementation-8)
  - [Resources and Further Reading](#resources-and-further-reading-5)
- [Sending Signals from the Kernel to the User Space](#sending-signals-from-the-kernel-to-the-user-space)
  - [Introduction](#introduction-5)
  - [Implementation](#implementation-9)
  - [Resources and Further Reading](#resources-and-further-reading-6)
- [Upcall](#upcall)
  - [Introduction](#introduction-6)
  - [Implementation](#implementation-10)
  - [Resources and Further Reading](#resources-and-further-reading-7)
- [Mmap](#mmap)
  - [Introduction](#introduction-7)
  - [Implementation](#implementation-11)
    - [my_open](#my_open)
    - [my_close](#my_close)
    - [my_mmap](#my_mmap)
  - [Resources and Further Reading](#resources-and-further-reading-8)

**Kernel Space, User Space Interfaces**

This document looks at the numerous and interesting ways the Linux kernel 2.6 interacts with user space programs. We explain sockets, procfs (and similar virtual filesystems), creating new Linux system calls, as well as mundane file and memory handling.

**Revision History**

First wiki revision- 2008-09-28 Wiki created: Ariane Keller
Revision 0.1 - 2008-10-04 Chapter one and two: Ariane Keller
Revision 0.2 - 2008-10-04 Added all chapters, needs format reviewing: Ariane Keller

* * * * *

Introduction
============

Preamble
--------

This how-to aims to provide an overview over all existing communication mechanisms between Linux user and kernel space. The goal is to present a starting point for a developer who needs to transfer data between Linux kernel modules and user space programs. Each mechanism is described in its own section, with each section further divided into Description, Implementation, and Resources & Further Reading. In the description section we describe the basic mechanism and intended use of the mechanism. The implementation provides an example source code along with a short description. The Resources & Further reading section provides a list of useful articles and book chapters.

All source code is tested on Linux kernel 2.6.23\. Therefore it may not run on other (earlier or more recent) kernels. However, I try to keep it up to date.

If you find a bug or if you know a communication mechanism that is not covered in this how-to please send an email to the author or update this wiki yourself.

**Be warned:** playing with your kernel may cause it to crash, necessitating a system reboot! This means: save all your files and close all programs before you insert a kernel module! You may consider to run experimental kernels in a virtual machine so your actual host system does not get affected by any crashes.

How to Use This Document and Intended Audience
----------------------------------------------

This document is written for programmers with some experience in system programming. The focus is on a short explanation of individual mechanisms, and providing examples for them. It is assumed that the reader is able to understand the source code (with the documentation provided) and that he can use the examples as a basis for its own modules. It is assumed that a programmer reads first the description section, then has a look at the source code and finally compares the source code with the explanation in the implementation section.

The examples are deliberately kept simple, and real life modules will be much more complex.

Requirements to Use This Document
---------------------------------

-   Linux Distribution with all tools needed to build a new Linux kernel. The example code is tested on 2.6.23.
-   Root privileges (in order to execute the example modules)
-   Knowledge of C programming
-   Basic knowledge of operating system concepts

Obtain Example Source Code
--------------------------

Throughout this document we show a lot of example source code. This code can be downloaded as a [tar.gz](http://people.ee.ethz.ch/~arkeller/linux/code.tar.gz).

probably better upload the code in the LDP wiki like this: [test.tgz](https://wiki.tldp.org/static/kernel_user_space_howto.html "Upload new attachment "test.tgz"")

Getting More Information
------------------------

Each section contains a list to articles with further information. Personally, I like the following resources:

-   There are some excellent books from O'Reilly
    -   Linux Device Driver, 3rd edition, Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman, February 2005
    -   Understanding the Linux Kernel, 3rd edition, By Daniel P. Bovet, Marco Cesati, November 2005
    -   Understanding the Linux Network Internals, 1rd edition, By Christian Benvenuti, December 2005
-   The directory Documentation in the Linux kernel source code
-   And of course the Linux source code itself. An excellent web page for browsing the source code is <http://lxr.linux.no/linux/Documentation/>. It offers the possibility to search for keywords such as function names, defines, variables etc. It shows all the files that contain this data and provides links to the source code where the variable is used.

Disclaimer
----------

Use the information in this document at your own risk. I disavow any potential liability for the contents of this document. Use of the concepts, examples, and/or other content of this document is entirely at your own risk. Inserting a kernel module may cause your computer to be unresponsive and you may need to perform a "hard-reset". Therefore save any information you want to keep before you insert a kernel module.

All copyrights are owned by their owners, unless specifically noted otherwise. Use of a term in this document should not be regarded as affecting the validity of any trademark or service mark.

Naming of particular products or brands should not be seen as endorsements.

You are strongly recommended to take a backup of your system before major installation and backups at regular intervals.

Copyright
---------

Copyright © 2008 Ariane Keller.
Permission is granted to copy, distribute and/or modify this document under the terms of the GNU Free Documentation License, Version 1.1 or any later version published by the Free Software Foundation with no Invariant Sections, no Front-Cover Texts, and no Back-Cover Texts.

[GNU Free Documentation License](http://wiki.tldp.org/LdpWikiDefaultLicence#GNUFreeDocumentationLicense)

Authors
-------

This document was written by Ariane Keller. To contact me you can search with google for my email address. The document was reviewed by Steven to improve its readability.

Most recent version
-------------------

The most recent version of this mini-HOWTO will be found on <http://wiki.tldp.org/kernel_user_space_howto>

* * * * *

Procfs, Sysfs, and Similar Kernel Interfaces
============================================

Introduction
------------

These file systems are optional to the Linux kernel, and may not be enabled on your system. The file `/lib/modules/`uname -r`/build/.config` will tell you how your kernel is configured.

In order to exchange data between user space and kernel space the Linux kernel provides a couple of RAM based file systems. These interfaces are, themselves, based on files. Usually a file represents a single value, but it may also represent a set of values. The user space can access these values by means of the standard read(2) and write(2) functions. For most file systems the read and write function results in a callback function in the Linux kernel which has access to the corresponding value.

Despite offering similar functionality, the different RAM based file systems are all designed for separate purposes. However it is easy to use these file systems for other purposes as well. Questions such as "Which file system should be used?" or "Why is there a need for the different file systems?" often arise on the Linux kernel mailing list. The arguments are controversial and each developer seems to have a unique view.

The benefit of using the read and write function in comparison to, for example, socket based approaches, is that the user space has a lot of tools available to send data to the kernel space (e.g. cat(1), echo (1)). These programs are well known to users and they can be used in scripts.

Procfs
------

### Description

The procfs, located in /proc, is the best known interface of this class. It was originally designed to export all kind of process information such as the current status of the process, or all open file descriptors to the user space. Despite its initial purposes, the procfs has been used for a lot of other purposes:

-   provide information about the running system such as cpu information, information about interrupts, about the available memory or the version of the kernel.
-   information about "ide devices", "scsi devices" and "tty's".
-   networking information such as the arp table, network statistics or lists of used sockets

There is a special subdirectory: /proc/sys. It allows to configure a lot of parameters of the running system. Usually each file consists of a single value. This value may correspond to:

-   a limit (e.g. maximum buffer size)
-   turn on or off a given functionality (for example routing)
-   or represent some other kernel variable

All directories and files below /proc/sys/ are not implemented with the procfs interface. Instead they use a mechanism called sysctl. See section sysctl for further details about sysctl.

Note, despite the wide use of the procfs, it is deprecated and should only be used to export information related to a process itself.

### Implementation

In order to use the procfs it needs to be compiled with the Linux kernel source code. This is done by setting the parameter CONFIG_PROC_FS=y. In most standard configurations this is enabled by default

Procfs supports two different APIs for kernel modules: The legacy procfs API: It is easy to use as long as the amount of data to be handled is small. In this context small means smaller than one page size (PAGE_SIZE), which is in i386 systems 4096 bytes. The seq_file API: Seq_file was designed to facilitate the handling of read requests. It supports read requests for more than PAGE_SIZE bytes and it provides mechanism to traverse a list, collect the elements of the list, and send all elements to user space.

#### 1\. Legacy procfs API

```
procfs.c        legacy procfs API
```

The legacy procfs API allows for the creation of files and directories. For each file you have to specify two callback functions: One which is executed when a user reads the file and the other when a user writes to the file. The use of this API is well described in the "Linux Kernel Procfs Guide" distributed with the Linux kernel source code. Therefore we give here only a very basic example: A module which creates a directory as well as a file. If your file provides more than PAGE_SIZE bytes of data it is easy to get things wrong. This is due to the API of the read function: read(char *page, char **start, off_t off, int count, int *eof, void *data) The first parameter of this function is a buffer with the size corresponding to one page. Hence, if there is more data, the read has to be split in multiple pieces.

#### 2\. Seq_file API

The seq_file API is concerned with read requests solely - no writes. It hides the PAGE_SIZE boundary from the developer and it provides an API to step through a series of objects, collect the data from each of them and put all those data in the file. An example module can be found at <http://lwn.net/Articles/22359/>.

### Further Reading and Resources

-   <http://lwn.net/Articles/22355/> lwn article "Driver porting: The seq_file interface".

-   <http://lwn.net/Articles/22359/> Example module that uses seq_file in relation with the lwn article.

-   <http://www.linux-mag.com/id/2739?r=s> Linux magazine article "Manipulating "Seq" Files" covers legacy API as well as seq_file.

-   <http://kernelnewbies.org/Documents/SeqFileHowTo> Documents/SeqFileHowTo seqfile how-to on kernelnewbies

-   Linux Kernel Procfs Guide available in the Linux kernel source code Documentation/DocBook/procfs-guide.tmpl. Description of the legacy procfs API
-   T H E /proc F I L E S Y S T E M Description of entries in /proc, available in the Linux kernel source code Documentation/filesystems/proc.txt

Sysfs
-----

### Description

Sysfs was designed to represent the whole device model as seen from the Linux kernel. It contains information about devices, drivers and buses and their interconnections. In order to represent the hierarchy and the interconnections sysfs is heavily structured and contains a lot of links between the individual directories. As for kernel 2.6.23 it contains the following 9 top-level directories:

-   sys/block/ all known block devices such as hda/ ram/ sda/
-   sys/bus/ all registered buses. Each directory below bus/ holds by default two subdirectories:
    -   o device/ for all devices attached to that bus o driver/ for all drivers assigned with that bus.
-   sys/class/ for each device type there is a subdirectory: for example /printer or /sound
-   sys/device/ all devices known by the kernel, organised by the bus they are connected to
-   sys/firmware/ files in this directory handle the firmware of some hardware devices
-   sys/fs/ files to control a file system, currently used by FUSE, a user space file system implementation
-   sys/kernel/ holds directories (mount points) for other filesystems such as debugfs, securityfs.
-   sys/module/ each kernel module loaded is represented with a directory.
-   sys/power/ files to handle the power state of some hardware

### Implementation

In order to use sysfs it needs to be compiled with the Linux kernel source code. This is done by setting the parameter CONFIG_SYSFS=y.

The philosophy behind sysfs is to represent each value with a dedicated file. In addition each file has a maximum size of PAGE_SIZE bytes.

For a kernel module there are three possibilities to use a file below /sys:

1.  module parameter
2.  register new subsystem
3.  debugfs: debugfs, mounted in /sys/kernel/debug. More information about debugfs.

#### Module Parameter API

Similar to command line arguments for applications, Linux kernel modules may allow a set of parameters. These parameters can not only be specified upon module insertion but also during module run time. A module parameter can be defined with the following macro: module_param_named(name, value, type, perm) This macro creates a parameter called "name" which corresponds to the variable with name "value" of type "type". There are many predefined types such as byte (for a single character), int (for an integer) or charp (for a string). It is also possible to add new types. The file include/linux/stat.h provides all predefined types as well as an introduction how to define new types.

The module_param macro creates a file called /sys/modules/module_name/name with the access rights specified by perm. Depending on the specified access rights, the file - and thereby the parameter value - can be read or written. If perm is set to 0 the file is not created, and therefore the parameter cannot be accessed during run time.

The module does not receive a notification when a user reads or writes a given parameter, but the value is silently changed. Therefore it is not possible to do some additional stuff when a parameter changes its value. This may be acceptable in some circumstances as for changing a debug level, but in most circumstances the module wants to do some additional stuff such as sanity checks or manipulating a data structure.

#### Standard Sysfs API

The standard sysfs API uses a dedicated terminology: A file is called an attribute, the function executed upon reading an attribute is called show and the one for writing an attribute store.

Before starting with the implementation of a module which uses sysfs you have to figure out which subdirectory it belongs to. If you deal with a bus, it belongs to bus/, with a file system it belongs to fs/ or with a block device it belongs to block/. The API to use depends on the given subdirectory. We first show an example which uses the low level sysfs functions to add a new directory to fs/, and in a second example we show how to add a new entry to the bus/ directory.

**1\. new fs/ entry**

```
sysfs_ex.c      creates the directory /sys/fs/myfs/ along with two files first and second. Both containing one single integer value.
```

The first step is to declare our subsystem. This can be done with the use of the decl_subsys macro (on top of the file). This macro creates a struct kset with the name myfs_subsys.

The module_init() function performs the proper registration of our subsystem: The macro kobj_set_kset_s initializes myfs_subsys so that it will be part of the fs_subsys. The field myfs_subsys.kobj.ktype points to a structure which holds all the attributes as well as the functions to read and write the attributes. And finally a call to register_subsystem() registers our subsystem.

Files are generally represented by a struct attribute. This struct holds the name as well as the access permission for the corresponding file, but no data. Therefore you have to create your own attribute type which consists of at least the struct attribute and the value corresponding to that file.

By design all attributes share the same show and store functions. Each time one of these two functions is invoked it gets the corresponding struct attribute as an argument. Therefore in the show and store functions you can obtain the value corresponding to the file being read/written and you can manipulate it accordingly. For this purpose you need the macro container_of(ptr, type, member). ptr is a pointer to the member of the struct. type is the type of the struct this member is emeded in and member is the name of the member within the struct.

**2\. new bus/ entry**

```
sysfs_ex2.c     use of sysfs in combination with a bus. It provides the possibility to read and write one value with the help of "my_pseude_bus".
```

First of all we define our bus my_pseudo_bus. Then we create our attribute with the help of the BUS_ATTR macro. In the init function we register our pseudo bus and we create a file (attribute). If we would like more than one attribute we would have to use BUS_ATTR several times and provide for each attribute its own store and show function. This example is similar to the debugging facility of the scsi bus, which is implemented in drivers/scsi/scsi_debug.c Resources and Further Reading

-   Understanding the Linux Kernel p. 527
-   Linux Device Driver p. 362, 377 for the bus directory
-   Linux kernel source code: drivers/scsi/scsi_debug.c

Configfs
--------

### Description

The configfs is somewhat the counterpart of the sysfs. It can be seen as a filesystem based manager of kernel objects. An important difference between configfs and sysfs is that in configfs all objects are created from user space with a call to mkdir(2). The kernel responds with creating the attributes (files) and then they can be read and written by the user. If the user no longer needs the files, he calls rmdir(2) and everything gets deleted. Therefore the life cycle of a configfs object is fully controlled by user space.

Each time mkdir is invoked a new "config_item" is created by the kernel implementation. This config_item represents the files (attributes), the show and store callback functions as well as the associated value. Therefore each mkdir creates a new directory along with new files which represent new values.

Configfs has the same limitations than sysfs: each file should represent only one value and it should be smaller than PAGE_SIZE bytes. Implementation

In order to use configfs it needs to be compiled with the Linux kernel source code. This is done by setting the parameter CONFIG_CONFIGFS_FS=y.

In order to access configfs it has to be mounted with the following command:

```
mount -t configfs none /config
```

The Linux kernel documentation provides a good manual for configfs along with an example module. Therefore we do not describe the configfs implementation aspects.

### Resources and Further Reading

-   Linux kernel source code: Documentation/filesystems/configfs

Debugfs
-------

### Description

Debugfs is a simple to use RAM based file system especially designed for debugging purposes. Developers are encouraged to use debugfs instead of procfs in order to obtain some debugging information from their kernel code. Debugfs is quite flexible: it provides the possibility to set or get a single value with the help of just one line of code but the developer is also allowed to write its own read/write functions, and he can use the seq_file interface described in the procfs section.

### Implementation

In order to use debugfs it needs to be compiled with the Linux kernel source code. This is done by setting the parameter CONFIG_DEBUG_FS=y.

Before having access to the debugfs it has to be mounted with the following command.

```
mount -t debugfs none /sys/kernel/debug
```

```
debugfs.c       kernel module that implements the "one line" API for a variable of type u8 as well as the API with which you can specify your own read and write functions.
```

All the "one line" APIs start with debugfs_create_ and are listed in include/linux/debugfs.h

The API with which you can provide your own read and write functions is similar to the one of procfs. In contrast to sysfs, you may create directories and files without having to care about a given hierarchy. Resources and Further Reading

-   Kernel source code: include/linux/debugfs.h List of all one line APIs.
-   <http://kerneltrap.org/node/4394> announcement of the debugfs by Greg KH.

-   <http://lwn.net/Articles/115405/> article on lwn about debugfs

Sysctl
------

### Description

The sysctl infrastructure is designed to configure kernel parameters at run time. The sysctl interface is heavily used by the Linux networking subsystem. It can be used to configure some core kernel parameters; represented as files in /proc/sys/*. The values can be accessed by using cat(1), echo(1) or the sysctl(8) commands. If a value is set by the echo command it only persists as long as the kernel is running, but gets lost as soon as the machine is rebooted. In order to change the values permanently they have to be written to the file /etc/sysctl.conf. Upon restarting the machine all values specified in this file are written to the corresponding files in /proc/sys/.

### Implementation

```
sysctl.c        sysctl example module: write an integer to /proc/sys/net/test/value1 and value2 respectively
```

Each entry in the /proc/sys directory is represented by an entry in a table maintained by the Linux kernel, arranged in a hierarchy. A directory is represented by an entry pointing to a subtable. A file is represented by an entry of type struct ctl_table. This entry consists of the data represented by this file along with some access rules.

New files and directories can be added by expanding one of the subtables. In this example we add a new directory called test below the /proc/sys/net/ directory. Our directory has got two files: value1 and value2\. Each of these files hold an integer variable which can have a value between 10 and 20\. The user root is allowed to change the entries whereas normal user are allowed to read the entries.

Each file is represented with an entry in the test_table[] array:

```
    static ctl_table test_table[] = {
            {
            .ctl_name       = CTL_UNNUMBERED,
            .procname       = "value1",
            .data           = &value1,
            .maxlen         = sizeof(int),
            .mode           = 0644,
            .proc_handler   = &proc_dointvec_minmax,
            .strategy       = &sysctl_intvec,
            .extra1         = &min,
            .extra2         = &max
            },
            ...
    }
```

The struct ctl_table entries are:

-   .ctl_name: For new entries this has to be CTL_UNNUMBERED (according to Documentation/sysctl/ctl_unnumbered.txt).
-   .procname: The name of the file.
-   .data: A reference to the data we want to be shown in the file.
-   maxlen: The size of the data.
-   mode: Access permissions (read, write, execute for user, group, others)
-   proc_handler: The routine which handles read and write requests. There is a set of default routines declared near the end of include/linux/sysctl.h
-   strategy: Some routine that enforces additional access control. In this example it checks that the value to be written is between min and max.

```
    static ctl_table test_net_table[] = {
            {
                    .ctl_name       = CTL_UNNUMBERED,
                    .procname       = "test",
                    .mode           = 0555,
                    .child          = test_table
            },
            { .ctl_name = 0 }
    };
```

This table represents our test directory. The entry .child says that the elements below this directory are represented by the table test_table, discussed above.

```
    static ctl_table test_root_table[] = {
            {
                    .ctl_name       = CTL_UNNUMBERED,
                    .procname       = "net",
                    .mode           = 0555,
                    .child          = test_net_table
            },
            { .ctl_name = 0 }
    };
```

This table represents the directory to which we want to attach our new directory. In this example, this is the net directory. In the module_init() function we have to register this root table with a call to register_sysctl_table(test_root_table);

### Resources and Further Reading

-   Linux kernel source code: net/core/sysctl_net_core.c
-   Linux kernel Documentation: Documentation/sysctl/ctl_unnumbered.txt

Character Devices
-----------------

### Description

As the name suggests, this interface was designed for character device drivers, and is commonly used for communication between uer and kernel space. (For example, users with sufficient privileges my write directly to the virtual terminal 1 with echo "hi there" > /dev/tty1).

Each module can register itself as a character device and provide some read and write functions which handle the data. Files representing character devices are located within the /dev directory (where you will also find block devices, but we will not be describing them further). Usually these files correspond to a hardware device.

### Implementation

```
cdev.c  kernel module that prints its majorNumber to the system log. The minorNumber can be chosen to be 0.
```

As with all file system based approaches the module has to specify a read and a write callback function. Therefore, we have to register ourself with the function register_chrdev(unsigned int major, const char *name, struct file_operations *ops);. major is the major number of this device. We can set it to 0 to let the kernel choose an appropriate number. name is the name of this character device, as it will be shown below the /dev directory. ops is a pointer to the read and write functions.

In contrast to most file system based approaches seen so far, the user has to create the device file explicitly with a call to:

```
mknod /dev/arbitrary_name c majorNumber minorNumber
```

### Resources and Further Reading

-   man mknod(1)
-   Linux Device Driver chapter 3.

* * * * *

Socket Based Mechanisms
=======================

Introduction
------------

Beside the file system based mechanism described in the last section, there are mechanisms based on the socket interface. Sockets allow the Linux kernel to send notifications to a user space application. This is in contrast to the file system mechanisms, where the kernel can alter a file, but the user program does not know of any change until it choses to access that file. The socket based mechanisms allow the applications to listen on a socket, and the kernel can send them messages at any time. This leads to a communication mechanism in which user space and kernel space are equal partners.

There are different socket families which can be used to achieve this goal:

**AF_INET**: designed for network communication, but UDP sockets can also be used for the communication between a kernel module and the user space. The use of UDP sockets for node local communication involves a lot of overhead.

**AF_PACKET**: allows the user to define all packet headers.

**AF_NETLINK** (netlink sockets): They are especially designed for the communication between the kernel space and the user space. There are different netlink socket types currently implemented in the kernel, all of which deal with a specific subset of the networking part of the Linux kernel.

In this section we briefly cover UDP sockets, as they are the most flexible communication mechanism. Afterwards we look at netlink sockets, with a special focus on "generic netlink sockets".

UDP Sockets
-----------

### Description

We briefly describe UDP sockets, since their usage in user space is well known and they provide a lot of flexibility. With UDP sockets it is possible to have communication between a kernel module on one system and a user space application on an other machine.

### Implementation

```
udpRecvCallback.c       kernel module that provides a callback function to receive messages on port number 5555.
udpUser.c       user space program that sends a message to the kernel which sends the message back.
```

The user space program sends the string "hello world" to the kernel, which listens on port 5555\. The module prints the payload to the system log file and sends it back to the application. The application receives the message and prints the payload to stdout.

The user space code does not need any explanation, since a lot of documentation is available for socket programming. The module implementation requires some understanding of the Linux kernel. The module_init function performs three actions:

1.  Create a socket to receive UDP packets. As is well known from user space, we first create the socket and then we bind it. In addition we specify a callback function, which is executed every time a packet is received on that socket. This callback function is executed in "interrupt context". This implies that only a restricted set of operations can be performed in that function, and it is not allowed to send any messages.
2.  Therefore we define a work_queue with which we can delay the sending of the answer until we have left interrupt context.
3.  Finally, we create a socket to send the answer back to the application.

Each time the module receives a packet, the callback function cb_data() is executed. The only thing this callback function does is to submit the send_answer() function to the work_queue. If the kernel decides that there is no better work to do, it executes the send_answer function. The send_answer() function receives the packets, by dequeuing them from the sockets-receive-queue, with skb_dequeue. Since more than one packet can be received by the socket between two consecutive executions of send_answer(), this function may need to dequeue multiple messages. The number of messages in the socket queue can be obtained with a call to skb_queue_len(). After having dequeued the packet the message is printed to the system log and a message is send back to the application with the sock_sendmsg() function. Since this function assumes to be executed from user space, we have to adjust the boundaries for the allowed memory region with the help of the set_fs macro.

Netlink Sockets
---------------

### Description

Netlink is a special IPC used for transferring information between kernel and user space processes, and provides a full-duplex communication link between the Linux kernel and user space. It makes use of the standard socket APIs for user-space processes, and a special kernel API for kernel modules. Netlink sockets use the address family AF_NETLINK, as compared to AF_INET used by a TCP/IP socket.

Netlink sockets have the following advantages over other communication mechanisms:

-   It is simple to interact with the standard Linux kernel as only a constant has to be added to the Linux kernel source code. There is no risk to pollute the kernel or to drive it in instability, since the socket can immediately be used.
-   Netlink sockets are asynchronous as they provide queues, meaning they do not disturb kernel scheduling. This is in contrast to system calls which have to be executed immediately.
-   Netlink sockets provide the possibility of multicast.
-   Netlink sockets provide a truly bidirectional communication channel: A message transfer can be initiated by either the kernel or the user space application.
-   They have less overhead (header and processing) compared to standard UDP sockets.

Beside these advantages netlink sockets have two drawbacks:

-   Each entity using netlink sockets has to define its own protocol type (family) in the kernel header file include/linux/netlink.h, necessiating a kernel re-compilation before it can be used.
-   The maximum number of netlink families is fixed to 32\. If everyone registers its own protocol this number will be exhausted.

To eliminate these drawbacks the "Generic Netlink Family" was implemented. It acts as a Netlink multiplexer, in a sense that different applications may use the generic netlink address family.

Generic Netlink communications are essentially a series of different communication channels which are multiplexed on a single Netlink family. Communication channels are uniquely identified by channel numbers which are dynamically allocated by the Generic Netlink controller. Kernel or user space users which provide services, establish new communication channels by registering their services with the Generic Netlink controller. Users of the service, then query the controller to see if the service exists and to determine the correct channel number.

Each generic netlink family can provide different "attributes" and "commands". Each command has its own callback function in the kernel module and may receive messages with different attributes. Both commands and attributes, are "addressed" by an identifier.

Implementation
--------------

```
gnKernel.c      kernel module that implements a generic netlink family (CONTROL_EXMPL)
gnUser.c        user space program that sends a message to the kernel and receives an answer back
```

Here we describe how to use this generic netlink protocol to exchange data between user space and kernel space.

### User Space

Although the user space application could be written just with the help of the well known socket operations it is not reasonable to do so. For convenient user space programming there exists the libnl netlink library. It provides functions dedicated to be used for generic netlink socket communication. The libnetlink library supports generic netlink as of version 1.1, so probably you need to download the actual version from <http://people.suug.ch/~tgr/libnl/>. A program that uses this library needs to be compiled with -lnl specified.

User Space Sending Phase::

1.  nl_handle_alloc: create a socket
2.  genl_connect: connect to the NETLINK_GENERIC socket family.
3.  genl_ctrl_resolve: resolve the ID for the particular generic netlink family we want to talk with. In this example we have called the family CONTROL_EXMPL.
4.  genlmsg_put: create the generic netlink message header. In most cases you can leave all the arguments as in the example except the DOC_EXMPL_C_ECHO argument. This specifies which callback function of your kernel module gets executed. In this example there is just one callback function.
5.  nla_put_*: put the data into the message. All the possibilities are listed in the file attr.c of libnl. The second argument is used by the kernel module to distinguish which attribute was sent. In this example we have only one attribute: DOC_EXMPL_A_MSG which is a null terminated string.
6.  nl_send_auto_complete: send the message to the kernel

Receiving Phase:

1.  nl_socket_modify_cb: Add a callback function to the socket. This callback function gets executed when the socket receives a message. In the callback function the message needs to be decoded. We use nla_parse for this. Using genlmsg_parse would be more specific, but I could not link genlmsg_parse with my program. The nla_get_* functions are the counterpart of the nla_put_* functions, and are used to get a specific attribute from the message.
2.  nl_recvmsgs_default: wait until a message is received.

### Kernel Module

In the module_init function we need to register our generic netlink family with a call to genl_register_family. As an argument we have to specify a struct genl_family which holds the name of our family (CONTROL_EXMPL). In a second step we have to register the functions that get executed upon receiving a message from the user space. This is done with genl_register_ops which takes as an argument the family to which this function belongs and a struct genl_ops. This struct specifies the actual callback function as well as a security policy checked before the actual callback function gets executed. This means that if you want to receive, for example an integer, but the user space program sends a string, your callback function does not get invoked.

The actual callback function is doc_exmpl_echo. It performs two things: prints the received message, and sends a message back to the user space process. The callback function has as an argument a struct genl_info *info which holds the already parsed message. This struct contains an array which has an entry for each possible attribute. Our example has only one attribute (DOC_EXMPL_A_MSG). The data related to this attribute is saved in info->attrs[DOC_EXMPL_A_MSG];. In order to obtain the data for a given attribute there is a simple function: nla_data.

The sending process is very similar to the user space sending process.

1.  genlmsg_new: this allocates a new skb that will be sent to the user space. Since we do not yet know the final size, we use the macro NLMSG_GOODSIZE.
2.  genlmsg_put: fills the generic netlink header. All messages sent by kernel have pid = 0
3.  nla_put_string: write a string into the message.
4.  genlmsg_end: finalize the message
5.  genlmsg_unicast: send the message back to the user space program.

Resources and Further Reading
-----------------------------

-   <http://www.linuxjournal.com/article/7356> Kernel Korner - Why and How to Use Netlink Socket, Kevin Kaichuan He

-   man 3 netlink
-   man 7 netlink
-   man 3 rtnetlink
-   man 7 rtnetlink
-   <http://www.linux-foundation.org/en/Net:Generic_Netlink_HOWTO>

-   <http://lwn.net/Articles/208755/> Patch: Generic Netlink HOW-TO based on Jamal's original doc

-   <http://people.suug.ch/~tgr/libnl/> libnl - netlink library: the library along with a doxygen documentation.

* * * * *

Ioctl
=====

Introduction
------------

In addition to the read and write functions described in the section Procfs, Sysfs, and Similar Kernel Interfaces all file based mechanisms offer the possibility of additional control commands which are supported by the ioctl method.. The ioctl mechanism is implemented as a single system call which multiplexes the different commands to the appropriate kernel space function. A call to ioctl has three arguments: a file (or socket) descriptor, a number identifying the command, and a data argument. The multiplexing is done based on a) the file descriptor and b) the number of the command. Conceptually it would be possible to use any number for your new ioctl command, but this is strongly discouraged, and a system wide unique number should be used instead. This ensures that it is not possible to execute an ioctl on a wrong device leading to unexpected behavior. The exact mechanism to obtain a unique command number is described in the Linux kernel file Documentation/ioctl-number.txt.

There are different argument types for an ioctl.

-   The command does not require any data.
-   The command writes some data to the kernel.
-   The command reads some data from the kernel.
-   The kernel module reads the data and exchanges it with some new data.

The argument type can be specified when the command number is generated as described in the Linux kernel file Documentation/ioctl-number.txt.

Implementation
--------------

```
ioctl.c         kernel module that uses ioctl in combination with a character device.The ioctl allows to send a message of up to 200 bytes.
ioctl_user.c    user space program that uses ioctl to send a message to the kernel
```

In order to demonstrate the use of the ioctl mechanism we extend the character device example from section Character Device. Despite sending some data with the read and write system calls, we can now also use ioctl. We have implemented two ioctl commands one for sending data to the kernel and the other for reading data from the kernel. This extension is quite straight forward: Define your ioctl handler function in the struct file_operations (as was already done for the read and write handler function). The callback function consists mainly of a switch statement, which parses the different commands. If the specified command number does not exist ENOTTY is returned, which translates to "Inappropriate ioctl for device".

Resources and Further Reading
-----------------------------

-   Linux Device Driver p. 135
-   Linux kernel source code: Documentation/ioctl-number.txt

* * * * *

Kernel System Calls
===================

Introduction
------------

System calls are used when a user space program wants to use some data or some service provided by the Linux kernel. In current Linux kernel (2.6.23) there exist 324 system calls. All system calls provide a function which is useful for many application programs (such as file operations, network operations or process related operations). There is no point in adding a very specific system call which can be used only by a specific program.

Usually the system call gets invoked by a wrapper function provided by glibc (for example open(), socket(), getpid()). Internally each system call is identified by a unique number.

When a user space process invokes a system call, the CPU switches to kernel mode and executes the appropriate kernel function.

In order to actually do the switch from user mode to kernel mode there are some assembly instructions. For x86 architectures there are two possibilities: "int $0x80", or the newer "sysenter". Both possibilities cause:

-   the CPU to switch to kernel mode,
-   the necessary registers to be saved
-   some validity checks
-   invoke the system call corresponding to the number provided by the user space process.

If the system call service routine has finished, the system_call function checks whether there is some other work to do (such as rescheduling or signal processing) and it finally restores the user mode process context.

Implementation
--------------

System calls are a low level construct, therefore they are heavily integrated in the Linux kernel. In order to add a new system call to the Linux kernel, you have to modify a number of Linux kernel files:

1.  include/asm-i386/unistd.h
    -   This file defines the system call numbers provided by the user space to identify the system call. Add your system call at the end and increment the total number of system calls.

```
                  #define __NR_mysyscall          325
                  #define NR_syscalls 326
```

1.  include/linux/syscalls.h
    -   This file contains the declaration of all system calls. Add your system call at the end.

```
                  asmlinkage long sys_mysyscall(char __user *data, int len);
```

1.  arch/i386/kernel/syscall_table.S
    -   This file is a table with all the available system calls. In order to make your call accessible add the following line at the bottom of this file:

```
                  .long sys_mycall
```

1.  add a new directory mysyscall
    -   Since the system call has to be integrated in the kernel we add a new directory "mysyscall" for our system call in the top level directory of the Linux source code.
2.  Makefile
    -   Add the new directory to the variable core-y (search for core-y.*+=) in the top level Makefile.
3.  Write your new system call function in the file mysyscall/mysyscall.c
    -   This example prints the message provided by the user space to the system log file and exchanges it with "hello world".

```
                  #include <linux/linkage.h>
                  #include <asm/uaccess.h>
                  asmlinkage long sys_mysyscall(char __user *buf, int len)
                  {
                    char msg [200];
                    if(strlen_user(buf) > 200)
                             return -EINVAL;
                    copy_from_user(msg, buf, strlen_user(buf));
                    printk("mysyscall: %s\n", msg);
                    copy_to_user(buf, "hello world", strlen("hello world")+1);
                    return strlen("hello world")+1;
                  }
```

1.  Add a Makefile to your directory with the following line:

```
                  obj-y:= mysyscall.o
```

With this you have completed the kernel implementation. After having compiled and installed the new kernel, user space applications can make use of the new system call. The following is an example code which uses the new system call:

```
            #include <stdio.h>
            #include <string.h>
            #include <unistd.h>
            #include <sys/syscall.h>
            #define MYSYSCALL 325
            int main(){
                    char *buf [20];
                    memcpy(buf, "hi kernel", strlen("hi kernel") +1);
                    syscall(MYSYSCALL, buf, 10);
                    printf("kernel said %s\n", buf);
                    return 0;
            }
```

Resources and Further Reading
-----------------------------

-   Linux Device Driver, p. 398

* * * * *

Sending Signals from the Kernel to the User Space
=================================================

Introduction
------------

This approach is somewhat different from the others, since only the kernel can send a signal to the user space, but not vice versa. Additionally, the amount of data to be sent is quite limited. There are two types of signal APIs in user space: "normal" signals which do not have any data, and "realtime" signals which carry 32 bits of data. The main difference between them is that real time signals are queued, whereas normal signals are not. This means that if more than one normal signal is sent to a process before it is able to process it, it receives this signal only once, whereas he receives all real time signals.

The user space process registers a signal handler function with the kernel. This adds the address of the signal handler function to the process descriptor. This function gets executed each time a certain signal is delivered.

The sending phase of a signal consists of two parts:

1.  Update the process descriptor with the new signal.
2.  If this process is to be rescheduled, or if it returns from an interrupt, it first checks whether there is a signal pending. If yes, it executes first the signal handler and only then does it continue with the rest of the program.

Implementation
--------------

```
signal_kernel.c         kernel module that sends a signal to a user space process. The kernel needs to know the PID of the user space process. Therefore the user space process writes its PID in the debugfs file signalconfpid.
signal_user.c   user space program that receives the signal.
```

In order to be able to send a signal from kernel space to user space, the kernel needs to know the pid of the user space process. Therefore in this dummy example the user space process sends its pid to the kernel. As soon as the kernel module receives the pid, it looks for the corresponding process descriptor, and sends a signal to it. All information related to the signal is saved in a struct siginfo. We are able to send 32 bit of data in si_int. In order that the user space interprets this signal as a real time signal and that we can therefore access the data part, we need to set si_code to SI_QUEUE. Otherwise our data is not received by the user space function.

Resources and Further Reading
-----------------------------

-   man 7 signal
-   Understanding the Linux Kernel chapter 11

* * * * *

Upcall
======

Introduction
------------

The upcall functionality of the Linux kernel allows a kernel module to invoke a function in user space. It is possible to start a program in user space, and give it some command line arguments, as well as setting environment variables.

Implementation
--------------

```
usermodehelper.c        kernel module that starts a process
callee.c        user space program that will be executed on behalf of the kernel
```

Our example consists of a kernel module usermodehelper and a user space program callee. Since callee is not started within a shell, we cannot use printf to verify its correct execution, instead we let him run the beep command and let the kernel module specify how many times to beep. The callee gets invoked by the following function prototype: int call_usermodehelper(char *path, char **argv, char **envp, enum umh_wait wait) With the following arguments:

-   path: The path to the user space program
-   argv: The arguments for the user space program
-   envp: A set of environment varialbes
-   umh_wait: Enum that says whether the kernel module has to wait or whether it can continue with the execution:
    -   UMH_NO_WAIT: don't wait at all
    -   UMH_WAIT_EXEC: wait for the exec, but not the process
    -   UMH_WAIT_PROC: wait for the process to complete

Resources and Further Reading
-----------------------------

-   Understanding the Linux Network Internals Chapter 5

Mmap
====

Introduction
------------

Memory mapping is the only way to transfer data between user and kernel spaces that does not involve explicit copying, and is the fastest way to handle large amounts of data.

There is a major difference between the conventional read(2) and write(2) functions and mmap. While data is transfered with mmap no "control" messages are exchanged. This means that a user space process can put data into the memory, but that the kernel does not know that new data is available. The same holds for the opposite scenario: The kernel puts its data into the shared memory, but the user space process does not get a notification of this event. This characteristic implies that memory mapping has to be used with some other communication means that transfers control messages, or that the shared memory needs to be checked in regular intervals for new content. Similar to the read and write function calls mmap can be used with different file systems and with sockets.

Implementation
--------------

```
mmap_simple_kernel.c    kernel module that provides the mmap system call based on debugfs.
mmap_user.c     user space program that will share a memory area with the kernel module
```

Of course we need some memory that we want to map between user space and kernel space. In this example we share some RAM but if you are writing a device driver, this could be the memory of your device. We use debugfs and attach the memory area to a file. This allows the user space process to access the shared memory area with the help of a file descriptor.

The user space program uses the system calls open, mmap, memcpy and close, which are all documented in the Linux man pages.

The kernel module is more challenging. Please note that the discussed module offers only the most basic functionality, and that usually mmap is just one of the functions provided to handle a file. In the module_init function we create the file as discussed in section debugfs. Since it is an example module, our file_operations struct contains only three entries: my_open, my_close and my_mmap.

### my_open

In this function we allocate the memory that will later be shared with the user space process. Since memory mapping is done on a PAGE_SIZE basis we allocate one page of memory with get_zeroed_page(GFP_KERNEL) We initialize the memory with a message form the kernel that states the name of this file. We set the private_data pointer of this file to the allocated memory in order to access it later in the my_mmap and my_close function

### my_close

This function frees the memory allocated during my_open.

### my_mmap

This function initializes the vm_area_struct to point to the mmap functions specific to our implementation. mmap_open und mmap_close are used only for bookkeeping. mmap_nopages is called when the user space process references a memory area that is not in its memory. Therefore mmap_nopages does the real mapping between user space and kernel space. The most important function is virt_to_page which takes the memory area to be shared as an argument and returns a struct page * that can be used by the user space to access this memory area.

Resources and Further Reading
-----------------------------

-   man(2) mmap
-   Linux Device Driver, Chapter 15