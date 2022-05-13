---
layout: post
title: 24/7CTF-PWN-Less confused environment write
tags: [pwn]
excerpt: 年轻人的第二次盲字符串格式化溢出
---


![](/assets/img/247ctf/pwn/less_confused_environment_write/logo.jpg)

## 信息收集
根据"局部性原理"，推测程序与[Confused environment write](https://dam0cles.github.io/2022/05/05/confused_envrionment_write.html)存在相同信息：

- [x] 程序为32位
- [x] libc版本为libc6-i386_2.27-3ubuntu1_amd64
- [x] 未开启PIE、full relro及canary
- [x] 存在字符串格式化漏洞

连上程序环境进行观察：
![](/assets/img/247ctf/pwn/less_confused_environment_write/1.png)

由上图得知：
	1) 程序为32位，且未开启PIE，代码段加载地址为默认0x08048000
	2) 字符串格式化溢出漏洞偏移为11
	3) 程序执行一次printf后结束，没有循环


使用[Confused environment write的方式二](https://dam0cles.github.io/2022/05/05/confused_envrionment_write.html#method_2),获得.dynamic、.dynsym及.dynstr地址:

![](/assets/img/247ctf/pwn/less_confused_environment_write/2.png)
<span id="dynamic_layout"></span>
![](/assets/img/247ctf/pwn/less_confused_environment_write/3.png)

	.dynamic address: 0x08049f0c
	.dynstr address : 0x0804828c
	.dynsym address : 0x080481cc

根据Confused environment write的[关系图](https://dam0cles.github.io/2022/05/05/confused_envrionment_write.html#all_rela)
获得.got.plt函数布局:

![](/assets/img/247ctf/pwn/less_confused_environment_write/4.png)

![](/assets/img/247ctf/pwn/less_confused_environment_write/5.png)

自此，相关信息获取完毕。

## 构造getshell
由于程序只执行一次，需要使程序可以循环。思路有以下三种：
- [ ] 修改退出程序前打印"Goodbye!"的libc库函数[.got.plt]为main函数地址
- [ ] 修改.fini_array[0]为main函数地址
- [x] 修改malloc_hook，并同时触发printf内执行malloc的逻辑

第一种思路不成功，因为打印函数的[.got.plt]被修改后，每次碰到该打印函数都会跳到main，无法继续执行。

第二种思路，通过[.dynamic](#dynamic_layout)中ELF Segment Types=0x1a找到.fini_array section的地址(0x08049f08)。[.fini_array保存了程序退出时要执行的函数](http://blog.k3170makan.com/2018/10/introduction-to-elf-format-part-v.html)，修改其中内容可以使程序再次执行main函数。经过多次尝试，发现无法修改0x08049f08上的内容，推测是程序开了partial relro的原因。

第三种思路：根据[介绍](https://github.com/Naetw/CTF-pwn-tips#use-printf-to-trigger-malloc-and-free)，一次输入超过65536个字节到printf（32位）会执行printf内部的malloc，只需要在格式化字符串加入"%65536$c"即可触发。

修改malloc_hook需要知道libc加载地址，但是没有循环，没办法泄露信息。通过多次远程连接程序，发现libc基址虽然随机，但都是0xf7dXX000，即地址中只有一个字节随机。所以可以硬编码，有1/256的概率碰撞出正确的libc加载地址。（概率较"高"）

getshell流程：
	1）修改malloc_hook为main函数地址，并触发printf内部malloc；
	2）(碰撞libc地址成功后)程序执行malloc，发现malloc_hook!=0，跳转执行main函数；
	3）修改printf@.got.plt为system@libc，并触发printf内部malloc，跳转执行main函数；
	4）输入"/bin/sh"，getshell；

## getshell

![](/assets/img/247ctf/pwn/less_confused_environment_write/6.png)

![](/assets/img/247ctf/pwn/less_confused_environment_write/7.png)