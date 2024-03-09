---
title: "[Cryptohack] Bytes and Big Integers"
author: d0razi
date: 2024-03-09 21:30
categories: [InfoSec, Crypto]
tags: [Cryptohack]
image: /assets/img/media/banner/cryptohack.png
---


[CryptoHack – Introduction to CryptoHack - Bytes and Big Integers](https://cryptohack.org/courses/intro/enc4/)

RSA와 같은 암호 시스템은 숫자로 작동하지만 메세지는 문자로 구성된다. 우리가 어떻게 수학적 연산이 적용되도록 메세지를 숫자로 변환해야 할까?

가장 일반적인 방법은 메세지의 ordinal(서수) 바이트를 가져와 16진수로 변환하고 연결하는 것이다. 이는 base-16/hexadecimal number로 해석될 수 있으며, base-10/decimal로 표현할 수 있다.

설명:

> message: HELLO
> 
> ascii bytes: [72, 69, 76, 76, 79]
> 
> hex bytes: [0x48, 0x45, 0x4c, 0x4c, 0x4f]
> 
> base-16: 0x48454c4c4f
> 
> base-10: 310400273487
> 

> Python의 PyCryptodome 라이브러리는 bytes_to_long() 및 long_to_bytes() 메서드로 이를 구현합니다. 먼저 PyCryptodome을 설치하고 from Crypto.Util.number import *로 가져와야 합니다
{: .prompt-info}

다음 정수를 다시 메시지로 변환합니다.

`11515195063862318899931685488813747395775516287289682636499965282714637259206269`

# 풀이

**long_to_bytes()함수**

**`long_to_bytes()`** 함수는 첫 번째 매개변수로 변환할 정수(long) 값을 받으며, 두 번째 매개변수로는 변환된 바이트 값의 길이를 나타내는 정수(length)를 받습니다. 만약 두 번째 매개변수를 지정하지 않으면 최소 필요한 바이트 수만큼 반환합니다.

예를 들어, **`long_to_bytes(2835, 2)`**는 2835를 2바이트 크기의 바이트 값으로 변환하며, 결과값은 **`b'\x0b\x13'`**이 됩니다.

```python
from Crypto.Util.number import *

flag = 11515195063862318899931685488813747395775516287289682636499965282714637259206269
print(long_to_bytes(flag))
```