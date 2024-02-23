---
title: "[Pawnyable] 리눅스 커널 보호기법"
author: d0razi
date: 2024-02-23 20:45
categories: [InfoSec, Pwn]
tags: [kernel, Pawnyable]
image: /assets/img/media/banner/pawnyable_background.png
---

커널 exploit에 대한 대비책으로 리눅스 커널에는 보호기법들이 있습니다. 유저 영역의 보호기법 중 하나인 NX Bit처럼 하드웨어 수준에서의 보호기법도 존재하기 때문에, 몇가지 커널 보호기법에 대한 지식은 윈도우 커널 exploit에도 똑같이 적용이 가능합니다.

이 글에서 다루는 기법들은 커널만의 보호기법으로 Stack Canary와 같은 보호기법은 디바이스 드라이버에도 존재하지만 유저 영역의 Stack Canary와 다른 점이 없기에 설명하지 않겠습니다.

커널 부팅 시 파라미터에 대해서는 [공식 문서](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.txt)에 설명되어 있습니다.
# SMEP (Supervisor Mode Execution Prevention)
대표적인 커널 보호기법은 SMEP와 SMAP입니다.

SMEP는 커널 영역의 코드를 실행하는 동안 갑자기 사용자 영역의 코드를 실행하는 것을 금지하는 보호기법입니다(NX 보호기법을 생각하시면 비슷합니다).

SMEP는 단독으로 강한 보호기법이 아닙니다. 예를 들어 커널 취약점을 통해 공격자가 RIP를 컨트롤 할 수 있다고 해보겠습니다. 만약 SMEP가 비활성화되면 아래와 같이 유저 영역에 준비해놓은 셸코드를 실행하게 됩니다.
```c
char *shellcode = mmap(NULL, 0x1000, PROT_READ PROT_WRITE PROT_EXECUTE, MAP_ANONYMOUS   MAP_PRIVATE, -1, 0);

memcpy(shellcode, SHELLCODE, sizeof(SHELLCODE));  
  
control_rip(shellcode); // RIP = shellcode
```

그러나 SMEP가 유효한 경우 위와 같이 유저 영역에 준비한 셸 코드를 실행하려고 하면 커널 패닉이 발생합니다. 이로 인해 공격자는 RIP를 빼앗아도 권한 상승으로 연결할 수 없게 될 가능성이 높아집니다.


> 커널 공간의 셸 코드에서는 무엇을 실행하면 좋을까요?
> 권한 상승 방법은 또 다른 장에서 공부하겠습니다.
{: .prompt-tip}

SMEP는 qemu 실행 시 옵션으로 활성화 할 수 있습니다. 아래와 같이 `-cpu`옵션으로 `+smep` 라고 붙어 있으면 SMEP가 활성화됩니다.
```
-cpu kvm64,+smep
```

셸에서는 `/proc/cpuinfo`를 보는 것으로도 확인할 수 있습니다.
```bash
cat /proc/cpuinfo | grep smep
```

SMEP는 하드웨어 보호기법입니다. CR4 레지스터의 21번째 비트를 활성화하면 SMEP가 활성화됩니다.
# SMAP (Supervisor Mode Access Prevention)
유저 영역에서 커널 공간의 메모리를 읽고 쓸 수 없는 것은 보안상 당연하지만 커널 영역에서 유저 영역의 메모리를 읽고 쓸 수 없게 하는 **SMAP**(Supervisor Mode Access Prevention)라는 보안기구가 존재합니다. 커널 영역에서 유저 영역의 데이터를 읽고 쓰려면 `copy_from_user`라는 `copy_to_user` 함수를 사용해야 합니다.  
그런데 어째서 높은 권한의 커널 영역에서 낮은 권한의 유저 영역의 데이터를 읽고 쓸 수 없게 만드는 걸까요?

역사에 대해서는 모르지만, SMAP에 의한 혜택은 주로 2가지가 있습니다.

일단 첫 번째가 Stack Pivot 방지입니다.  
SMEP에서는 제시한 예시에선 RIP를 제어할 수 있어도 셸 코드는 실행할 수 없게 되었습니다. 그러나 리눅스 커널은 매우 방대한 양의 기계어를 가지고 있기 때문에 아래와 같은 ROP gadget이 반드시 존재합니다.
```
mov esp, 0x12345678; ret;
```

ESP에 들어가는 값이 무엇이든 이 gadget이 실행되면 RSP는 그 값으로 변경됩니다. 이러한 낮은 주소는 유저 영역에서 `mmap`으로 확보 가능하기 때문에 SMEP가 유효해도 RIP 제어가 가능하다면 아래와 같이 ROP chain을 실행할 수 있습니다.
```c
void *p = mmap(0x12340000, 0x10000, ...);  
unsigned long *chain = (unsigned long*)(p + 0x5678);  
*chain++ = rop_pop_rdi;  
*chain++ = 0;  
*chain++ = ...;  
...  
  
control_rip(rop_mov_esp_12345678h);
```

만약 SMAP가 유효하다면 유저 영역에서 mmap한 데이터(ROP chain)는 커널 영역에서 볼 수 없으므로 stack pivot의 `ret`명령으로 커널 패닉을 발생시킵니다.  
이와 같이 SMEP에 더해 SMAP가 활성화됨으로써 ROP에 의한 공격을 완화할 수 있습니다.

SMAP에 의한 두 번째 이점은 커널 프로그래밍으로 일어나기 쉬운 버그 방지입니다.  
여기에는 디바이스 드라이버 등의 프로그래머가 만드는 커널 특유의 버그와 연관이 있습니다. 드라이버가 다음과 같은 코드를 썼다고 치겠습니다(지금은 함수 정의는 몰라도 됩니다).
```c
char buffer[0x10];  
  
static long mydevice_ioctl(struct file *filp, unsigned int cmd, unsigned long arg) {  
	if (cmd == 0xdead) {  
		memcpy(buffer, arg, 0x10);  
	} else if (cmd == 0xcafe) {  
		memcpy(arg, buffer, 0x10);  
	}  
	return   
}
```

`memcpy`으로 `buffer`라는 글로벌 변수 데이터를 읽고 쓰는 것을 예상할 수 있습니다.

이 모듈은 유저 영역에서 다음과 같이 사용하면 0x10 바이트의 데이터를 기억해줍니다.
```c
int fd = open("/dev/mydevice", O_RDWR);  
  
char src[0x10] = "Hello, World!";  
char dst[0x10];  
  
ioctl(fd, 0xdead, src);  
ioctl(fd, 0xcafe, dst);  
  
printf("%s\n", dst); // --> Hello, World!
```

유저 영역 프로그래밍을 잘하면 버그가 발생할 일이 거의 없습니다. `memcpy`의 사이즈도 고정되어 있어서 특별한 문제는 없어 보입니다.

하지만 SMAP이 비활성화되면 다음과 같은 호출도 허용됩니다.
```c
ioctl(fd, 0xdead, 0xffffffffdeadbeef)
```

`0xffffffffdeadbeef`라는 주소는 유저 영역에서는 없는 주소지만, 만약 리눅스 커널한에 특정 데이터가 들어가는 주소면 디바이스 드라이버는
```c
memcpy(buffer, 0xffffffffdeadbeef, 0x10)
```

를 실행해서 특정 데이터를 읽어 버립니다. 이번 예시와 같이 아무런 체크없이 유저 영역에서 받은 주소로 `memcpy`를 호출해버리면, 유저 영역으로부터 커널 영역의 임의 주소 읽기, 쓰기(AAR, AAW)가 가능해집니다.

커널 프로그래밍에 익숙하지 않은 분들은 매우 찾기 힘든 취약점이지만 AAR, AAW가 발생하기 떄문에 영향이 매우 큰 취약점입니다. 이런 실수를 방지하는데도 SMAP는 도움이 많이 됩니다.

SMAP는 qemu 실행 시 옵션으로 활성화할 수 있습니다. 아래와 같이 `-cpu` 옵션에 `+smap`라고 적혀있으면 SMAP가 활성화됩니다.
```
-cpu kvm64, +smap
```

실행된 커널에서는 `/proc/cpuinfo`를 보는 것으로 확인할 수 있습니다.
```bash
cat /proc/cpuinfo | grep smep
```

SMAP도 SMEP와 같이 하드웨어 보호 기법입니다. CR4 레지스터의 22번째 비트를 활성화하면 SMAP이 활성화됩니다.
# KASLR / FGKASLR
유저 영역에서는 주소를 랜덤화해주는 ASLR(Address Space Layout Randomization)이 존재합니다. 이와 마찬가지로 리눅스 커널이나 디바이스 드라이버의 코드, 데이터 영역의 주소를 랜덤화하는 KASLR(Kernel ASLR)이라는 보호기법도 존재합니다.  
커널은 한번 로딩되면 주소가 변하지 않기 때문에 KASLR은 부팅 시 한번만 작동합니다. 어떤 주소라도 리눅스 커널 안에 있는 함수나 데이터 주소를 유출할 수 있으면 베이스 주소를 구할 수 있습니다.

2020년에 **FGKASLR**(Funtion Granular KASLR)이라는 더욱 강력한 KASLR이 등장했습니다. 이 기술은 리눅스 커널의 함수별로 주소를 랜덤화 하는 기술입니다. 이로인해 리눅스 커널 내부 함수 주소가 Leak 되더라도 베이스 주소를 구할 수 없습니다(현재는 비활성화 된 것 같다고 합니다). 그러나 FGKASLR은 데이터 영역 등은 랜덤화하지 않으므로 데이터 주소를 Leak 할 수 있다면 베이스 주소를 구할 수 있습니다. 무엇보다 베이스 주소를 기반으로 특정 함수의 주소를 구할 수 없지만, 나중에 등장하는 특수한 공격 기법에는 이용이 가능합니다.

주소는 커널 영역에서 겹치는 점을 주의하세요. 어떤 디바이스 드라이버가 KASLR 때문에 Exploit이 불가능하다고 해도 다른 드라이버에서 커널의 주소를 Leak해버리면 주소는 겹치기 때문에 exploit이 가능하게 됩니다.

KASLR은 커널 부팅 시 옵션으로 비활성화 할 수 있습니다. qemu `-append`옵션으로 `nokaslr`이라고 되어 있으면 KASLR은 무효화됩니다.
```bash
-append "... nokaslr ..."
```
# KPTI (Kernel Page-Table Isolation)
2018년에 Intel 등의 CPU에서 Meltdown이라 불리는 사이드 채널 공격이 발견되었습니다. 이 취약점에 대해서는 설명하지 않지만 커널 영역의 메모리를 사용자 권한으로 읽을 수 있는 위험한 취약점으로 KASLR 우회 등이 가능했습니다. 최근 Linux 커널에서는 Meltdown의 대책으로서 **KPTI**(Kernel Page-Table Isolation), 혹은 오래된 명칭으로 **KAISER**라고 불리는 기법이 활성화 되어 있습니다.

가상주소에서 물리주소로 변환할 때 페이지 테이블이 이용되는 것은 아시겠지만 이 페이지 테이블을 유저 모드와 커널 모드로 분리하는 것이 바로 이 보호기법입니다. KPTI는 어디까지나 Meltdown을 막기 위한 보안기구이므로 일반적인 커널 exploit에서는 문제가 되지 않습니다. 하지만 커널 영역에서 ROP 하는 경우 등 KPTI가 유효하면 마지막으로 사용자 공간으로 돌아갈 때 문제가 발생합니다. 구체적인 해결 방법은 Kernel ROP 장에서 설명하겠습니다.

KPTI는 커널 부팅 시 옵션으로 활성화 할 수 있습니다. qemu의 `-append` 옵션으로 `pti=on`라고 붙어있으면 KPTI는 유효화되고, `pti=off` 혹은 `nopti`가 붙어있으면 비활성화 됩니다.
```bash
-append "... pti=on ..."
```

KPTI는 `/sys/devices/system/cpu/vulnerabilities/meltdown`에서도 확인할 수 있습니다. 아래와 같이 "Mitigation: PTI"라고 쓰여 있으면 KPTI가 유효한 것입니다.
```bash
# cat /sys/devices/system/cpu/vulnerabilities/meltdown
Mitigation: PTI
```

유효하지 않은 경우는 \[Vulnerable] 이 됩니다.

KPTI는 페이지 테이블의 전환이므로 CR3 레지스터의 조작으로 사용자 커널 공간을 전환할 수 있습니다. 리눅스에서는 CR3에 0x1000을 OR(즉 PDBR을 변경)연산을 함으로써 커널 공간에서 사용자 공간으로 전환됩니다.
# KADR (Kernel Address Display Restriction)
리눅스 커널에서는 함수의 이름과 주소 정보를 `/proc/kallsyms`에서 읽을 수 있습니다. 또한 디바이스 드라이버에 따라서 `printk`함수 등을 사용하여 다양한 디버깅 정보를 로그로 출력할 수 도 있으며, 이 로그는 `dmesg` 명령어 등으로 사용자가 볼 수 있습니다.  
이와 같이 커널 공간의 함수나 데이터, 힙 등의 주소 정보 유출을 막기 위한 기법이 리눅스에는 존재합니다. 정식 명칭은 아직 없다고 생각하지만, 참고 문헌에서 **KADR**(Kernel Address Display Restriction)이라고 불러서 이 사이트에서도 그 명칭을 채용하겠습니다.

이 기능은 `/proc/sys/kernel/kptr_restirct`의 값에 따라 변경할 수 있습니다. `kptr_restrict`가 0인경우 주소 표시에 제한이 없습니다. `kptr_restrict`가 1인 경우, `CAP_SYSLOG`권한을 가진 사용자에게는 주소가 표시됩니다. 2인 경우 사용자가 특권 수준이여도 커널 주소는 숨겨집니다. KADR이 유효하지 않은 경우에는 주소 유출이 필요하지 않기에 처음에 확인하면 exploit이 간단해질 수 있습니다.

> **예제**
> 연습문제 LK01의 커널에서 아래의 과제를 해보세요(root 권한의 셸을 획득한 상태로 진행하세요).
> 1. `run.sh`를 확인하고 KASLR, KPTI, SMAP, SMEP가 유효한지 확인해보세요
> 2. SMAP, SMEP 둘 다 활성화 하는 옵션을 적어 부팅하고, `/proc/cpuinfo`를 보고 SMAP, SMEP가 활성화 되었는지 확인해보세요(확인 후 SMAP, SMEP는 다시 무효화해주세요).
> 3. `head /proc/kallsyms`에서 가장 먼저 출력되는 주소는 커널의 베이스 주소입니다. KASLR이 유효하지 않은 경우 베이스 주소가 몇개인지 확인해주세요(힌트 : KADR 주의).
{: .prompt-info}