---
title: "Learning Note: TCP/IP Network"
datePublished: Fri Feb 23 2024 02:38:50 GMT+0000 (Coordinated Universal Time)
cuid: clsy1mhvw00020ajr48tg9l8d
slug: learning-note-tcpip-network
tags: windows, networking, tcpip

---

This article is a summary of Chapter 9 of:

Yazawa, H. (2015). *计算机是怎么跑起来的\[How Computer Works\]* (Hu Y., Trans.). People's Posts and Telecommunications Press

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708650706869/3336fa3f-20c1-44dd-9c0c-40d88e33489d.png align="center")

**LAN** (Local Area Network) is an internal network among a few devices. **WAN** (Wide Area Network) are local area networks connected by **routers**, forming the complete Internet.

### MAC Address

The computer hardware used for network access is called **Network Interface Card (NIC).** Every NIC has a unique identification address stored in its ROM called Media Access Control (MAC). The MAC address is determined by the manufacturer of NIC, and is composed of manufacturer address and product address. Thus, every MAC address is unique.

In Ethernet, computers connected in LAN communicate with a mechanism called **CSMA/CD (Career Sense Multiple Access with Collision Detection)**. Here's a partition of the terminology:

1. Career Sense: each device would sense the electrical signal (career) being used in the network
    
2. Multiple Access: multiple devices can reuse the same communication medium
    
3. Collision Detection: each device will detect whether there are conflict (collision) of electrical signal due to simultaneously transmission by multiple devices.
    

With this mechanism, when a computer wants to transmit signal, it first determines whether there are other computers occuping the information channel. If multiple computers want to send signal simultaneously, they all wait for a random amount of time before sending the signal. In Ethernet, the signal send to one computer will also be received by other computers. MAC address plays the role of specifying the intended receiver.

In Windows, we can type **ipconfig /all** command to terminal to check our MAC address. A MAC address consists of six hexadecimal numbers. The first three numbers represent manufacturer and the last three represents product.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708651961233/313466c7-43de-473f-94ed-982514d99e8c.png align="center")

### IP Address

MAC address can be used to identify devices on a hardware level. However, there is no way for a company or organization to unify the leading places of their internal devices. Another address, called **IP address**, is an address of a device on software level. We can find IP address with the same command ipconfig /all

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708652599457/ab9b72de-0fd0-49c9-96bb-471d373ff9d4.png align="center")

A computer with an IP address is called a **Host**. The IP address indicates which LAN the host belongs to. Subnet mask indicates the places of IP address for the LAN address and host address. From the above example, 255.255.255.240 converted to binary is 11111111.11111111.11111111.11110000. This means that the first 28 bits of IP address is the address of LAN where the host belongs to, and last 4 bits is the address of host in the LAN.

The conversion between IP address and MAC address is implemented with a program called **ARP (Address Resolution Protocol)**. Given an IP address, ARP broadcasts through the LAN, and the computer with corresponding IP would return its MAC address. To optimize the process, ARP usually use cache to temporarily store previous mappings. We can check ARP cache with command **arp -a**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708655433346/71442ebe-1177-47f1-9a79-573b41cb5233.png align="center")

### DHCP Server

We can manually set the IP address of a computer. However, oftentimes we rely on **Dynamic Host Configuration Protocol (DHCP)** to set IP address and subnet mask for us. DHCP server stores the range of IP addresses and subnet masks that can be assigned to computers in the LAN. **Default gateway** refers to the IP address of router in the LAN, and can also be set automatically by DHCP.

### Router

**Router** is a device that transmit data among different LANs. When a computer broadcasts an IP address that doesn't belong to the LAN, the router will capture this message and sends it outside into the WAN. Each router contains a **routing table** storing routers connected to it, and message is passed from one LAN to another through passing among different routers.

To check the routing table, we can type in command **route print**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708654599646/c28d1a86-daa6-455d-aa2d-5647654e78f6.png align="center")

Network Destination, Netmask, Gateway, Interface stores information about the data destination and router IP. **Metric** indicates the weight of a route based on certain algorithm. The router tries to send the data via the route with smallest metric.

**tracert** command shows the whole routing processing that connects this device with a given host.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708654772487/96645570-9329-411a-a95c-775d8bec7c2f.png align="center")

### DNS Server

**Domain Name System (DNS)** maps an IP address with a domain name that is easier to memorize for humans. Each computer has a host name, and each LAN has a domain name. Combining host name and domain name, called FQDN (Fully Quantified Domain Name), can also specify a single computer, and is equivalent to an IP address.

DNS servers stores the mapping between FQDN and IP address. A DNS server can parse a domain name to an IP address with a mapping. If the domain name is unknown, it attemps to "ask" other DNS servers.

Command **hostname** can return the domain name of a computer. To check FQDN, we need to use ipconfig /all again. FQDN is combining host name with DNS suffix search list.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708655134067/53bcf483-2147-4801-976f-418003826ee5.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708655143700/746c405f-9068-41ae-9b9a-7c163f8b831a.png align="center")

**nslookup** command allows us to check the corresponding IP address with a given domain name.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708655198598/fd1d872c-e320-4a05-af46-de97943cf52f.png align="center")

### TCP/IP Networking Model

TCP/IP stands for TCP protocal and IP protocal. TCP protocal is ued to ensure reliable communication between data sender and receiver. TCP specifies **three handshake** to check the reliability of communication.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708655595934/5117edc1-4265-4856-8217-df5f1c6ee495.png align="center")

TCP protocal also requires data to be sent in packets. Each packet contains MAC, IP, TCP, error checking information, in addition to the content.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708655692893/9f2a4936-3761-4f5f-b2be-232ed0ada967.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708655716364/5b1f1185-89dd-4233-ae99-cdb9b7b0a23e.png align="center")

The TCP/IP network structure consists of multiple layers: the NIC hardware, device driver that controls NIC, programs that implement IP protocal, programs that implement TCP protocal, and finally applications running on top of TCP protocal.