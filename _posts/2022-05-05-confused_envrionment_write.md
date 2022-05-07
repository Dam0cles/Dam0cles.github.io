# 24/7CTF-PWN-Confused environment write
## 信息收集
题目是一个PWN题，但是没有给二进制文件，只给了题目部署IP和端口。连上题目环境进行观察：
![](/assets/img/247ctf/pwn/confused_environment_write/1.png)
题目打印了一些常规信息，并请求用户输入数据。既然是PWN题，就先尝试输入超长字符串看看会不会栈溢出：
![](/assets/img/247ctf/pwn/confused_environment_write/2.png)

![](/assets/img/247ctf/pwn/confused_environment_write/3.png)

由上图可以发现，程序限制了单次输入字符串的长度（64字节），保证进程不会栈溢出。
无法栈溢出，试试字符串格式化溢出：
![](/assets/img/247ctf/pwn/confused_environment_write/4.png)

根据红框中的信息，可以明确知道这题存在字符串格式化溢出。而且根据上图中的0x08048787，可以推断此程序为32位，且没有开启PIE保护机制。
```
热知识1: 在默认及未开启PIE的情况下，x86平台下二进制可执行文件的代码段加载地址为0x08048000，x86-64的则为0x40000:
```
![](/assets/img/247ctf/pwn/confused_environment_write/5.png)


为了便于了解32位程序的内存布局，我自己写了一个类似功能的c代码并运行起来，观察其内存布局：
![](/assets/img/247ctf/pwn/confused_environment_write/6.png)

可以看到如果程序以默认地址0x08048000加载代码段的话，那么0x804a000很可能如上图中显示可读可写,且这个段的开头是.got.plt。接下来尝试读出0x0804a000中的内容，观察是否确实为.got.plt。
```
热知识2：未开启relro或者开启partial relro的情况下，.got.plt才可以写。
```
要读出0x0804a000的内容，首先需要知道输入字符串在栈中的地址与call printf@plt时偏移多少。这个步骤比较容易，就是从1开始不断加上去，构造AAAA%1$lx..%2$lx..%3$lx…，直到打印出41414141（"AAAA"的ascii表示）。偏移为11：
![](/assets/img/247ctf/pwn/confused_environment_write/7.png)

<span id="leak_got_plt">leak_got_plt</span>
之前推测0x0804a000为可读可写段，既然知道了偏移为11，那么可以读出0x0804a000中的内容。由于0x0804a000中包含字符串截断符’\x00’，所以需要将这个地址放到最后避免影响构造。
构造:
![](/assets/img/247ctf/pwn/confused_environment_write/8.png)

得到结果:
![](/assets/img/247ctf/pwn/confused_environment_write/9.png)

跳过我设置的分隔符”||||”，可以看到第一个泄漏的地址为0x0804XXXX，后面的地址都是0xf7XXXXXX，与.got.plt基本吻合。
```
热知识3：之所以通过上图可以推断与.got.plt基本吻合，是因为32位下的.got.plt存在规律：
前三项分别保存着.dynamic、link_map和_dl_runtime_resolve，从第四项开始是要延迟绑定的libc函数的位置。

	.got.plt layout: 
	  0x0804a000   0x0804a004        0x0804a008        0x0804a00c     0x0804a010    
	| .dynamic   |  link_map  | _dl_runtime_resolve |  libc_func1  |  libc_func2  | ...
```

### 断定1
0x0804a000是.got.plt开头。

### 断定1-遗留问题
	1）根据热知识3，0x0804a00c及其后分别存储着哪些libc函数未明;
	2) 程序是否开启full_relro未明，所以能否覆写.got.plt未明;
	3) 程序使用的libc版本未明;

### 遗留问题解决方案
我们泄露出了.got.plt的部分libc库函数，现在需要推测出它们具体是什么函数。由于题目没有提供二进制可执行文件，所以需要猜测伪代码逻辑：
![](/assets/img/247ctf/pwn/confused_environment_write/10.png)

根据上述伪代码逻辑，做出以下尝试——修改0x0804a00c及其后地址的内容:

1)修改0x0804a00c地址里的数据为一个不可执行的地址(0x1111)

![](/assets/img/247ctf/pwn/confused_environment_write/11.png)

![](/assets/img/247ctf/pwn/confused_environment_write/12.png)

程序功能没有受到显著影响，但是在第二次循环时我刻意去观察了0x0804a00c地址里的数据，确实被更改了，这说明了：
	
	1.此程序只开启了partial relro(甚至no relro)，.got.plt段的内容可以覆写；(遗留问题2)
	2.0x0804a00c不是伪代码逻辑里第7~12行里的任何libc函数，否则程序将报错退出，无法继续循环。

2)修改0x0804a010地址里的数据为一个不可执行的地址

![](/assets/img/247ctf/pwn/confused_environment_write/13.png)

![](/assets/img/247ctf/pwn/confused_environment_write/14.png)

程序在第二次循环输入完成后不再打印后面的内容，说明程序崩溃于伪代码逻辑第10行，说明[0x0804a010]=printf@libc。

得到了printf@libc，需要怎么得到libc版本呢?[libc.blukat.me](https://libc.blukat.me/)根据libc库函数的最后24bits偏移地址，列出所有符合的libc库，可以辅助我们找到正确的libc库版本。根据[泄露.got.plt](#leak_got_plt)时得到的[0x0804a010]，在[libc.blukat.me](https://libc.blukat.me/)里输入printf及我们得到的printf@libc最后24bits：
![](/assets/img/247ctf/pwn/confused_environment_write/15.png)

这里有多个版本可能性，所以还需要更多libc库函数进一步缩小寻找范围。


3)修改0x0804a018地址里的数据为一个不可执行的地址(此处省略0x0804a014)

![](/assets/img/247ctf/pwn/confused_environment_write/16.png)

![](/assets/img/247ctf/pwn/confused_environment_write/17.png)

程序在第二次循环打印完"What's your name again?"后崩溃退出，说明程序崩溃于伪代码逻辑第9行，进一步说明[0x0804a018]=some_input_func@libc。将几个常见的libc库输入函数结合printf一起放入[libc.blukat.me](https://libc.blukat.me/)进行查询，得到结果：
![](/assets/img/247ctf/pwn/confused_environment_write/18.png)

### 断定2
程序使用libc版本为libc6-i386_2.27-3ubuntu1_amd64.so。(遗留问题3)

## 构造getshell
两种思路：

	1) 将[某个libc库函数的.got.plt]修改为one_gadget直接跳过去执行；
	2）将[printf@.got.plt]修改为system@libc，在下次循环时输入"/bin/sh"，执行伪代码逻辑第11行时就会变成system("/bin/sh")；

两种思路都需要先循环一次以泄露libc基址。在思路1）中，我试了libc6-i386_2.27-3ubuntu1_amd64.so中的所有one_gadget都不行，所以只能用思路2）。

## getshell

![](/assets/img/247ctf/pwn/confused_environment_write/19.png)

![](/assets/img/247ctf/pwn/confused_environment_write/20.png)
