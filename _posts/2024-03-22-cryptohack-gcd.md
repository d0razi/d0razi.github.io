---
title: "[Cryptohack] Greatest Common Divisor"
author: d0razi
date: 2024-03-22 10:47
categories: [InfoSec, Crypto]
tags: [Cryptohack, MODULAR ARITHMETIC]
image: /assets/img/media/banner/cryptohack.pn
use_math: trueg
---

# 개념

최대 공약수(GCD)는 가장 높은 공약수입니다. 두 개의 양의 정수(a, b)를 나누는 가장 큰 숫자입니다.

`**a = 12, b = 8`** 의 경우, 우리는 a의 약수 `**{1,2,3,4,6,12}**`와 b의 약수 `**{1,2,4,8}**`을 구할 수 있습니다. 이 두 가지를 비교하면 `**gcd(a,b) = 4**`를 알 수 있습니다.

이제 `**a = 11**`, `**b = 17**`을 가정해 보겠습니다. a와 b는 모두 소수이다. 소수는 자기 자신과 1을 약수로 가지므로 gcd(a, b) = 1입니다.

우리는 임의의 두 정수 a, b에 대해 gcd(a,b) = 1이면 a와 b는 서로소(coprime integers)라고 말한다.

a와 b가 소수라면(a != b), 그들은 서로소이다. 만약 a가 소수이고 b < a이면 a와 b는 서로소이다.

> ! a가 소수이고 b > a일 경우를 생각해보자, 왜 이 둘은 반드시 서로소가 아닐까?
{: .prompt-info}

# 예제

두 정수의 GCD를 계산하는 도구는 많지만, 이 작업을 위해 우리는 유클리드 알고리즘(유클리드 호제법)에 대해 찾아볼 것을 권장한다.

[유클리드 호제법](https://d0razi.github.io/posts/%EC%9C%A0%ED%81%B4%EB%A6%AC%EB%93%9C-%ED%98%B8%EC%A0%9C%EB%B2%95/)

그것을 코딩해보자. 몇 줄 밖에 안된다. a=12, b=8을 사용하여 테스트하라.

이제 a=66528, b=52920에 대한 gcd(a,b)를 계산하고 아래에 입력하자.

```python
def gcd(a, b):
    while b > 0:
        before = a
        a = b
        b = before%b
    return a

a = 66528
b = 52920
print(gcd(a,b))
```

math 모듈 사용

```python
from math import *
print(gcd(66528,52920))
```