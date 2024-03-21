---
title: "[Cryptohack] Base64"
author: d0razi
date: 2024-03-09 21:28
categories: [InfoSec, Crypto]
tags: [Cryptohack, Intro]
image: /assets/img/media/banner/cryptohack.png
---

# 문제

다른 인코딩 방식중엔 Base64가 있습니다. Base64란 64자의 알파벳을 사용하여 이진 데이터를 ASCII 문자열로 표현하는 것 입니다. Base64 문자열의 한 문자는 6개의 이진 숫자(비트)를 인코딩하므로 Base64의 4개 문자는 3개의 8비트 바이트를 인코딩합니다.

Base64는 온라인에서 가장 일반적으로 사용되므로 이미지와 같은 이진 데이터를 HTML 또는 CSS 파일에 쉽게 포함할 수 있습니다.

아래 16진수 문자열을 바이트로 디코딩한 다음 Base64로 인코딩하세요.

`72bca9b68fc16ac7beeb8f849dca1d8a783e8acf9679bf9269f7bf`

> Python에서는 import base64로 base64 모듈을 가져온 후 base64.b64encode() 함수를 사용할 수 있습니다. 챌린지 설명에 나와 있는 대로 먼저 16진수를 해독해야 합니다.
{: .prompt-info}

# 풀이

먼저 16진수 문자열을 바이트로 변환해줬습니다. 그리고 변환된 바이트를 base64로 인코딩해서 출력했습니다.

```python
import base64
flag = "72bca9b68fc16ac7beeb8f849dca1d8a783e8acf9679bf9269f7bf"

flag = bytes.fromhex(flag)
flag = base64.b64encode(flag)
print(flag)
```