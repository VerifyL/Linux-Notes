[TOC]

# ARP

## ARP Overview

**conception:** Address resolution Protocol

To get MAC address through IP address. Normally if we wanna send a packet to a IP address, getting the MAC address is the first thing to wrap a packet. The host ususally maintain a IP-MAC table by ARP protocol.

**Assignment:** Creation, Query, Update and delete ARP table.

## Procedure

 ### Loacl Area Network

1. Query MAC address corresponding to destination IP address. If result hitted, goto 3.
2. Flood ARP request packets and while receiving the reply message, updating ARP table. 
3. Wrap packets in accordance with MAC address got and send it.

### Internet

1. Query the route table of host, if no result hitted and need send to gateway, and wrap packets used MAC address of gateway.
2. Query MAC address of gateway which connect to the sender host.
3. The gateway will do arp query and forward it again.

## ARP message

![ARP_format](https://user-images.githubusercontent.com/49341598/94519791-c37e9900-025d-11eb-8880-8f95e97004ac.png)

|            **Field**             | Length |                           Meaning                            |
| :------------------------------: | :----: | :----------------------------------------------------------: |
| Ethernet Address of destionation | 6Bytes |     Destination MAC address, APR request: FF:FF:FF:FF:FF     |
|    Ethernet Address of sender    | 6Bytes |                      Source MAC address                      |
|            Frame Type            | 2Bytes |              Following message type. ARP:0x0806              |
|          Hardware Type           | 2Bytes |                  Hardware type. ether: 0x1                   |
|          Protocol Type           | 2Bytes |                   Protocol type:IP(0x0800)                   |
|         Hardware Length          | 1Bytes |                  Hardware address length:6                   |
|         Protocol Length          | 1Bytes |                Protocol address (IP) length:4                |
|                OP                | 2Bytes | Operation type:<br/>- 0x1 ARP request<br/>- 0x2 ARP reply<br/>- 0x3 RARP request<br/>- 0x4 RARP reply<br/> |
|    Ethernet Address of sender    | 6Bytes |                    MAC address of sender.                    |
|       IP address of sender       | 4Bytes |                     IP address of sender                     |
| Ethernet Address of destination  | 6Bytes |                   destination MAC address                    |
|    IP Address of destination     | 4Bytes |                  IP address of destination                   |

# ***NOTE***

The ethernet frame is 64 bytes at least and 1536 at most .  For APR message:

dmac(6)+smac(6)+ 2+4 +4 +6+4+6+4 + FCS(4) = 46 bytes

So we usually add  pad field that the size is  18 bytes.

Finally, the size is 64 bytes totally.

### Code

```c
typedef struct  arp_form{
    unsigned short hdtype;
    unsigned short pro_type;
    unsigned char hd_len;
    unsigned char pro_len;
    unsigned short op;
    unsigned char mac_src[6];
    unsigned char ip_src[4];
    unsigned char mac_dest[6];
    unsigned char ip_dest[4];
    unsigned char pad[18];       
}__attribute__((__packed__)) ARP;
int main(int argc, char ** argv)
{
	struct in_addr in;
	char eth_dest[6]={0xFF,0xFF,0xFF,0xFF,0xFF,0xFF};
	char eth_src[6]={0x12,0x34,0x56,0x78,0x87,0x65};

	struct ethhdr ethhdr;
	ARP arp;

	//built an arp request message
	memcpy(ethhdr.h_dest,eth_dest,6);
	memcpy(ethhdr.h_source,eth_src,6);
	ethhdr.h_proto=htons(0x0806);

	arp.hdtype=htons(0x01);
	arp.pro_type=htons(0x0800);
	arp.hd_len=6;
	arp.pro_len=4;
	arp.op = htons(0x01);
	memcpy(arp.mac_dest,eth_dest,6);
	memcpy(arp.mac_src,eth_src,6);

	in.s_addr=inet_addr("192.168.1.97");
	memcpy(arp.ip_src,&in.s_addr,4);
	in.s_addr=inet_addr("192.168.1.24");
	memcpy(arp.ip_dest,&in.s_addr,4);
	bzero(arp.pad,18);

	return 0;
}
```

