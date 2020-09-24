[TOC]

# Sockopt

It is easier way to communicate between kernel and user. However, Its disadvantage is that it use copy_from_user()/copy_to_user() to achieve the communication. 

So, it cannot use for interrupt and is not suitable for large data transmission.

# Implementation

## Kernel

1. Register 

   ```c
   nf_register_sockopt(struct nf_sockopt_ops *sockops);
   ```

2. Unregister

   ```C
   nf_unregister_sockopt(struct nf_sockopt_ops *sockops);
   ```

3. ```C
   static struct nf_sockopt_ops nso = {  
    .pf  = PF_INET,       // Protocol family 
    .set_optmin = constant,    // Minimum set command word 
    .set_optmax = constant+N,  // Maximum set command word 
    .set  = set_function,   // set function  
    .get_optmin = constant,    // Minimum get command word  
    .get_optmax = constant+N,  // Maximum get command word 
    .get  = get_function,   // get function 
   };  
   Among them, the command word cannot be repeated with the existing kernel, it should be big rather than small. The command word is very important and is used as an identifier. And user mode and kernel mode should be defined the same,
   ```

4. using the **copy_from_user** to receive data from user and **copy_to_user** to send data to user.



## User

1. Creation socket

   ```C
   socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
   ```

2. Transmission

   ```c
   int setsockopt ( int sockfd, int proto, int cmd, void *data, int datelen);
   ```

3. Reception

   ```C
   int getsockopt(int sockfd, int proto, int cmd, void *data, int datalen);
   ```



# Example

## Kernel

```C
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/types.h>
#include <linux/string.h>
#include <linux/netfilter_ipv4.h>
#include <linux/init.h>
#include <asm/uaccess.h> 
 
#define SOCKET_OPS_BASE          128
#define SOCKET_OPS_SET       (SOCKET_OPS_BASE)
#define SOCKET_OPS_GET      (SOCKET_OPS_BASE)
#define SOCKET_OPS_MAX       (SOCKET_OPS_BASE + 1)
 
#define KMSG          "--------kernel---------"
#define KMSG_LEN      sizeof("--------kernel---------")
 
MODULE_LICENSE("GPL");
MODULE_AUTHOR("SiasJack");/*作者*/
MODULE_DESCRIPTION("sockopt module,simple module");//描述
MODULE_VERSION("1.0");//版本号
 
static int recv_msg(struct sock *sk, int cmd, void __user *user, unsigned int len)
{
    int ret = 0;
    printk(KERN_INFO "sockopt: recv_msg()\n"); 
 
    if (cmd == SOCKET_OPS_SET)
    {   
        char umsg[64];
        int len = sizeof(char)*64;
        memset(umsg, 0, len);
        ret = copy_from_user(umsg, user, len);
        printk("recv_msg: umsg = %s. ret = %d\n", umsg, ret);    
    }   
    return 0;
} 
 
static int send_msg(struct sock *sk, int cmd, void __user *user, int *len)
{
    int ret = 0;
    printk(KERN_INFO "sockopt: send_msg()\n"); 
    if (cmd == SOCKET_OPS_GET)
    {   
        ret = copy_to_user(user, KMSG, KMSG_LEN);
        printk("send_msg: umsg = %s. ret = %d. success\n", KMSG, ret);
    }   
    return 0;
 
}
 
static struct nf_sockopt_ops test_sockops =
{
    .pf = PF_INET,
    .set_optmin = SOCKET_OPS_SET,
    .set_optmax = SOCKET_OPS_MAX,
    .set = recv_msg,
    .get_optmin = SOCKET_OPS_GET,
    .get_optmax = SOCKET_OPS_MAX,
    .get = send_msg,
    .owner = THIS_MODULE,
};
 
static int __init init_sockopt(void)
{
    printk(KERN_INFO "sockopt: init_sockopt()\n");
    return nf_register_sockopt(&test_sockops);
}
 
static void __exit exit_sockopt(void)
{
    printk(KERN_INFO "sockopt: fini_sockopt()\n");
    nf_unregister_sockopt(&test_sockops);
}
 
module_init(init_sockopt);
module_exit(exit_sockopt);
```



## User

```C
#include <unistd.h>
#include <stdio.h>
#include <sys/socket.h>
#include <linux/in.h>
#include <string.h>
#include <errno.h> 
 
#define SOCKET_OPS_BASE      128
#define SOCKET_OPS_SET       (SOCKET_OPS_BASE)
#define SOCKET_OPS_GET      (SOCKET_OPS_BASE)
#define SOCKET_OPS_MAX       (SOCKET_OPS_BASE + 1) 
 
#define UMSG      "----------user------------"
#define UMSG_LEN  sizeof("----------user------------") 
 
char kmsg[64]; 
 
int main(void)
{
    int sockfd;
    int len;
    int ret; 
 
    sockfd = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
    if(sockfd < 0)
    {   
        printf("can not create a socket\n");
        return -1; 
    }   
 
    /*call function recv_msg()*/
    ret = setsockopt(sockfd, IPPROTO_IP, SOCKET_OPS_SET, UMSG, UMSG_LEN);
    printf("setsockopt: ret = %d. msg = %s\n", ret, UMSG);
    len = sizeof(char)*64; 
 
    /*call function send_msg()*/
    ret = getsockopt(sockfd, IPPROTO_IP, SOCKET_OPS_GET, kmsg, &len);
    printf("getsockopt: ret = %d. msg = %s\n", ret, kmsg);
    if (ret != 0)
    {   
        printf("getsockopt error: errno = %d, errstr = %s\n", errno, strerror(errno));
    }   
 
    close(sockfd);
    return 0;
}
```



![nf_sockopt通信](https://user-images.githubusercontent.com/49341598/94114936-99069780-fe7b-11ea-9a1c-7285c32e311f.jpg)