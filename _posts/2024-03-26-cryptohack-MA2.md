---
title: "Modular Arithmetic 2"
author: d0razi
date: 2024-03-26 14:43
categories: [InfoSec, Crypto]
tags: [Cryptohack, MODULAR ARITHMETIC]
image: /assets/img/media/banner/cryptohack.png
---

# 문제

우리는 지난 도전에서 선택하고 모듈러스 `p`를 선택했다고 상상하고 `p`가 소수인 경우로 제한할 것입니다. 정수 모듈로 `p`는 `Fp`로 표시된 필드를 정의합니다.

> 만약 모듈러가 소수가 아니라면, 정수 모듈러 n 집합은 환(ring)을 형성합니다
{: .prompt-info}

유한체 Fp는 정수 {0,1...,p−1}의 집합이며, 덧셈과 곱셈 모두에서 a+b=0, a×b=1과 같이 집합의 모든 원소 a에 대한 역원이 존재합니다.

> 덧셈과 곱셈의 항등원(identity element)는 서로 다릅니다! 항등원은 연산자와 함께 동작할 때 아무것도 수행하지 않아야 합니다.  
> : a+0=a   
> : a×1=a  
{: .prompt-info}

![Image](https://i.imgur.com/0vwEvdU.png)
# 풀이

![Untitled](https://imgur.com/4XPaMxx.png)

`print(273246787654 ** 65536 % 65537)`