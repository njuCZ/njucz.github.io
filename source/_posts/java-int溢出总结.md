title: java int溢出总结
date: 2017-08-16 11:03:46
tags: [java]
---
### java溢出初体验
java的int是32位有符号整数类型，其最大值是0x7fffffff,最小值则是0x80000000。即int表示的数的范围是-2147483648 ～ 2147483647之间。当int类型的运算结果超出了这个范围时则发生了溢出，而且不会有任何异常抛出。

```
System.out.println(Integer.MAX_VALUE + 1);
System.out.println(Integer.MIN_VALUE - 1);

-2147483648
2147483647
```
观察上述语句可发现：
- 当结果超出了integer的最大表示范围时，结果变成了负数
- 当结果超出了integer的最小表示范围时，结果变成了正数

那么上述的结论究竟对不对呢？是不是只要两个正数的操作（加法或乘法）发生溢出时结果就会变成负数呢？

### java溢出机制
```
int num = 907654321;
System.out.println(num * 16);
System.out.println(num * 16L);

1637567248
14522469136
```
通过上述例子居然发现int和int类型相乘的结果虽然发生了溢出，但是结果并不是负数，还是一个正数。那么java在处理溢出的逻辑到底是怎样的？又是怎么得到这个结果的呢？

我们先看看下面的例子:
```
int num = 907654321;
System.out.println(Integer.toHexString(num));
System.out.println(Long.toHexString(num * 16L));
System.out.println((int)((num * 16L) & 0xffffffff));

3619b4b1
3619b4b10
1637567248
```
大家有没有发现神奇的一点： `(int)((num * 16L) & 0xffffffff) == num * 16`

在Java Language Specifictionz中所述([JSL 15.7.1](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.17.1))

> If an integer multiplication overflows, then the result is the low-order bits of the mathematical product as represented in some sufficiently large two's-complement format. As a result, if overflow occurs, then the sign of the result may not be the same as the sign of the mathematical product of the two operand values.

也就是说int型整数相乘，结果只会保留低32位，高位会抛弃掉。所以num * 16L的值与0xffffffff做位与操作（即取后32位）就可以得到实际运算的结果了。

那么下面的这些语句有没有bug呢？
```
long MonthNanoSeconds1 = 30 * 24 * 3600 * 1000 * 1000;
long MonthNanoSeconds2 = 30 * 24 * 3600 * 1000 * 1000L;
```
将一个大的数赋值给long型变量，好像没什么问题。但是要注意到java是先计算右值，再赋值给long变量的。在计算右值的过程中（int型相乘）发生溢出，然后将溢出后截断的值赋给变量。而在第二行语句中由于java的运算规则从左到右，再与最后一个long型的1000相乘之前就已经溢出，所以结果也不对。正确的方式应该如下：
```
long MonthNanoSeconds = 30L * 24 * 3600 * 1000 * 1000;
```

### java 类型提升
*int型整数相乘并不会进行类型提升(type promotion)，在[widening conversion(JSL 5.1.2)](https://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.2)写到*：
> If any of the operands is of a reference type,  [unboxing conversion(JSL 5.1.8)](https://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.8) is performed. Then:
> If either operand is of type double, the other is converted to double.
> Otherwise, if either operand is of type float, the other is converted to float.
> Otherwise, if either operand is of type long, the other is converted to long.
> Otherwise, both operands are converted to type int.

所以下述语句并不会发生溢出：
```
byte a = 40;
byte b = 50;
byte c = 100;
int d = a * b / c;
```
虽然40 * 50 = 2000已经超过了byte的范围，但是java compile在计算a * b之前已经都将它们转换成了int类型了，所以并不会有编译问题。

### prevent overflow
- 使用类型提升，int型换成long型，long型转换BigDecimal
- 使用Math.addExact和Math.multiplyExact，溢出时会抛出异常
- 可以先用最大最小数判断一下，比如`if(b > Integer.MAX_VALUE / a)`
