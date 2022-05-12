---
layout: post
title: 24/7CTF-PWN-Less confused environment write
tags: [pwn]
excerpt: 年轻人的第二次盲字符串格式化溢出
---


![](/assets/img/247ctf/pwn/less_confused_environment_write/logo.jpg)

## 信息收集
根据"局部性原理"，推测程序与[Confused environment write](https://dam0cles.github.io/2022/05/05/confused_envrionment_write.html/#leak_got_plt)存在相同信息：

- [x] 存在字符串格式化漏洞
- [x] libc版本为libc6-i386_2.27-3ubuntu1_amd64
- [x] 未开启PIE、full relro及canary

