---
title: "Learning Note: the Interactions between Memory and Hard Disk"
datePublished: Fri Dec 08 2023 16:07:28 GMT+0000 (Coordinated Universal Time)
cuid: clpwtlsyo000408l1hahwbhnq
slug: learning-note-the-interactions-between-memory-and-hard-disk
tags: cache, virtual-memory, hard-disks

---

This article is a summary of Chapter 5 of:

Yazawa, H. (2015). *程序是怎么跑起来的\[How Program Works\]* (L. Fengjun, Trans.). People's Posts and Telecommunications Press

### Programs are only execued in memory

The first thing to note about in this section is that programs stored in the hard disk must be first loaded to memory to be executed. This is because CPU reads programs by finding the their memory addresses with program counter. Also, even if CPU can directly access the hard disk, it would be highly inefficient.

Fig 1

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701977610020/56f81a62-98d8-4f20-a1b9-ba5e7bb435d2.png align="center")

(Yazama, 2015, Fig 5-1, p85)

### Disk Cache

Since memory works much faster than hard disk, the hard disk would load certain recent data to the memory, known as **disk cache**. Using cache is a common method when dealing with communications between low-speed and high-speed device. For instance, Web browsers would also store large files get from servers to the hard disk, since accessing hard disk is much faster than accessing a remote server.

### Virtual Memory

Virtual memory is a mechanism that uses part of the hard disk as "virtual" memory. The use of virtual memory allows programs to run even with insufficent memory. Since CPU can only execute programs in the physical memory, a process called **swap** is involved to move programs between physical memory and virtual memory.

There are two common swap methods, paging and segmentation. Paging divides programs according to "pages" with a specific size, without considering the program structure. The size of one page in Windows system is usually 4KB. When the physical memory is full, the operating system would move some pages to the hard disks to free up spaces, a process called **page out**. When pages from the virtual memory are needed, they are swapped back to the physical memory, a process known as **page in**.

Fig 2

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701979770103/356fb471-0146-48da-8ada-056d6ae03c90.png align="center")

(Yazama, 2015, Fig 5-3, p88)

Segmentation is a method that separates programs into variable-size blocks according to logical units, such as code, stack, heap. The swaping operation involves exchanging whole segments between physical memory and virtual memory.

### Programming Methods to Save Memory

**DDL (Dynamic Link Library)** is a file that allows programs to dynamically access library programs. The use of a shared DLL file among multiple programs to access the same library functions. For example, if programs A and B both use a function MyFunc(), adding MyFunc() separatey in different programs, known as **static link**, would make two same functions exist in the memory.

Fig 3

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701980526235/bf41519e-5ed6-4117-9cfa-86a45c52de08.png align="center")

(Yazama, 2015, Fig 5-5, p90)

However, if we make MyFunc() into a DLL file. There will only be one MyFunc() in the memory used by two programs. Also, this method allows easier program updates, as a change in MyFunc() only involves changes in DLL file, without modfying program A and B.

Fig 4

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701980661105/cda7ea7b-efc9-4766-ae75-94f0f844a49c.png align="center")

(Yazama, 2015, Fig 5-6, p91)

The use of **standard call** (like **\_stdcall** in C) is another technique to reduce program file size. The use of standard call can reduce the number of stack clearing in the program. **Stack clearing** is an operation that resets part of the stack used by the function call. This operation is automatically added by the compiler during the compiling stage, and by default, is added to the function caller. However, by modifying the function to be standard call, the stacking clearing operation is implemented in the function definiton, which saves memory if the function is called multiple times

Fig 5

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701984203815/cfbf3a0c-97b0-4e87-8bb7-a3192d502710.png align="center")

(Yazama, 2015, Fig 5-7, p94)

### The Physical Structure of Hard Disk

The hard disk divides its physical surface into many partitions, and the partition method includes **sector** method or **variable-length** method. For the sector method, the hard disk is divided into multiple concentric circles called **mangetic tracks**, each magnetic track is further divided into **sectors**. Each sector has the same storage space, and is the smallest unit for reading and writing hard disk.

Fig 6

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702051097551/e7359cec-615f-42f0-8cd5-115e398a10b9.png align="center")

(Yazama, 2015, Fig 5-8, p95)

The unit for reading and writing files on hard disk is either one sector or integer multiples of a sector. Different files cannot be stored in the same sector since it could cause one files overwritten by another one. As a result, no matter how small a file it, it always occupies at least one sector. For example, in Windows system where the sector is normally 512 bytes, a file with only 1 byte will still occupy a whole sector, and a file with 513 bytes will occupy two sectors.