---
title: "Learning Notes: What CPU means for programmers"
datePublished: Fri Nov 24 2023 13:48:24 GMT+0000 (Coordinated Universal Time)
cuid: clpcoh1fh000609jo5z4ybl8g
slug: learning-notes-what-cpu-means-for-programmers
tags: cpu, register, assembly

---

This article is a summary of Chapter 1 of:

Yazawa, H. (2015). *程序是怎么跑起来的\[How Program Works\]* (L. Fengjun, Trans.). People's Posts and Telecommunications Press

### **The Internal Structure of CPU**

A CPU consists of registers, controllers, arithmetic units, and clock. **Register** is used to temporarily store commands and data; **Controller** is used to read commands and data from memory to register; **Arithmetic** **Units** is used to perform arithmetic operations on data in the register; And **Clock** is used to send clock puzzle for CPU.

### **The Register**

The keywords (memonics) in assembly languages usually have one-to-one coorespondance with CPU operations. For instance, **mov** and **add** would represent moving and adding data.

For example:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700747997419/212b0f95-8122-4315-bb54-f8fc0f83504c.png align="center")

1 Copy value from memory to register eax

2 add the value in eax with value in memory

3 store the value in eax (the result from above commands) to memory

There are mainly 8 types of registers:

1 **accumulator register**: store data during and after operations

2 **flag register**: store the state of CPU after operations

3 **program counter**: store the address of the next command

4 **base register**: store the starting address of the data stored in memory

5 **index register**: store addresses relative to the base register

6 **general purpose register**: store any data

7 **instruction register**: store internal CPU commands, unaccessible by programs

8 **stack register**: store the starting address of stack

### The Program Counter

When a program starts running, the operating system first copies the program from hard drive to memory, and sets the program counter to the starting address of the program in memory. The CPU controller would retrieve the command at the address indicated by the program counter from the memory and execute the command. When CPU finishes executing one line, the program counter increases by 1 automatically, so the CPU retrieves and executes the next line.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700831033213/d8be09d4-289b-45a7-a9ad-4a738b24c97f.png align="center")

(Yazawa, 2015, Fig 1-4, p 11)

The program counter can be set by programs. This is exactly how conditional branches and loops work. The **jump** command can set the program counter to a specific address with a given condition

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700831053734/32dba82a-ad97-4913-9999-4e0a5839d13a.png align="center")

(Yazawa, 2015, Fig 1-5, p 12)

The condition of **jump** command can be whether the value stored in the accumulator register is equal (not equal/greater than/less than) a certain value, or whether it is greater than/less than zero. The **flag register** holds the status of the accumulator register's value - whether it is positive, negative, or zero. The flag register is updated after every CPU operation. Comparing whether a value is greater than/less than zero is achieved through checking the corresponding bit in flag register. The comparison between two non-zero values **x** and **y** is implemented internally through first calculating (x - y), and then checking whether the flag register indicates positive/negative/zero for this operation.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700831582815/15891160-5ae8-478f-90c1-69964ce7cb52.png align="center")

(Yazawa, 2015, Fig 1-6, p 13)

### Function Call

Function call cannot be implemented using only **jump** command, since it requires jumping to the starting address of the function, and then jumping back to the address of function caller when the function is returned. We need to know simultaneously the address where the function starts, and the address of the program that calls the function.

Function call is implemented through **call** and **return** commands. The call command stores the the address of the function caller in a specific region of the memory called **stack**, before changing the program counter to the function's sarting address. After finishing executing the function body, the return command sets the program counter to the address stored previously in stack.

The stack acts as a temporary storage area for these return addresses, ensuring that the program can maintain a record of multiple function calls and returns, allowing for the correct sequence of operations within the program flow.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700833027288/22dfcba1-802f-4d10-8233-c4e73c31b7ba.png align="center")

(Yazawa, 2015, Fig 1-8, p 16)

### Implement Arrays with Address and Index

An array is a data structure consisting of elements with same lengthes stored in a consecutive partition in memory. The **base register** and **index register** can create specific partitions in the memory and implement array. The base register stores the starting address of the array, and makes the index register changing in the range between starting address and ending address. The index register works similarly to the indexing operation in advance languages. The array index to be accessed will be (value of base register + value of index register)

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1700833556787/0b372b2c-60f3-4947-8d26-2ce02148769e.png align="center")

(Yazawa, 2015, Fig 1-9, p 18)