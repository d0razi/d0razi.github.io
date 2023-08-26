---
title: "RTC [Return to CSU]"
author: d0razi
date: 2023-08-11 19:00
categories: [InfoSec, Pwn]
tags: [linux, Exploit Tech]
image: /assets/img/media/banner/pwnable.jpg
---

# Return to CSU

ROP를 진행할 때는 가젯이 필요합니다.

예를 들어 `puts(”Hello World”)` 코드를 실행시키려고 하면 `pop rdi ret` 가젯이 필요합니다.

하지만 바이너리에 필요한 가젯이 없는 경우가 있습니다. 이런 상황에 사용할 수 있는 기법이 RTC기법입니다.

![[출처] https://wogh8732.tistory.com/156](/assets/img/media/post_img/pwn/RTC%20[Return%20to%20CSU]/Untitled.png)

[출처] https://wogh8732.tistory.com/156

ELF 파일은 위 사진처럼

**start → libc_start_main → libc_csu_init → ...  → main**

내부적인 여러 단계를 거쳐 main 함수가 실행됩니다.

이때 실행되는 libc_csu_init 함수를 살펴보겠습니다.

![Untitled](/assets/img/media/post_img/pwn/RTC%20[Return%20to%20CSU]/Untitled%201.png)

두 부분으로 나누어서 설명하겠습니다.

**파란 박스** = csu_init

**빨간 박스** = csu_call

## csu_init

스택에 있는 값을 rbx - rbp - r12 - r13 - r14 - r15 순서대로 레지스터에 저장합니다.

## csu_call

**r13에 들어있는 값을 rdx로, r14에 들어있는 값을 rsi로, r15에 들어있는 값을 edi로 옮겨줍니다.**

그러면 우리는 csu_init을 호출한 후 csu_call을 호출함으로서 **rdi, rsi, rdx 레지스터를 조작**할 수 있게됩니다. 아래 보면 `call QWORD PTR [r12+rbx*8]` 명령이 있습니다. **csu_init**에서 **rbx** 와 **r12를** 조작할 수 있기 때문에 이 명령으로 우리가 원하는 주소를 실행할 수 있습니다. 

rbx에 0을 넣고 r12에 원하는 함수의 got 주소를 입력해주면 됩니다.

> **PLT는 안되는 이유**
plt에는 got의 주소가 저장되어 있습니다. r12에 적힌 값(주소)를 바로 실행하는데, plt 주소를 주면 segmentation fault가 발생합니다. 그렇기에 실주소가 담긴 got를 입력해줘야합니다.
> 

그 아래에는 rbx에 1을 더하고, rbx와 rbp를 비교 후 값이 동일하면 그대로 코드가 진행되어서 다음 라인으로 진행됩니다. **즉 csu_init이 한번 더 실행된다는 뜻입니다.** 그렇기 때문에 rbp에는 1을 넣어두어야 **csu_init**을 한번 더 실행하는 효과를 내서 원하는 주소를 또 실행할 수 있게 됩니다.

이후 진행할 때 주의해야하는 부분은 `add rsp, 0x8` 명령으로 스택을 늘려주기 때문에 `payload += b’A’ * 8` 처럼 늘어난 스택을 다시 채워줘야 합니다.

# Example

write 함수를 이용하여 read함수의 got 출력 후 read 함수로 write_got에 값을 입력받는 코드를 짜보겠습니다.

```python
# write(1, read_got, 8)
payload = bof 트리거
payload += p64(csu_init)
payload += p64(0)           # rbx
payload += p64(1)           # rbp
payload += p64(write_got)   # r12, no plt ok got
payload += p64(8) + p64(read_got) + p64(1) # r13(rdx), r14(rsi), r15(edi)
payload += p64(csu_call)

# read(0, write_got, 8)
payload += b'A' * 8 # add rsp, 8
payload += p64(0)           # rbx
payload += p64(1)           # rbp
payload += p64(read_got)    # r12, no plt ok got
payload += p64(8) + p64(write_got) + p64(0) # r13(rdx), r14(rsi), r15(edi)
```

---

**주의해야할 점**

call [r12] 은 r12 공간에 저장된 데이터를 참조한다는 것입니다.

**main을 call하고 싶다고 r12를 main 주소로 해놓으면 안됩니다.**

main으로 가고 싶으면 **bss 영역에 main 주소**를 넣어놓고, 해당 bss 주소를 r12로 설정해줘야 합니다.