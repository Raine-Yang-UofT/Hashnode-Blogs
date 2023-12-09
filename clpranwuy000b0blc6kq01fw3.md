---
title: "Learning Note: Floating-Point Numbers and Round-off Error"
datePublished: Mon Dec 04 2023 19:18:23 GMT+0000 (Coordinated Universal Time)
cuid: clpranwuy000b0blc6kq01fw3
slug: learning-note-floating-point-numbers-and-round-off-error
tags: regular-expressions, floating-point-arithmetic, excess

---

This article is a summary of Chapter 3 of:

Yazawa, H. (2015). *程序是怎么跑起来的\[How Program Works\]* (L. Fengjun, Trans.). People's Posts and Telecommunications Press

### Representing Fractions with Binary

The convertion between binary and decimal for fraction numbers works the same as that for integers: multipling each digit by its (base ^ place). The first digit after decimal point would multiply by 2 ^ -1, and the second digit multiplies by 2 ^ -2, and so on. For example, Fig 1 shows how to convert binary number 1011.0011 to decimal.

Fig. 1

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701350963028/1d7ca703-3a97-4f44-897a-16f9f493d8e3.png align="center")

(Yazama, 2015, Fig 3-2, p46)

### Round-off Error for Converting Decimal Fractions

Not all demical numbers can be precisely converted to binary. For example, binary numbers from 0.0000 - 0.1111 can only represent linear combinations of 0.5, 0.25, 0.125, 0.0625 (Fig 2).

Fig 2.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701351676227/fc500d06-8d8d-4acf-a2c8-733003228024.png align="center")

(Yazama, 2015, Table 3-1, p47)

Some decimal numbers, like 0.1, cannot be represented in finite binary numbers at all. 0.1 would be converted to 0.00011001100... with an infinite repitition of 1100. This is similar to what happens if we represent 1/3 in decimal. Since computers can only hold finite places, the binary representation of 0.1 would only be an approximation.

In cases when the round-off error must be avoided, the first way is to convert float to integers. For example, to calculate 0.1 + 0.1 + 0.1, we can first calculate 1 + 1 + 1, and divide the result by 10. In addition, **BCD (Binary Coded Decimal)** is another method to represent decimals by binary that does not involve round-off error. BCD uses four binary places to represent one decimal place

### Floating-point Number

**Floating-point number** is the common method to represent fractions in computers. It consisits of **sign, mantissa, base, and exponent**. The base is not really stores since it is always 2.

Fig 3

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701352072196/35d0eeb2-d086-44b3-9b16-94f2a8d12d48.png align="center")

(Yazama, 2015, Fig 3-3, p49)

In **double-precision float number** (called double type in most programming languages), the float number has 11 exponent places and 52 mantissa places. In **single-precision float number** (called float type), the float number consists of 8 enponent places and 23 mantissa places.

The **sign place**, like integers, represents positive number with 0 and negative number with 1. The expressions for mantissa and exponent will be discussed in the next sections.

### Regular Expression and EXCESS System

The mantissa is expressed in **regular expression**. A regular expression stands for a normalized expression to represent data. For example, when expressing 0.1101 in float number, we can write eith 0.1101X2^0, 11.01X10^-2, 1.101X10^-1. In regular expression for float numbers, we always fix the digit before the decimal point to be 1, that is, the 1.101X10^-1 form. This is achieved through logical right shifting or left shifting of the fraction number.

Fig 4

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701354041765/d14f98c4-3d88-4b05-874c-864f12f65a03.png align="center")

(Yazama, 2015, Fig 3-6, p52)

For example, to get the regular expression for the mantissa of 1011.0011. We first right shift the integer part so the integer part becomes 1. Then we keep the portion after the decimal point, which turns out to be the final regular expression.

The **EXCESS system** used to represent the exponents is a way to represent positive and negative numbers without a sign place. This is achieved through defining 0 as the middle of the number range. For example, for single-precision float number with 8 exponent places. The maximim number that can be expressed is 11111111 = 255. Zero is defined as 01111111 = 127. Values above that will be positive numbers and values below are negative numbers. As a result, 8 places can represent -127 to 128 exponents.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701705424826/6c6a81a1-72bc-45f9-8789-ef55e5512268.png align="center")

(Yazama, 2015, Table 3-2, p53)

To summarize what we described above through an example, let's see how to convert 0.75 to floating-point binary number. The sign place is 0 since 0.75 is a positive number. Converting 0.75 to binary would be 0.11, which in regular expression would be 1.1 X 2^-1.

The matissa is 10000000000000000000000 (omitting the leading 1, and make sure the matissa has 23 places)

The exponent -1 in EXCESS system would be zero (011111111) minus 1, which is 01111110.

Thus, the resulting float number is 0-01111110-10000000000000000000000

### Binary and Hexadecimal

**Hexadecimal** is a number system with 16 digits (0-9, A-F). In C language, adding 0x before a number would indicate hexidecimal. One place in hexadecimal can represent exactly four places in binary.