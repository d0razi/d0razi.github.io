---
title: "확장된 유클리드 알고리즘"
author: d0razi
date: 2023-03-22 10:51
categories: [InfoSec, Crypto]
tags: [python]
image: /assets/img/media/banner/Crypto.jpg
---


# 개념

유클리드 알고리즘이 a, b의 최대공약수 GCD(a, b)를 구하는 알고리즘이었다면

확장 유클리드 알고리즘은 a * s + b * t = GCD(a, b)를 만족하게 하는 정수 s, t를 구하는 알고리즘이다.

![그림](https://file.notion.so/f/f/8e307d96-2e7e-4f15-9203-84ce5ca02072/5677fee4-2b6c-45db-a4dd-83aec763c279/Untitled.png?id=31185540-a09e-4cc0-a11e-067420de5371&table=block&spaceId=8e307d96-2e7e-4f15-9203-84ce5ca02072&expirationTimestamp=1711159200000&signature=Pb750J8N0i986fXhw-cq_e69bXJEpwgdsYu2VgdJCI0&downloadName=Untitled.png)

초기 값은 `**s1, s2 = 1, 0**` `**t1, t2 = 0, 1`** 로 세팅해줍니다.

**초기값을 세팅하는 이유**

초기값 0,1
s와 t를 0, 1로 초기화하면, a와 b의 최대공약수 gcd(a, b)를 구하기 위해 b로 나누어 떨어지는 첫번째 값을 찾아냅니다. 이 값이 b 자체이므로 s=0, t=1로 초기화하면 gcd(a, b) = b가 됩니다.

초기값 1,0
그 다음, b와 a%b의 최대공약수를 찾기 위해 s와 t를 1,0으로 초기화합니다. 이 때, s=1, t=0으로 초기화하면 a%b가 b인 경우 s=1, t=0이 유지됩니다. 이는 a와 b의 최대공약수가 b이므로, b는 s의 계수가 되어야 하기 때문입니다. 이후, b와 a%b의 최대공약수를 찾기 위해서는 다시 s=0, t=1로 초기화하여 위의 과정을 반복합니다.

```python
def EEA(r1, r2, u1, u2, v1, v2) :
    if r2 == 0:
        print(f'gcd: {r1}\nu: {u1}\nv: {v1}')
        return
    q = r1//r2
    r = r1%r2
    u = u1 - q*u2
    v = v1 - q*v2

    return EEA(r2, r, u2, u, v2, v)

EEA(26513, 32321, 1, 0, 0, 1)
```