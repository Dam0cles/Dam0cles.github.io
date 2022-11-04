---
layout: post
title: StarCTF2019-PWN-BlindPwn
tags: [pwn]
excerpt: 年轻人的第一次盲栈溢出
---

## 信息收集
程序提供了相关描述：

- [x] 程序为64位
- [x] 提供了libc版本(libc-2.23.so: ELF 64-bit LSB shared object)
- [x] 未开启PIE、full relro及canary

模拟程序远程环境：

![](/assets/img/starCTF/blindpwn/1.png)


探测溢出长度：

![](/assets/img/starCTF/blindpwn/2.png)

![](/assets/img/starCTF/blindpwn/3.png)

看到当offset=41时没有输出"GoodBye",说明返回地址被篡改，说明溢出长度为40。

接下来可以溢出一位，看看能达到什么效果：

![](/assets/img/starCTF/blindpwn/8.png)

👆当溢出一位为0x5或者0x76时，程序正常运行，推测0x5或者0x76为原来的数据。


![](/assets/img/starCTF/blindpwn/5.png)

![](/assets/img/starCTF/blindpwn/6.png)

![](/assets/img/starCTF/blindpwn/7.png)

👆当溢出一位为0x0a/0x0f/0x14时，程序会打印一些数据。仔细观察，发现打印的数据量均为0x100,且会输出0x00字符，说明:
1. 使用的输出函数不为puts、printf，可能为write;
2. 输入函数接收0x100后，寄存器未被清空，导致输出函数接收0x100个字符;
3. 输出的内容中包含地址0x4005XX~0x4007XX，推测为程序代码段地址
验证3. ：
![](/assets/img/starCTF/blindpwn/13.png)

![](/assets/img/starCTF/blindpwn/12.png)

👆当输入地址为0x400520时，程序输出0x100字节。0x20!= 0x0a/0x0f/0x14，推测是第二位0x40XX20与它们三者不同。

👆当输入地址为0x400570时，程序打印了两次"Welcome to this blind pwn!\n"且允许用户继续输入，所以主函数地址为0x400570。

自己在本地写一个使用write函数进行输出的程序，打印反编译结果：

![](/assets/img/starCTF/blindpwn/9.png)

可以看出，实际上当溢出一位为0x0a，类比上图的0x4011ae处（字节码不同，长度不一定为7），此时rdx寄存器的值继承于程序原执行流中输入函数赋于的0x100,所以输出了"GoodBye!"及其后面加起来共0x100字节的data段数据；

当溢出一位为0x0f，类比上图的0x4011b5，此时rdx寄存器为0x100，rsi为栈地址（提供给输入函数的输入缓冲区地址），所以输出了栈上的0x100数据；

溢出一位为0x14类比上图的4011ba，效果与溢出一位为0x0f一样。

由于程序为64为，接下来需要找到pop rdi;ret和pop rsi;ret的Gadget指令地址，才能在构造ROP链时构造函数参数。

我们知道一个程序存在__libc_csu_init函数。这个函数的末尾存在ret2csu中必要的Gadget：

![](/assets/img/starCTF/blindpwn/10.png)

pop r14;pop r15;ret可以变形为pop rsi;pop r15;ret, pop r15;ret可以变形为pop rdi;ret:

![](/assets/img/starCTF/blindpwn/11.png)


函数__libc_csu_init通常位于程序员编写的函数之下，在上面泄露信息中了解到程序代码段地址为0x4005XX~0X4007XX,可以从0x400700开始进行一位爆破:

![](/assets/img/starCTF/blindpwn/14.png)

![](/assets/img/starCTF/blindpwn/15.png)

👆此处验证pop rbx~pop r15;ret的gadget地址，因为单单验证一个pop XXX;ret不能确定其就是pop rdi;ret。 可以看到当溢出地址为0x40077a时，接收6个pop后成功执行主函数，说明0x40077a为 csu gadget，根据字节码个数可以算出pop rsi起始地址为csu gadget+7，pop rdi起始地址为csu gadget+9。


有可以打印函数的地址(0x400520)，有pop rdi与pop rsi的地址(rdx返回时固定为0x100)，就可以进行任意地址读。使用[Confused environment write的方式二](https://dam0cles.github.io/2022/05/05/confused_envrionment_write.html#method_2),获得.dynamic、.dynsym及.dynstr地址:

![](/assets/img/starCTF/blindpwn/16.png)

![](/assets/img/starCTF/blindpwn/17.png)

获取.dynsym、.dynstr及write的.got.plt与上述流程类似。并得知0x400520为write@plt。


## 构造getshell
第一轮先泄露write函数的.got.plt地址以获得libc基址，第二轮构造system('/bin/sh\x00'):

![](/assets/img/starCTF/blindpwn/19.png)

![](/assets/img/starCTF/blindpwn/20.png)

## 后记
本篇学习我在开始前没有看过BROP相关资料，效率/手法可能不如BROP高明。