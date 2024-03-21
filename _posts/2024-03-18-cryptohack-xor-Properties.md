---
title: "[Cryptohack] XOR Properties"
author: d0razi
date: 2024-03-18 12:27
categories: [InfoSec, Crypto]
tags: [Cryptohack, Intro]
image: /assets/img/media/banner/cryptohack.png
---

# 문제 

지난 문제에서는 XOR이 비트 수준에서 어떻게 동작하는지 보았다.

이번 문제서는 XOR 작업의 속성을 다루고 플래그를 암호화한 일련의 작업을 무력화하는데 사용할 것이다.

이것이 어떻게 작동하는지에 대한 직관을 얻는 것은 나중에 특히 블록 암호 범주에서 실제 암호 시스템을 공격할 떄 크게 도움이 될 것이다.

XOR 연산자를 사용하여 문제를 해결할 때 고려해야 할 네 가지 주요 특성은 다음과 같다.

> 교환법칙: A ⊕ B = B ⊕ A
> 
> 
> 결합법칙: A ⊕ (B ⊕ C) = (A ⊕ B) ⊕ C
> 
> 항등식: A ⊕ 0 = A
> 
> 역원: A ⊕ A = 0
> 
{: .prompt-tip}

이걸 분석해보자. 교환법칙은 XOR 작업의 순서가 중요하지 않음을 의마한다. 결합법칙은 일련의 작업을 순서 없이 수행할 수 있음을 의미한다.(우리는 괄호를 걱정할 필요가 없다.) 항등원이 0이므로 0과 XOR하는 것은 "아무것도 하지 않는다" 라고 하며, 마지막으로 자기 자신을 XOR 연산하면 0을 반환한다.

아래는 세 개의 임의 키가 플래그와 함께 XOR된 일련의 출력입니다. 플래그를 얻기 위해 위의 속성을 사용하여 마지막 줄에서 암호화를 실행 취소합니다.

KEY1 = a6c8b6733c9b22de7bc0253266a3867df55acde8635e19c73313

KEY2 ^ KEY1 = 37dcb292030faa90d07eec17e3b1c6d8daf94c35d4c9191a5e1e

KEY2 ^ KEY3 = c1545756687e7573db23aa1c3452a098b71a7fbf0fddddde5fc1

FLAG ^ KEY1 ^ KEY3 ^ KEY2 = 04ee9855208a2cd59091d04767ae47963170d1660df7f56f5faf

> 이러한 개체를 XOR하기 전에 16진수에서 바이트로 디코딩해야 합니다.
{: .prompt-info}

# 풀이

```python
from Crypto.Util.number import *

key1 = 0xa6c8b6733c9b22de7bc0253266a3867df55acde8635e19c73313
key23 = 0xc1545756687e7573db23aa1c3452a098b71a7fbf0fddddde5fc1
print(long_to_bytes(key1^key23^0x04ee9855208a2cd59091d04767ae47963170d1660df7f56f5faf))
```