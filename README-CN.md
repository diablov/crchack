**crchack** is a public domain tool to force CRC checksums of input messages to
arbitrarily chosen values. The main advantage over existing CRC alteration
tools is the ability to obtain the target checksum by changing non-contiguous
input bits. Furthermore, crchack supports *all* CRC algorithms including custom
CRC parameters. crchack is also an arbitrary-precision CRC calculator.

- [Usage](#usage)
- [Examples](#examples)
- [CRC algorithms](#crc-algorithms)
- [How it works?](#how-it-works)
- [Use cases](#use-cases)


# Usage

```
usage: ./crchack [options] file [target_checksum]
如果实在不想写临时文件，可以用管道操作，用 - 代替标准输入
options:
  -o pos    输入串的可改变的byte.bit位置    byte.bit position of mutable input bits
  -O pos    从输入串尾部的位置偏移    position offset from the end of the input
  -b l:r:s  明确位置bits,l..r,step为s    specify bits at positions l..r with step s
  -h        show this help
  -v        verbose mode

CRC 参数 (默认: CRC-32):
  -p poly   生成多项式    generator polynomial    
  -w size   CRC寄存器大小（bits）    register size in bits
  -i init   寄存器初始值        initial register value
  -x xor    最终寄存器XOR掩码    final register XOR mask
  -r        反转输入字节    reverse input bytes
  -R        反转最终输出    reverse final register
```
输入信息从**file**读入，补丁后信息输出到stdout。默认，crchack追加4个bytes到输入（新消息的CRC-32值为目标值）。
其他CRC算法可以使用CRC参数定义。如果**target_checksum**没有给的话，crchack计算并打印输入消息的CRC值。

# Examples

```
[crchack]$ echo "hello" > foo
[crchack]$ ./crchack foo                   #计算crc值
363a3020
[crchack]$ ./crchack foo 42424242 > bar    #输入foo文件，目标crc：42424242，输出写到bar文件
[crchack]$ ./crchack bar                   #校验
42424242
[crchack]$ xxd bar                         #查看bar文件
00000000: 6865 6c6c 6f0a d2d1 eb7a                 hello....z

[crchack]$ echo "foobar" | ./crchack - DEADBEEF | ./crchack -    #使用 - 代替文件作为输入
deadbeef

[crchack]$ echo "PING 1234" | ./crchack -
29092540
```

## -o/-b
```
[crchack]$ echo "PING XXXX" | ./crchack -o5 - 29092540
PING 1234
```
为了修正不连续的输入消息，指定可变bits位可以使用`-b start:end:step`开关，类似PYthon风格的切片，以`start`位（含）开始，`end`位（不含）结束，连续的`step`间隔位。
如果为空，`start`为消息头， `end`为消息尾，`step`等于1，选择所有消息。
`-b 4:`选择**byte**位置4开始的所有字节。
注意 `-b 4` 没有分号，只选择第32bit。

```
[crchack]$ echo "aXbXXcXd" | ./crchack -b1:2 -b3:5 -b6:7 - cafebabe | xxd
00000000: 61d6 6298 f763 4d64 0a                   a.b..cMd.
[crchack]$ echo -e "a\xD6b\x98\xF7c\x4Dd" | ./crchack -
cafebabe

[crchack]$ echo "1234PQPQ" | ./crchack -b 4: - 12345678
1234u>|7
[crchack]$ echo "1234PQPQ" | ./crchack -b :4 - 12345678
_MLPPQPQ
[crchack]$ echo "1234u>|7" | ./crchack - && echo "_MLPPQPQ" | ./crchack -
12345678
12345678
```
Byte位后面可选跟着点分割的**bit**位。例如 `-b0.32`, `-b2.16` 和 `-b4.0` 都选择同样的第32nd bit。
在Byte中，bits从最低有效位least significant bit (0) 到最高有效位most significant bit(7)计数。
 “ - ” 位置视为输入位置从尾部的偏移。
内置的解析器支持`0x`开头的16进制数，以及基本的算术操作`+-*/`。
最后，`end`也可以定义成相对于 `start`，使用一元操作符`+`。

```
[crchack]$ python -c 'print("A"*32)' | ./crchack -b "0.5:+.8*32:.8" - 1337c0de
AAAaAaaaaaAAAaAaAaAaAaaAaAaaAAaA
[crchack]$ echo "AAAaAaaaaaAAAaAaAaAaAaaAaAaaAAaA" | ./crchack -
1337c0de

[crchack]$ echo "1234567654321" | ./crchack -b .0:-1:1 -b .1:-1:1 -b .2:-1:1 - baadf00d
0713715377223
[crchack]$ echo "0713715377223" | ./crchack -
baadf00d
```
如果给的可变bits不足，就不可能得到目标CRC值。一般而言，需要提供至少**w** bits， **w**是CRC寄存器的宽度，比如CRC-32需要32bits。

# CRC 算法：-w/-p/-rR/-i/-x
crchack对所有正常的CRCs都使用，包括标准化的CRCs。其他CRC功能可以通过命令行参数传递CRC参数指定。

```
[crchack]$ printf "123456789" > msg
[crchack]$ ./crchack -w8 -p7 msg                                     # CRC-8
f4
[crchack]$ ./crchack -w16 -p8005 -rR msg                             # CRC-16
bb3d
[crchack]$ ./crchack -w16 -p8005 -iffff -rR msg                      # MODBUS
4b37
[crchack]$ ./crchack -w32 -p04c11db7 -iffffffff -xffffffff -rR msg   # CRC-32
cbf43926
```

[CRC RevEng](http://reveng.sourceforge.net/) (by Greg Cook) includes a
comprehensive [catalogue](http://reveng.sourceforge.net/crc-catalogue/) of
cyclic redundancy check algorithms and their parameters. [check.sh](check.sh)
illustrates how to convert the CRC catalogue entries to crchack CLI flags.


# How it works?

CRC is often described as a linear function in the literature. However, CRC
implementations used in practice often differ from the theoretical definition
and satisfy only a *weak* linearity property (`^` denotes XOR operator):

    CRC(x ^ y ^ z) = CRC(x) ^ CRC(y) ^ CRC(z), for |x| = |y| = |z|

crchack applies this rule to construct a system of linear equations such that
the solution is a set of input bits which inverted yield the target checksum.

The intuition is that each input bit flip causes a fixed difference in the
resulting checksum (independent of the values of the neighbouring bits). This,
in addition to knowing the required difference, produces a system of linear
equations over [GF(2)](https://en.wikipedia.org/wiki/Finite_field). crchack
expresses the system in a matrix form and solves it with the Gauss-Jordan
elimination algorithm.

Notice that the CRC function is treated as a "black box", i.e., the internal
CRC parameters are used only for evaluation. Therefore, the same approach is
applicable to any function that satisfies the weak linearity property.


# Use cases

So why would someone want to forge CRC checksums? Because `CRC32("It'S coOL To
wRitE meSSAGes LikE this :DddD") == 0xDEADBEEF`.

In a more serious sense, there exist a bunch of firmwares, protocols, file
formats and standards that utilize CRCs. Hacking these may involve, e.g.,
modification of data so that the original CRC checksum remains intact. One
interesting possibility is the ability to store arbitrary data (e.g., binary
code) to checksum fields in packet and file headers.
