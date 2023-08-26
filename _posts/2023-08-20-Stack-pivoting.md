---
title: "Stack pivoting"
author: d0razi
date: 2023-08-20 19:00
categories: [InfoSec, Pwn]
tags: [linux, Exploit Tech]
image: /assets/img/media/banner/pwnable.jpg
---

## Stack pivoting이란?
ROP를 해야하는데 ret까지 밖에 bof가 가능할 때 사용 가능한 기법입니다.
* 특정 영역에 Write 권한이 있을때 영역에 가젯들을 세팅해놓고 sfp를 조작하고 ret에 leave; ret 가젯을 넣어서 원하는 주소를 실행시킬 수 있습니다.

### leave; ret 가젯
leave와 ret 명령어는 각각 아래와 같은 동작을 합니다.
**leave**
>mov esp, ebp
>pop ebp

**ret**
>pop eip
>jmp eip

때문에 leave; ret 가젯 이전에 rbp값을 [원하는 주소 - 8] 로 세팅 할 수 있다면 원하는 주소를 실행시킬 수 있습니다.