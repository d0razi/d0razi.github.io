---
title: "[Cryptohack] Favourite Byte"
author: d0razi
date: 2024-03-19 21:22
categories: [InfoSec, Crypto]
tags: [Cryptohack, Intro]
image: /assets/img/media/banner/cryptohack.png
use_math: true
---

# 문제

다음 몇 과제에서는 이전에 배운 내용을 사용하여 XOR 퍼즐을 더 풀어볼 것이다.

단일 바이트로 XOR을 사용하여 데이터를 숨겼지만 그 바이트는 비밀이다.

16진수부터 디코딩하는 것을 잊지 마십시오.

`73626960647f6b206821204f21254f7d694f7624662065622127234f726927756d`

# 풀이

먼저 bytes.fromhex() 함수를 사용해서 바이트로 변경해줍니다.

그리고 단일 바이트로 XOR을 숨겼다고 했으니 브루트 포싱으로 찾았습니다.

```python
flag = bytes.fromhex("73626960647f6b206821204f21254f7d694f7624662065622127234f726927756d")

for i in range(1,100):
    result = "".join(chr(j ^ i) for j in flag)
    if ("crypto" in result):
        print(result)
```