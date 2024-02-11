---
title: "[Pawnyable] gdb로 커널 디버깅해보기"
author: d0razi
date: 2024-02-11 16:10
categories: [InfoSec, Pwn]
tags: [kernel, Pawnyable]
image: /assets/img/media/banner/pawnyable_background.png
---

커널 exploit에 입문하기 어려운 가장 큰 원인은 디버깅 방법을 모른다는 것입니다.
이번 글에서는 gdb를 이용해 qemu 위에서 작동하는 리눅스 커널을 디버깅 하는 방법을 알아보겠습니다.
먼저 pawnyable의 연습문제 LK01 파일을 받아주세요.
# root 권한 취득
커널을 디버깅할 때 일반 사용자 권한이면 불편한 경우가 많습니다. 특히 커널이나 커널 드라이버에 브레이크 포인트를 설정하거나, Leak한 주소가 무슨 함수의 주소인지 알아볼 때 root 권한이 있어야 커널 공간의 주소 정보를 얻을 수 있습니다.
커널 exploit을 진행할 때는 먼저 root 권한을 취득합니다. 이번 파트는 LK01 문제를 root 권한으로 실행시키는 내용입니다.
<br>
커널이 시작되면 맨 처음에 특정 프로그램이 실행됩니다. 이 프로그램은 설정에 따라 경로는 다양하지만 대부분의 경우 `/init`이나`/sbin/init`등에 존재합니다. LK01의 `rootfs.cpio` 파일을 압축 해제하면 `/init` 파일이 존재합니다.
```bash
#!/bin/sh
# devtmpfs does not get automounted for initramfs
/bin/mount -t devtmpfs devtmpfs /dev

# use the /dev/console device node from devtmpfs if possible to not
# confuse glibc's ttyname_r().
# This may fail (E.G. booted with console=), and errors from exec will
# terminate the shell, so use a subshell for the test
if (exec 0</dev/console) 2>/dev/null; then
    exec 0</dev/console
    exec 1>/dev/console
    exec 2>/dev/console
fi

exec /sbin/init "$@"
```

이 프로그램에선 중요한 처리를 하고 있지는 않지만, `/sbin/init`을 실행하고 있습니다. 또한 CTF처럼 배포되는 미니멀한 환경에서는 `/init`에 직접 드라이버를 설치하거나 쉘을 시작하는 코드가 적혀있는 경우가 있습니다. 사실 마지막 `exec`줄 앞에 `/bin/sh`라고 쓰면 커널 부팅 시 root 권한으로 쉘을 부팅할 수 있습니다. 단, 드라이버 설치 등 필요한 다른 초기화 처리들이 실행 되지 않으므로 이번에는 이 방법을 사용하지 않겠습니다.
`/sbin/init`파일에서 최종적으로 `/etc/init.d/rcS`라는 쉘 스크립트가 실행됩니다. 이 스크립트는 `/etc/init.d`안에 `S`로 시작하는 이름의 파일을 실행합니다. 이번에는 `S99pawnyable`이라는 스크립트가 존재합니다. 이 스크립트에는 다양한 초기화 처리 코드가 적혀있는데, 마지막 줄에 중요한 코드가 적혀 있습니다.
```bash
setsid cttyhack setuidgid 1337 sh
```

위 코드가 이번 문제에서 커널 시작 시 사용자 권한으로 셸을 실행시켜 주는 코드입니다. `cttyhack` 명령어는 Ctrl+C 등의 입력을 사용할 수 있게 해주는 명령어입니다. 그리고 `setuidgid`명령어를 사용해 사용자 ID와 그룹 ID를 1337로 설정하고 `/bin/sh`을 실행합니다.
이 1337 숫자를 0(=root 사용자)로 바꿔줍니다.
```bash
setsid cttyhack setuidgid 0 sh
```

일부 보호기법 비활성화를 위해 아래 코드를 주석처리 해주세요.
```bash
 - echo 2 > /proc/sys/kernel/kptr_restrict
 + #echo 2 > /proc/sys/kernel/kptr_restrict
```

변경해준다음 cpio를 다시 압축해준 다음 `run.sh`를 실행하면 아래 사진과 같이 root 권한으로 셸이 실행되는 것을 볼 수 있습니다. (압축 방법은 이전 게시글 참고)
![](https://i.imgur.com/qU7jS5z.png)
# qemu를 이용해 커널에 gdb attach
qemu는 gdb에서 디버깅하기 위한 기능을 탑재하고 있습니다. qemu 명령어에 `-gdb`옵션을 주면 프로토콜, 호스트, 포트 번호를 지정하여 listen 할 수 있습니다. 예를 들어 `run.sh`에 아래 옵션을 추가하면 로컬 호스트에서 TCP 12345번 포트에서 gdb attach를 기다릴 수 있습니다.

> qemu 명령어에 -s 옵션을 사용하면 알아서 1234번 포트를 열리도록 할 수 있습니다.
{: .prompt-tip}

```bash
-gdb tcp::12345
```

이후에는 포트 번호를 변경하지 않고 12345번 포트를 이용하여 디버깅하지만, 자신이 원하는 포트로 변경해도 문제 없습니다.
gdb로 attach하려면 `target` 명령으로 대상을 지정할 수 있습니다.
```bash
gef> target remote localhost:12345
```

위 명령어로 연결이 되면 성공입니다. 나머지는 일반적인 gdb 커맨드를 이용해 레지스터나 메모리 읽기, 쓰기, 브레이크 포인트 설정 등이 가능합니다.
이번 문제는 대상 커널의 아키텍처가 x86-64입니다. 만약 여러분의 gdb가 표준인데 디버깅 대상의 아키텍처를 인식하지 못하면 아래 명령어로 아키텍처를 설정할 수 있습니다.
```bash
gef> set architecture i386:x86-64:intel
```
# 커널 디버깅
`/proc/kallsyms`라는 procfs를 통해서 리눅스 커널 안에서 정의된 주소와 심볼 목록을 볼 수 있습니다. KADR 파트에서 설명하겠지만 보호 기법에 의해 커널의 주소는 root 권한이여도 안 보일 수 있습니다.
**root 권한 획득 파트**에서 이미 했지만, 초기화 스크립트의 아래 코드를 주석 처리하는 것을 잊지 말아주세요. 이 작업을 안하면 커널 공간의 포인터가 보이지 않습니다.
```bash
echo 2 > /proc/sys/kernel/kptr_restrict
#echo 2 > /proc/sys/kernel/kptr_restrict
```

`kallsyms`파일은 매우 크기 때문에 head 명령어로 확인해보겠습니다.
![](https://i.imgur.com/AHWyGQr.png)

위처럼 심볼의 주소, 주소의 위치 섹션, 심볼 이름 순서로 출력됩니다. 섹션은 예를들어 "T" 면 text 섹션, "D"면 data 섹션과 같이 출력되며 대문자는 글로벌한 심볼을 나타냅니다. 이 문자들의 자세한 설명은 `nm` 명령어로 확인 할 수 있습니다.
예를들어 위 사진에선 0xffffffff81000000가 `_stext`라는 심볼의 주소임을 알 수 있습니다. 이 주소는 커널이 로드된 베이스 주소입니다.
<br>
그 다음에 `commit_creds`라는 이름의 함수의 주소를 grep을 사용해서 찾으면 됩니다. 찾은 주소에 bp를 설정하고 진행해주세요.
![](https://i.imgur.com/KSc4nE7.png)

![](https://i.imgur.com/P21yfDk.png)

위 함수는 새로 프로세스가 만들어질 때 호출되는 함수입니다. 셸에서 ls 명령어를 실행하면 중단점에서 멈춥니다.
![](https://i.imgur.com/u2v1EK7.png)

첫번째 인자를 전달해주는 RDI 레지스터에는 커널 공간의 포인터가 들어 있습니다. 이 포인터가 가리키는 메모리에는 1이 들어있습니다.
![](https://i.imgur.com/fh4kOpv.png)

위처럼 커널 공간에서도 유저 영역과 동일하게 gdb 명령어를 사용할 수 있습니다. pwndbg, gef 등의 플러그인을 사용할 수 있지만, 커널 디버깅을 지원하지 않는 플러그인이면 제대로 동작하지 않을 수 있습니다.
커널 디버깅을 지원하는 gdb 플러그인들도 많기 때문에 선호하는 디버거를 사용하시면 됩니다.
# 커널 모듈 디버깅
이제 커널 모듈을 디버깅 해보겠습니다.
LK01에는 vuln이라는 이름의 커널 모듈이 로드되어 있습니다. 로드되어 있는 모듈 리스트와 각 베이스 주소는 `/proc/modules`에서 확인할 수 있습니다.
![](https://i.imgur.com/2dO0QS6.png)

위 사진을 보면 `vuln`이라는 모듈이 0xffffffffc0000000 주소에 로드되어 있는 것을 알 수 있습니다. 이 모듈의 소스코드와 바이너리는 배포된 파일의 `src` 디렉토리에 있습니다. 상세한 소스 코드 분석은 다른 글에서 하겠지만, 일단 이 모듈의 함수들에 브레이크 포인트를 걸어보겠습니다.
IDA, Binary Ninja 같은 분석 툴로 `src/vuln.ko`를 확인해보면 함수 몇개를 볼 수 있습니다. 예시로 `module_close` 함수의 상대 주소는 0x25f 임을 알 수 있습니다.
![](https://i.imgur.com/OIit6Nq.png)

따라서 현재 커널 상에서는 0xffffffffc0000000 + 0x25f 메모리에 이 함수가 존재할 것입니다. 여기에 브레이크 포인트를 걸어 보겠습니다(바이너리 닌자에서는 0x25f라고 나왔지만 gdb로 디버깅을 해보니 0x20f로 찾았습니다).
![](https://i.imgur.com/k6NuTR6.png)

자세한 내용은 앞에서 해석했지만, 이 모듈은 `/dev/holstein` 이라는 파일에 매핑이 되어 있습니다. `cat /dev/holstein`커맨드를 사용하면 `module_close`를 호출하기 때문에 브레이크 포인트를 설정해놓고 멈추는 것을 확인해봅시다.

> 드라이버의 심볼 정보를 원할 경우 add-symbols-file 명령을 사용하여 첫 번째 인자에 보유한 드라이버, 두 번째 인자에 베이스 주소를 주면 심볼 정보를 읽어줍니다.  
> 함수명을 사용해서 브레이크 포인트를 설정할 수 있어요.
{: .prompt-tip}


```bash
# cat /dev/holstein
```

![](https://i.imgur.com/iiZZKIq.png)

![](https://i.imgur.com/4ViOUFZ.png)

> **예제**  
> 이번 글에서는 `commit_creds`에 브레이크 포인트를 설정해서 RDI 레지스터가 가리키는 메모리 영역을 확인했습니다. 해당 작업을 유저 권한의 쉘(uid = 1337)에서 했을 경우 어떻게 될 지 gdb를 사용해 확인해봅시다.  
> 또한 root 권한(uid = 0)의 경우와 일반 사용자 권한(uid = 1337)의 경우를 비교하여 `commit_creds` 첫 번째 인자로 전달되는 데이터에 어떤 차이가 있는지 확인해보세요.
{: .prompt-info}
