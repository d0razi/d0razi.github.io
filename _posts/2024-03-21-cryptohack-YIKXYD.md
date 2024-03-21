---
title: "[Cryptohack] You either know, XOR you don't"
author: d0razi
date: 2024-03-21 18:12
categories: [InfoSec, Crypto]
tags: [Cryptohack, Intro]
image: /assets/img/media/banner/cryptohack.png
---

# 문제

내 비밀키로 암호화 했는데, 당신은 절대 추측할 수 없을 것이다.

> 플래그 형식과 이 과제에 도움이 될 수 있는 방법을 기억하라.
{: .prompt-tip}

`0e0b213f26041e480b26217f27342e175d0e070a3c5b103e2526217f27342e175d0e077e263451150104`

# 풀이

먼저 플래그 형식인 crypto{}로 XOR 키를 찾아보겠습니다. }는 맨 마지막에 나오기 때문에 마지막 바이트랑 연산해주었습니다.

```python
flag_format = "crypto{}"
key = bytes.fromhex('0e0b213f26041e480b26217f27342e175d0e070a3c5b103e2526217f27342e175d0e077e263451150104')
xor_key = ""

for i in range(0,len(flag_format)):
    if i == len(flag_format)-1:
        xor_key += chr(ord(flag_format[i]) ^ key[-1])
    else:
        xor_key += chr(ord(flag_format[i]) ^ key[i])
print(xor_key)
```

```bash
❯ py solve.py
myXORkey
```

XOR키를 구했으니 바이트별로 연산했습니다.

```python
flag_format = "crypto{}"
key = bytes.fromhex('0e0b213f26041e480b26217f27342e175d0e070a3c5b103e2526217f27342e175d0e077e263451150104')
xor_key = ""

for i in range(0,len(flag_format)):
    if i == len(flag_format)-1:
        xor_key += chr(ord(flag_format[i]) ^ key[-1])
    else:
        xor_key += chr(ord(flag_format[i]) ^ key[i])

for i in range(0,len(key)):
    print(chr(ord(xor_key[i%len(xor_key)]) ^ key[i]), end = "")
```
