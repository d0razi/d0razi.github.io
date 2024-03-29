---
title: "[Cryptohack] XOR Starter"
author: d0razi
date: 2024-03-15 22:22
categories: [InfoSec, Crypto]
tags: [Cryptohack, Intro]
image: /assets/img/media/banner/cryptohack.png
use_math: true
---
# 문제 

XOR은 비트가 같으면 0을 반환하고 그렇지 않으면 1을 반하는 비트 연산자이다.

교과서에서 XOR 연산자는 ⊕로 표시되지만, 대부분의 과제와 프로그래밍 언어에서는 caret `^`이 대신 사용된다.

더 긴 이진수의 경우 비트별 XOR: 0110 ^ 1010 = 1100이다. 우리는 먼저 정수를 십진법에서 이진법으로 변환함으로써 정수를 XOR할 수 있다. 먼저 각 문자를 유니코드 문자를 나타내는 정수로 변환하여 문자열을 XOR 할 수 있다.

문자열 "label"이 주어지면 정수 13으로 각 문자를 XOR 한다. 이런 정수를 다시 문자열로 변환하고 플래그를 crypto{new_string}으로 제출한다.
> python pwntools 라이브러리에는 다양한 유형과 길이의 데이터를 XOR할 수 있는 편리한 xor() 기능이 있다. 그러나 먼저, 이것을 해결하기 위해 스스로 함수를 구현하기를 권한다.
{: .prompt-info}

# 풀이

```python
fin_flag = "crypto{"
flag = b"label"

for i in flag:
    fin_flag += chr(i ^ 13)

print(fin_flag + "}")
```