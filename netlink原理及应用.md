## 什么是netlink
netlink是一种基于网络的通信机制，允许内核内部、内核与用户态应用之间甚至用户态应用之间进行通信；netlink的主要作用是内核与用户态之间通信；它的思想是，基于BSD的socket使用网络框架在内核和用户态之间进行通信；
## 为什么要有netlink
内核中有其他一些方法可以实现用户空间和内核通信，如procfs、sysfs和ioctrl等；netlink相比于这些方法，有以下优势：

- 任何一方都不需要轮询；如果通过文件通信，用户态应用需要不断检查是否有新消息到达；
- netlink使用简单，它是基于socket的，可以使用socket api；
- 只需要在netlink协议族中新增加一个协议；使用netlink的内核部分可以采用模块的方式实现，之后使用socket api进行通信；
- 内核可以直接向用户层发送信息，而无需用户层事先请求；
- netlink支持单播、组播；内核模块可以把消息发送到一个多播组；
## 数据结构
### struct sockaddr_nl
netlink是基于网络的，使用socket通信；类似于其它网络协议，每个netlink socket都需要分配一个地址；`struct sockaddr_nl`表示netlink地址；
```c
struct sockaddr_nl {
	__kernel_sa_family_t	nl_family;	/* AF_NETLINK	*/
	unsigned short	nl_pad;		/* zero		*/
	__u32		nl_pid;		/* port ID	*/
       	__u32		nl_groups;	/* multicast groups mask */
};
```

- `nl_family`，固定为`AF_NETLINK`，表示netlink协议族；

netlink协议族包含多个协议，最大值32；理论上32以内未被占用的协议号，可以用于自定义netlink协议，但这种方法并不规范，对于未来更新内核版本兼容性不友好；更加合适的方法，是在generic netlink协议族中，添加子协议，如nl80211就是generic netlink的一个子协议；
```c
#define NETLINK_ROUTE		0	/* Routing/device hook				*/
#define NETLINK_UNUSED		1	/* Unused number				*/
#define NETLINK_USERSOCK	2	/* Reserved for user mode socket protocols 	*/
#define NETLINK_FIREWALL	3	/* Unused number, formerly ip_queue		*/
#define NETLINK_SOCK_DIAG	4	/* socket monitoring				*/
#define NETLINK_NFLOG		5	/* netfilter/iptables ULOG */
#define NETLINK_XFRM		6	/* ipsec */
#define NETLINK_SELINUX		7	/* SELinux event notifications */
#define NETLINK_ISCSI		8	/* Open-iSCSI */
#define NETLINK_AUDIT		9	/* auditing */
#define NETLINK_FIB_LOOKUP	10	
#define NETLINK_CONNECTOR	11
#define NETLINK_NETFILTER	12	/* netfilter subsystem */
#define NETLINK_IP6_FW		13
#define NETLINK_DNRTMSG		14	/* DECnet routing messages */
#define NETLINK_KOBJECT_UEVENT	15	/* Kernel messages to userspace */
#define NETLINK_GENERIC		16
/* leave room for NETLINK_DM (DM Events) */
#define NETLINK_SCSITRANSPORT	18	/* SCSI Transports */
#define NETLINK_ECRYPTFS	19
#define NETLINK_RDMA		20
#define NETLINK_CRYPTO		21	/* Crypto layer */
#define NETLINK_SMC		22	/* SMC monitoring */

#define NETLINK_INET_DIAG	NETLINK_SOCK_DIAG

#define MAX_LINKS 32	
```

- `nl_pid`，socket的唯一标识符；对内核自身来说，该字段是0，而用户空间的应用程序通常使用其线程组ID；netlink并没有要求该字段是进程ID，它可以是任何值，只需要保证其唯一性；使用线程组ID不过是方便而已；`nl_pid`是一个单播地址；
- `nl_groups`，多播组掩码，每个bit表示一个多播组；每个netlink协议族最多支持32个多播组；
## netlink内核核心函数
### netlink_kernel_create
内核创建netlink socket；
```c
static inline struct sock *
netlink_kernel_create(struct net *net, int unit, struct netlink_kernel_cfg *cfg)
{
	return __netlink_kernel_create(net, unit, THIS_MODULE, cfg);
}
```

- `net`，表示网络命令空间；
- `uint`，表示netlink子协议族，如：
```c
#define NETLINK_ROUTE		0	/* Routing/device hook				*/
#define NETLINK_GENERIC		16
```

- `cfg`，netlink kernel创建socket的可选参数；其中，`input`是该内核netlink模块收到消息后的处理函数；
```c
/* optional Netlink kernel configuration parameters */
struct netlink_kernel_cfg {
	unsigned int	groups;
	unsigned int	flags;
	void		(*input)(struct sk_buff *skb);
	struct mutex	*cb_mutex;
	int		(*bind)(struct net *net, int group);
	void		(*unbind)(struct net *net, int group);
	bool		(*compare)(struct net *net, struct sock *sk);
};
```
## netlink消息格式
netlink消息由两部分组成：消息头和消息体；消息头固定为16字节，消息体长度可变；
![image.png](https://cdn.nlark.com/yuque/0/2024/png/756577/1709119003206-7502ddee-f53a-433a-9eab-cf6195e752a8.png#averageHue=%23fefdfd&clientId=ud6c835e9-5fd0-4&from=paste&height=134&id=ud55cd748&originHeight=147&originWidth=798&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=17127&status=done&style=none&taskId=ub345f1d5-b29f-4a0c-9c0c-44d0c030bec&title=&width=725.454529730742)
### 消息头
消息头定义如下：
```c
struct nlmsghdr {
	__u32		nlmsg_len;	/* Length of message including header */
	__u16		nlmsg_type;	/* Message content */
	__u16		nlmsg_flags;	/* Additional flags */
	__u32		nlmsg_seq;	/* Sequence number */
	__u32		nlmsg_pid;	/* Sending process port ID */
};
```

- `nlmsg_len`，整个消息的长度，包括消息头；
- `nlmsg_type`，消息类型，netlink定义一下四种通用消息类型
```c
#define NLMSG_NOOP		0x1	/* Nothing.		*/
#define NLMSG_ERROR		0x2	/* Error		*/
#define NLMSG_DONE		0x3	/* End of a dump	*/
#define NLMSG_OVERRUN		0x4	/* Data lost		*/

#define NLMSG_MIN_TYPE		0x10	/* < 0x10: reserved control messages */
```

- `nlmsg_flags`，消息标志；如`NLM_F_REQUEST`等
```c
/* Flags values */

#define NLM_F_REQUEST		0x01	/* It is request message. 	*/
#define NLM_F_MULTI		0x02	/* Multipart message, terminated by NLMSG_DONE */
#define NLM_F_ACK		0x04	/* Reply with ack, with zero or error code */
#define NLM_F_ECHO		0x08	/* Echo this request 		*/
#define NLM_F_DUMP_INTR		0x10	/* Dump was inconsistent due to sequence change */
#define NLM_F_DUMP_FILTERED	0x20	/* Dump was filtered as requested */

/* Modifiers to GET request */
#define NLM_F_ROOT	0x100	/* specify tree	root	*/
#define NLM_F_MATCH	0x200	/* return all matching	*/
#define NLM_F_ATOMIC	0x400	/* atomic GET		*/
#define NLM_F_DUMP	(NLM_F_ROOT|NLM_F_MATCH)

/* Modifiers to NEW request */
#define NLM_F_REPLACE	0x100	/* Override existing		*/
#define NLM_F_EXCL	0x200	/* Do not touch, if it exists	*/
#define NLM_F_CREATE	0x400	/* Create, if it does not exist	*/
#define NLM_F_APPEND	0x800	/* Add to end of list		*/

/* Flags for ACK message */
#define NLM_F_CAPPED	0x100	/* request was capped */
#define NLM_F_ACK_TLVS	0x200	/* extended ACK TVLs were included */
```

- `nlmsg_seq`，消息序列号，表示一系列消息之间在时间上的前后关系；也可以通过request消息和ack消息使用相同的序列号，保证消息不丢失；
- `nlmsg_pid`，消息发送者的port id；
### 消息体
netlink协议并没有严格要求消息体的格式，可以发送任意消息；但一般标准做法，消息体是用`nlattr`，即属性，采用tlv的形式；消息体组织形式如下：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/756577/1709121732240-45e8398f-6b4b-4a2e-b24a-aaf0722a1228.png#averageHue=%23fafafa&clientId=ud6c835e9-5fd0-4&from=paste&height=265&id=u6839d796&originHeight=292&originWidth=693&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=18917&status=done&style=none&taskId=u00c31041-f946-4f42-9445-757c39818a9&title=&width=629.999986345118)

`struct nlattr`定义如下：
```c
/*
 *  <------- NLA_HDRLEN ------> <-- NLA_ALIGN(payload)-->
 * +---------------------+- - -+- - - - - - - - - -+- - -+
 * |        Header       | Pad |     Payload       | Pad |
 * |   (struct nlattr)   | ing |                   | ing |
 * +---------------------+- - -+- - - - - - - - - -+- - -+
 *  <-------------- nlattr->nla_len -------------->
 */

struct nlattr {
	__u16           nla_len;
	__u16           nla_type;
};
```

## netlink协议族组织形式
netlink协议族、子协议族、子协议、命令，组织结构如下：
![image.png](https://cdn.nlark.com/yuque/0/2024/png/756577/1709123598869-105884c0-3fef-47fe-93df-28134ce2eeb5.png#averageHue=%23fbfbfb&clientId=ud6c835e9-5fd0-4&from=paste&height=505&id=u0ac981b3&originHeight=556&originWidth=1647&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=89303&status=done&style=none&taskId=ude8c0315-83bc-43f9-8357-66a0a999cd4&title=&width=1497.2726948202155)

## 如何新增netlink子协议族
如何将自定义netlink协议加入到netlink协议族中，于`NETLINK_GENERIC`同一级别？只需定义一个netlink协议号即可，由于netlink对消息体格式不做强制要求，可以传输简单的字符串；实际使用中，不建议这样做，但作为学习，可以简单的这样操作；实际使用中增加自定义netlink协议，建议加入到`NETLINK_GENERIC`协议族中，类似`nl80211`这样；
下面代码，是直接在netlink中直接加入新的协议，定义协议号为`30`；内核中新增一个模块，处理该协议的消息；应用程序通过该协议，和内核通信；简单起见，直接传输字符串；应用程序先向内核发送一条消息，内核收到消息后进行回复；
### 内核代码
内核代码如下：
```c

#include <linux/init.h>
#include <linux/module.h>
#include <linux/types.h>
#include <net/sock.h>
#include <linux/netlink.h>

#define NETLINK_TEST     30
#define MSG_LEN            125

MODULE_LICENSE("GPL");

struct sock *nlsk = NULL;
extern struct net init_net;

int send_usrmsg(char *pbuf, uint16_t len, uint32_t pid)
{
    struct sk_buff *nl_skb;
    struct nlmsghdr *nlh;

    int ret;

    /* Allocate a new netlink message */
    nl_skb = nlmsg_new(len + 1, GFP_ATOMIC);
    if(!nl_skb)
    {
        printk("\nError:netlink alloc failure.\n\n");
        return -1;
    }

    /* Add a new netlink message to an skb
        pid是0，说明是从内核发送的
    */
    nlh = nlmsg_put(nl_skb, 0, 0, NETLINK_TEST, len, 0);
    if(nlh == NULL)
    {
        printk("\nError:nlmsg_put failaure. \n\n");
        nlmsg_free(nl_skb);
        return -1;
    }

    /* copy payload */
    memcpy(nlmsg_data(nlh), pbuf, len);
    ret = netlink_unicast(nlsk, nl_skb, pid, MSG_DONTWAIT);

    return ret;
}

static void netlink_rcv_msg(struct sk_buff *skb)
{
    struct nlmsghdr *nlh = NULL;
    char *umsg = NULL;
    char *kmsg = "Hello user's program.";

    if(skb->len >= nlmsg_total_size(0))
    {
        nlh = nlmsg_hdr(skb);
        umsg = NLMSG_DATA(nlh);
        if(umsg)
        {
            printk("kernel recv from user space: %s\n", umsg);
            send_usrmsg(kmsg, strlen(kmsg), nlh->nlmsg_pid);
        }
    }
}

struct netlink_kernel_cfg cfg = {
        .input  = netlink_rcv_msg, /* set recv callback */
};

int test_netlink_init(void)
{
    /* create netlink socket */
    nlsk = (struct sock *)netlink_kernel_create(&init_net, NETLINK_TEST, &cfg);
    if(nlsk == NULL)
    {
        printk("\nError:netlink_kernel_create error !\n");
        return -1;
    }
    printk("\ntest_netlink_init\n");

    return 0;
}

void test_netlink_exit(void)
{
    if (nlsk){
        netlink_kernel_release(nlsk); /* release ..*/
        nlsk = NULL;
    }
    printk("test_netlink_exit!\n");
}

module_init(test_netlink_init);
module_exit(test_netlink_exit);
```
```c
#
#Desgin of Netlink
#
MODULE_NAME :=nl_test_kernel
obj-m:=$(MODULE_NAME).o
	

KERNELDIR ?=/lib/modules/$(shell uname -r)/build
PWD :=$(shell pwd)



all:

	$(MAKE) -C $(KERNELDIR) M=$(PWD)

clean:

	$(MAKE) -C $(KERNELDIR) M=$(PWD) clean

```
`nl_test_kernel.c`和`Makefile`放到同一目录下；直接`make`，编译生成`nl_test_kernel.ko`；
`insmod nl_test_kernel.ko`，将该模块加载到内核中；内核现在就可以处理`NETLINK_TEST`的消息了；
![image.png](https://cdn.nlark.com/yuque/0/2024/png/756577/1709175773827-c26e6da1-5e0c-4f18-b3f2-2528b9f26abd.png#averageHue=%23090806&clientId=ud6c835e9-5fd0-4&from=paste&height=182&id=u93fde49b&originHeight=200&originWidth=649&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=18179&status=done&style=none&taskId=uf3eb40c8-67d5-491e-a7fd-adc2287639b&title=&width=589.9999872120947)
### 应用程序代码
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <string.h>
#include <linux/netlink.h>
#include <stdint.h>
#include <unistd.h>
#include <errno.h>

#define NETLINK_TEST    30
#define MSG_LEN         125
#define MAX_PLOAD       125

typedef struct _user_msg_info
{
    struct nlmsghdr hdr;
    char  msg[MSG_LEN];
} user_msg_info;

int main(int argc, char **argv)
{
    int skfd;
    int ret;
    user_msg_info u_info;
    socklen_t len;
    struct nlmsghdr *nlh = NULL;
    struct sockaddr_nl saddr, daddr;
    char *umsg = "Hello Netlink protocol.";

    /* 创建NETLINK socket */
    skfd = socket(AF_NETLINK, SOCK_RAW, NETLINK_TEST);
    if(skfd == -1)
    {
        perror("\nError:Create socket error.\n");
        return -1;
    }

    memset(&saddr, 0, sizeof(saddr));
    saddr.nl_family = AF_NETLINK; //AF_NETLINK
    saddr.nl_pid = getpid();  //端口号(port ID)
    saddr.nl_groups = 0;
    if(bind(skfd, (struct sockaddr *)&saddr, sizeof(saddr)) != 0)
    {
        perror("\nError:bind() error.\n");
        close(skfd);
        return -1;
    }

    memset(&daddr, 0, sizeof(daddr));
    daddr.nl_family = AF_NETLINK;
    daddr.nl_pid = 0; // to kernel
    daddr.nl_groups = 0;

    nlh = (struct nlmsghdr *)malloc(NLMSG_SPACE(MAX_PLOAD));
    memset(nlh, 0, sizeof(struct nlmsghdr));
    nlh->nlmsg_len = NLMSG_SPACE(MAX_PLOAD);
    nlh->nlmsg_flags = 0;
    nlh->nlmsg_type = 0;
    nlh->nlmsg_seq = 0;
    nlh->nlmsg_pid = saddr.nl_pid; //self port

    memcpy(NLMSG_DATA(nlh), umsg, strlen(umsg));
    ret = sendto(skfd, nlh, nlh->nlmsg_len, 0, (struct sockaddr *)&daddr, sizeof(struct sockaddr_nl));
    if(!ret)
    {
        perror("\nError:sendto error.\n");
        close(skfd);
        exit(-1);
    }
    printf("\nApplication-->Send to kernel:%s\n\n", umsg);

    memset(&u_info, 0, sizeof(u_info));
    len = sizeof(struct sockaddr_nl);
    ret = recvfrom(skfd, &u_info, sizeof(user_msg_info), 0, (struct sockaddr *)&daddr, &len);
    if(!ret)
    {
        perror("\nError:recv form kernel error.\n");
        close(skfd);
        exit(-1);
    }

    printf("\nApplication-->From kernel:%s\n\n", u_info.msg);
    close(skfd);

    free((void *)nlh);
    return 0;
}
```

`gcc -o nl_test_user nl_test_user.c`
### 测试结果
![image.png](https://cdn.nlark.com/yuque/0/2024/png/756577/1709175888223-afdb04ef-6b9e-4159-9962-b55207905d1e.png#averageHue=%23080705&clientId=ud6c835e9-5fd0-4&from=paste&height=174&id=u892117e2&originHeight=191&originWidth=763&originalType=binary&ratio=1.100000023841858&rotation=0&showTitle=false&size=20692&status=done&style=none&taskId=u537a7566-a7b3-4bef-a34a-9e914728e4f&title=&width=693.6363486022007)
## 如何新增自定义netlink协议
如何在`NETLINK_GENERIC`中新增netlink协议？
参考nl80211

模块初始化时，通过`genl_register_family`注册通用netlink协议族，将命令以及处理函数进行注册；
```c
/* initialisation/exit functions */

int __init nl80211_init(void)
{
	int err;

	err = genl_register_family(&nl80211_fam);
	if (err)
		return err;

	err = netlink_register_notifier(&nl80211_netlink_notifier);
	if (err)
		goto err_out;

	return 0;
 err_out:
	genl_unregister_family(&nl80211_fam);
	return err;
}
```

```c
/**
 * genl_register_family - register a generic netlink family
 * @family: generic netlink family
 *
 * Registers the specified family after validating it first. Only one
 * family may be registered with the same family name or identifier.
 *
 * The family's ops, multicast groups and module pointer must already
 * be assigned.
 *
 * Return 0 on success or a negative error code.
 */
int genl_register_family(struct genl_family *family)
```

```c
static const struct genl_ops nl80211_ops[] = {
	{
		.cmd = NL80211_CMD_GET_WIPHY,
		.doit = nl80211_get_wiphy,
		.dumpit = nl80211_dump_wiphy,
		.done = nl80211_dump_wiphy_done,
		.policy = nl80211_policy,
		/* can be retrieved by unprivileged users */
		.internal_flags = NL80211_FLAG_NEED_WIPHY |
				  NL80211_FLAG_NEED_RTNL,
	},
	{
		.cmd = NL80211_CMD_SET_WIPHY,
		.doit = nl80211_set_wiphy,
		.policy = nl80211_policy,
		.flags = GENL_UNS_ADMIN_PERM,
		.internal_flags = NL80211_FLAG_NEED_RTNL,
	},
    ......
}
```

