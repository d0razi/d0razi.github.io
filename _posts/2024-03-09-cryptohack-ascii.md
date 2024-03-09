---
title: "[Cryptohack] ASCII"
author: d0razi
date: 2024-03-09 21:15
categories: [InfoSec, Crypto]
tags: [Cryptohack]
image: /assets/img/media/banner/cryptohack.png
---

# ASCII

# 문제

ASCII는 0-127 정수를 사용하여 텍스트를 표현할 수 있는 7비트 인코딩 표준입니다.

아래 정수 배열을 사용하여 숫자를 ASCII 문자로 변환하여 플래그를 얻어보세요.

`[99, 114, 121, 112, 116, 111, 123, 65, 83, 67, 73, 73, 95, 112, 114, 49, 110, 116, 52, 98, 108, 51, 125]`

> Python에서 chr() 함수는 ASCII 코드 숫자를 문자로 변환해줍니다(ord() 함수는 문자 → 숫자).
{: .prompt-info}

# 풀이

주어진 배열을 그대로 파이썬에 배열로 만들어줍니다.

그리고 for문을 사용하여 배열에 들어있는 숫자 하나씩 문자열로 변경해서 출력해줍니다.

```python
flag = [99, 114, 121, 112, 116, 111, 123, 65, 83, 67, 73, 73, 95, 112, 114, 49, 110, 116, 52, 98, 108, 51, 125]
for i in range(0, len(flag)):
  print(chr(flag[i]), end="")
```