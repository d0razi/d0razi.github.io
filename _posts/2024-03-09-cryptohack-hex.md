---
title: "[Cryptohack] Hex"
author: d0razi
date: 2024-03-09 21:22
categories: [InfoSec, Crypto]
tags: [Cryptohack, Intro]
image: /assets/img/media/banner/cryptohack.png
use_math: true
---

# 문제

우리가 암호화할 때, 결과적인 암호문은 일반적으로 출력 가능한 ASCII 문자가 아닌 바이트로 되어있습니다. 암호화된 데이터를 공유하려면 암호화된 데이터를 사용하기 쉽고 휴대하기 쉬운 것으로 인코딩하는 것이 일반적입니다.

16진수는 ASCII 문자열을 나타내는 방식으로 사용될 수 있습니다. 먼저 각 문자는 ASCII 코드 표에 따라 순서 번호로 변환됩니다(이전 문제에서처럼). 그런 다음 10진수는 16진수로 알려진 16진수로 변환됩니다. 숫자들은 하나의 긴 16진수 문자열로 결합될 수 있다.

아래에는 16진수 문자열로 인코딩된 플래그가 포함되어 있습니다. 이를 바이트로 다시 디코딩하여 플래그를 가져옵니다.

`63727970746f7b596f755f77696c6c5f62655f776f726b696e675f776974685f6865785f737472696e67735f615f6c6f747d`

> Python에서는 bytes.fromhex() 함수를 사용하여 16진수를 바이트로 변환할 수 있습니다. .hex() 인스턴스 메서드는 바이트 문자열에서 호출되어 16진수 표현을 얻을 수 있습니다.
{: .prompt-info}

# 풀이

문제에서 주어진 16진수값을 bytes.fromhex()함수를 사용해서 바이트로 변환해서 출력했습니다.

```python
flag = "63727970746f7b596f755f77696c6c5f62655f776f726b696e675f776974685f6865785f737472696e67735f615f6c6f747d"
print(bytes.fromhex(flag))
```