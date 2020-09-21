# Communication between kernel and user

> This topic discusses how to communicate between kernel and user what i have known. Future articles, i will introduce in detail.



As we have known, Linux is divided into kernel space and user space for security. And they can't communicate with each other directly. Therefore Linux provides several methods for communication between kernel and user.

## Overview

1. **system call**
2. **Ioctl**
3. **file system**
4. **socket**
5. **memory mapping**

## System call

Linux provides a set of standard interfaces for communication between kernel and user, we can access the system resource through the interfaces. The interfaces are called system call interface, and the calling procedure is a system call.

**Advantage:**

It is more efficient than others.

**Disadvantage:**

- Inconvenient, The interface is often too primitive and we need to understand more detail related to the system.

- poor compatibility. It is different in different operating systems. 

As for these problems, we use the library to resolve it. The libraries wrap the system call interface again, and we only need to use the library functions. It is easier for developer.

## Ioctl

For driver development, we usually create a character device, and the device has several functions, such as read,write,open,close and of course ioctl.

Using ioctl to realize communication between kernel and user, we realize the ioctl in kernel, and only need call ioctl function in kernel. But remember it: it's system call in nature.

**Advantage:**

It's easier than a system call. we just need to realize it in kernel and call ioctl function in user.

**Disadvantage:**

- It's a one-way communication
- It's better that data is command and don't recommend large data communication

## File system

We can use the file system to modify a kernel variable or get some kernel information. 

Therefore, I think it's suitable for minor modification or debugging. For instance, we use file system to modify a kernel variable to control a system action or a set of actions. Whether it looks like a switch? Yes, I think.

## Socket

Socket communication can be divided into Netlink and socketopt

### Netlink

A good way to realize communication between the kernel and user. Netlink socket is a special inter-process communication (IPC) used to realize the communication between the user process and kernel process, and it is also the most commonly used interface for network application and kernel communication.

Netlink is a special socket, which is unique to Linux, similar to AF_ROUTE in BSD but far more powerful than its function. At present, there are many applications that use N	etlink to communicate with the kernel in the Linux kernel; including: Routing daemon (NETLINK_ROUTE), user-mode socket protocol (NETLINK_USERSOCK),Firewall (NETLINK_FIREWALL), netfilter subsystem (NETLINK_NETFILTER), kernel event notification to user mode (NETLINK_KOBJECT_UEVENT),General Netlink (NETLINK_GENERIC), etc.

**Advantage:**

- It's easier, just need to add a new Netlink protocol in include/linux/netlink.h
- Asynchronous two-way communication
- It can be a kernel module
- It supports multicast

**Disadvantage:**

It's not suitable for large data communication, next I will introduce a more efficient way for large data communication.

## memory mapping

I prefer it when the kernel needs more effectiveness. It will map memory to user-space, the user can access directly. It provides a more efficient way.

How to realize it and why it is more efficient, please refer to future articles. I will introduce it in detail.

