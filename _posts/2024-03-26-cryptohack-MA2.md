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

![Untitled](https://file.notion.so/f/f/8e307d96-2e7e-4f15-9203-84ce5ca02072/ef5c35b5-19d9-4541-a74d-7fb9cefb1652/Untitled.png?id=033031d6-a4e3-45dd-8296-2a5c5d84d6e4&table=block&spaceId=8e307d96-2e7e-4f15-9203-84ce5ca02072&expirationTimestamp=1711641600000&signature=msyLyq3z2AlbfyLci021uGYfN7qv_yJWuNPdLMVOOfU&downloadName=Untitled.png)

# 풀이

![Untitled](https://file.notion.so/f/f/8e307d96-2e7e-4f15-9203-84ce5ca02072/74bc0b5f-23b7-4f6a-8a05-040245e0c5a1/Untitled.png?id=10855325-1c2d-4f37-91c0-6747335812b8&table=block&spaceId=8e307d96-2e7e-4f15-9203-84ce5ca02072&expirationTimestamp=1711641600000&signature=8-k3UpjSFtWru48qgzE-vlQKaJr8wQPpw0juZf-gynE&downloadName=Untitled.png)

`print(273246787654 ** 65536 % 65537)`