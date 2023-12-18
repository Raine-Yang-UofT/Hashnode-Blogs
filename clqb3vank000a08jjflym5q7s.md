---
title: "Learning Note: Hardware Control"
datePublished: Mon Dec 18 2023 16:03:33 GMT+0000 (Coordinated Universal Time)
cuid: clqb3vank000a08jjflym5q7s
slug: learning-note-hardware-control
tags: io, assembly, dma, vram, interruption, irq

---

This article is a summary of Chapter 11 of:

Yazawa, H. (2015). *程序是怎么跑起来的\[How Program Works\]* (L. Fengjun, Trans.). People's Posts and Telecommunications Press

### I/O Command for Hardware Input/Output

Since different peripheral devices have different voltage, digital signal and analog signal, **I/O controllers** are used to establish connections between CPU and peripheral devices. An I/O controller controls one or more peripheral devices. Every I/O controller has memory, known as **port**, to temporarily store data. Different I/O controllers are distinguished by **port number**, also known as **I/O address**. **IN** and **OUT** command in assembly are used to transfer data between CPU and ports with given port numbers.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702911377922/4b59b1f9-e8ee-4d3e-b8e1-09c542c8fedf.png align="center")

Fig 1. Connection between CPU and peripheral device with I/O controller. Image reproduced from (Yazama, 2015)

Here is a simple example that controls the buzzer in computer with IN/OUT command:

```c
void main() {
    _asm {
        IN    EAX,    61H
        OR    EAX,    03H
        OUT   61H,    EAX
    }

    for (int i = 0; i < 1000000; i++);

    _asm {
        IN    EAX,    61H
        AND   EAX,    0FCH
        OUT   61H,    EAX
    }
} 
```

The default port number of buzzer is 61H. To turn on the buzzer, we need to set the lowest two places to 1, which is done through OR calculation with 03H (00000011). This OR calculation changes the last two places to 1 and makes no changes to other places. The command is sent to the buzzer with OUT.

Similarly, to turn off thr buzzer, we set the lowest two places to 0 by AND calculation with 0FCH (hexadecimal numbers starting with letters need to add a 0 at front in assembly). The AND calculation changes the lowest two places to 0 and does not affect other places.

### Interruption Request (IRQ)

**Interruption** is a mechanism that pauses the current program and starts the run another program. The control of peripheral hardware is normally implemented through interruption mechanism, since most hardware runs much slower than CPU, and CPU can execute other programs when the other hardware are running, and process the hardware command only when it sends interruption requests.

Peripheral hardware are in charge of sending interruption requests, and the CPU is used to process these requests. The interruption requests have a specific number called **interruption number**, which differs from port number. Multiple interruption requests are sent to a buffer called **interruption controller**, which sends the requests to CPU in order.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702914641010/26a958b4-16ff-4f8b-bd11-2ec1044667ba.png align="center")

Fig 2. The Function of Interruption Controller. Image reproduced from (Yazama, 2015)

When CPU receives an interruption request, it first stores all value in registers to stack and terminates the process running. It then executes the program that handles the interruption. When the inteeruption request is processed, the CPU retrieves previously stored data from stack and continues the program.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702914782581/9f78e7d3-c7a2-4bde-8aac-cb43e9eb9d0a.png align="center")

Fig 4. The Process of Interruption Handling. Image reproduced from (Yazama, 2015)

### Transfering Data with DMA

Apart from I/O and interruption, **DMA (Direct Memory Access)** is another mechanism in hardware control. DMA refers to drect exchanging data between memory and peripheral device without the use of CPU. This allows quick transfer of large quantity of data.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702915118894/58c66668-f857-4cd2-9944-3e8981e7792e.png align="center")

Fig 5. Transferring Data with DMA. Image reproduced from (Yazama, 2015)

### The Display of Text and Images on Monitor

Data displayed on the monitor are stored in a part of memory called **VRAM (Video RAM)**. When we write data into VRAM, it will be displayed on the monitor. At the age of MS-DOS, VRAM used to be part of the main memory, and was very limited. Today's computers with GUIs normal have VRAM and GPU (Graphics Processing Unit) independent from the main memory.