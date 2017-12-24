#### 问题描述
tcp协议头中有seq和ack_seq两个字段，分别代表序列号和确认号。tcp协议通过序列号标识发送的报文段。seq的类型是__u32，当超过__u32的最大值时，会回绕到0。

一个tcp流的初始序列号（ISN）并不是从0开始的，而是采用一定的随机算法产生的，因此ISN可能很大（比如(2^32-10)），因此同一个tcp流的seq号可能会回绕到0。而我们tcp对于丢包和乱序等问题的判断都是依赖于序列号大小比较的。此时就出现了所谓的tcp序列号回绕（sequence wraparound）问题。

#### 内核解决办法
内核中给出的序列号(解决序列号回绕问题)判断解决方案十分简洁：
```c
/*
* The next routines deal with comparing 32 bit unsigned ints
* and worry about wraparound (automatic with unsigned arithmetic).
*/
static inline int before(__u32 seq1, __u32 seq2)
{
return (__s32)(seq1-seq2) < 0;
}
#define after(seq2, seq1) before(seq1, seq2)
```
#### 原理
为什么（__s32）(seq1-seq2)<0就可以判断seq1

为了方便说明，我们以unsigned char和char为例来说明：

假设seq1=255，seq2=1（发生了回绕）。

seq1 = 1111 1111 seq2 = 0000 0001

我们希望比较结果是seq1
```
 seq1 - seq2=
 1111 1111
-0000 0001
-----------
 1111 1110
```
由于我们将结果转化成了有符号数，由于最高位是1，因此结果是一个负数，负数的绝对值为
```
 0000 0001 + 1 = 0000 0010 = 2
```
因此seq1 - seq2 < 0

#### 注意：
如果seq2=128的话，我们会发现：
```
 seq1 - seq2=
 1111 1111
-1000 0000
-----------
 0111 1111
```
此时结果尤为正了，判断的结果是seq1>seq2。因此，上述算法正确的前提是，回绕后的增量小于2^(n-1)-1。

由于tcp序列号用的32位无符号数，因此可以支持的回绕幅度是2^31-1，满足要求了。
