---
title: "Learning Note: The Environments where Programs Run"
datePublished: Sun Dec 10 2023 15:23:19 GMT+0000 (Coordinated Universal Time)
cuid: clpzmwqub000108jv21ja6u2l
slug: learning-note-the-environments-where-programs-run
tags: jvm, virtual-machine, operating-system, ports, runtime-environment

---

This article is a summary of Chapter 7 of:

Yazawa, H. (2015). *程序是怎么跑起来的\[How Program Works\]* (L. Fengjun, Trans.). People's Posts and Telecommunications Press

### Runtime Environment = Operating System + Hardware

The **source code** written by programmers can be displayed an edited on any environments. However, when source code is compiled to machine language called **native code**, it can be executed in specific runtime environments, with given hardware requirements and operating system. Different CPU types, like x86, SPARC, PowerPC, can only execute its speicific type of machine langugae.

Today's operating systems like Windows can resolve hardware differences, besides the difference in CPU, on different computers. In early operating systems like MS-DOS, the control of hardware devices like memory, hard disk, I/O device are varied among different types of computers. As a result, one software program can only run on one type of hardware. Operating systems add an layer between software and hardware. Instead of directly controlling hardware, software sends command to the operating system, which controls the hardware.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702219042887/b917560f-9482-4f54-8818-375cdfd6ac8d.png align="center")

Fig 1. The difference between MS-DOS and Windows, Image reproduced from (Yazama, 2015)

Operating systems provide **API (Application Programming Interface)** for programs to invoke. Different operating systems provide different APIs for hardware control. As a result, for a program to be ported to another operating system, the API it uses need to be rewritten.

### FreeBSD Port Allows Flexible use of Source Code

**Ports** in FreeBSD operating system resolves the difference in CPU by directly porting the source code and compile the source code on local machines. If the targeted source code is not stored in the computer, Ports would use FTP to download code from remote sources.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702220017853/0896fd04-2df7-4a81-aa35-9642ff084dd7.png align="center")

Fig 2. Ports in FreeBSD. Image reproduced from (Yazama, 2015)

### Unifying Runtime Environments using Virtual Machines

Another way to resolve differences in hardware and operating systems than porting is the use of **virtual machines**. The most famous use of virtual machine would probably be Java, which compiles source code to **bytecode** that are run on **Java Virtual Machine (JVM)**. For example, for a source code sample.java, the compiler first convets it to bytecode (sample.class), and JVM (java.exe) then converts it to native code on x86 CPU.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702220812364/f6588489-a2cb-44b1-b5b1-7a49085fbb72.png align="center")

Fig 3. JVM further covers differences in hardware and operating systems. Image reproduced from (Yazama, 2015)

However, this approach still has some limitations. Firstly, sometimes bytecodes may not be perfectly reusable for all JVMs if we are using certain hardware-specific features. Secondly, adding an additional layer between programs and operating systems compromises efficiency. There are certain approaches to improve efficiency, though. For example, we can store the native code after complication, or optimize time-consuming parts of bytecode processing.

### BIOS Bootstrap

**BIOS (Basic Input/Output System)** is a program pre-stored in ROM. It contains control programs of keyboard, hard disk, GPU, and a bootstrap program used to launch operating system. When the computer starts, BIOS first check the functioning of hardware, and then the bootstrap program would load operating system from hard disk to memory to launch the OS.