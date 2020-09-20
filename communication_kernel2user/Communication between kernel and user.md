# Communication between kernel and user

> This Topic discusses how to communicate between kernel and user what i have known. Future articles, i will introduce in detail.



As we known, Linux is divided into kernel space and user space for security. And they can't communicate with each other directly. Therefore Linux provide several methods for communication between kernel and user.

## Overview

1. **system call**
2. **Ioctl**
3. **file system**
4. **socket**
5. **memory mapping**

## System call

Linux provide a set of standard interfaces for communication between kernel and user, we can access the system resource through the interfaces. The interfaces are called system call interface, and the calling procedure is system call.

**Advantage:**

It is more efficient than others.

**Disadvantage:**

- Inconvenient, The interface is often too primitive and we need understand more detail related to system.

- poor compatibility. It is different in different operation system. 

As for these problems, we use the library to resolve it. The libraries wrap the system call interface again, and we only need use the library functions. It is easier for development.

## Ioctl

For driver development, we usually create a charater device, and the device has several function, such as read,write,open,close and of course ioctl.

Using ioctl to realize communication between kernel and user, we realize the ioctl in kernel, and only need call ioctl function in kernel. But remember it: it's system call in nature.

**Advantage:**

It's easier then system call. we just need realize it in kernel and call ioctl function in user.

**Disadvantage:**

- It's a one-way communication
- It's better that data is command and don't recommend large data communication

## File system

We can use the file system to modify a kernel variable or get some kernel informations. 

Therefore, i think it's suitable for minor modification or debugging. For instance, we use file system to modify a kernel variable to control an system action or a set of actions. Whether it looks like a switch? Yes ,i think.

## Socket

Socket communication can be divided into netlink and socketopt

### Netlink

A good way to realize communication between kernel and user. Netlink socket is a special inter-process communication (IPC) used to realize the communication between user process and kernel process, and it is also the most commonly used interface for network application and kernel communication.

Netlink is a special socket, which is unique to Linux, similar to AF_ROUTE in BSD but far more powerful than its function. At present, there are many applications that use netlink to communicate with the kernel in the Linux kernel; including: Routing daemon (NETLINK_ROUTE), user mode socket protocol (NETLINK_USERSOCK),Firewall (NETLINK_FIREWALL), netfilter subsystem (NETLINK_NETFILTER), kernel event notification to user mode (NETLINK_KOBJECT_UEVENT),General netlink (NETLINK_GENERIC) etc.

**Advantage:**

- It's more easier, just need add a new netlink protocol in include/linux/netlink.h
- Asynchronous two-way communication
- It can be a kernel module
- It supports muticast

**Disadvantage:**

It's not suitable for large data communication, next i will introduce a more efficient way for large data communication.

## memory mapping

I prefer it when the kernel need more effectiveness. It will map a memory to user space, the user can access directly. It provides a more efficient way.

How to realize it and why it is more efficient, please refer to future articles. I will introduce it in detail.

