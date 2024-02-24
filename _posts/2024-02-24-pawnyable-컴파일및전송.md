---
title: "[Pawnyable] Exploit 코드 컴파일 및 바이너리 전송"
author: d0razi
date: 2024-02-24 15:32
categories: [InfoSec, Pwn]
tags: [kernel, Pawnyable]
image: /assets/img/media/banner/pawnyable_background.png
---

커널 부팅 방법, 디버깅 방법, 보호기법 등 Kernel Exploit을 시작하는데 필요한 지식들은 완벽하게 습득했습니다. 이제부터는 실제로 exploit을 어떻게 작성하는지, 작성한 exploit을 어떻게 qemu에 올린 kernel에서 작동하게 할 것인지를 배워보겠습니다.
# qemu에서 실행
qemu 위에서 exploit을 작성 후 빌드, 실행하면 커널이 크래시 될 때마다 모든 작업을 다시 해야하기 때문에 귀찮습니다. 그렇기 때문에 C언어로 작성한 exploit을 로컬에서 빌드를 한 후 qemu로 보내야 합니다. 이 작업을 매번 명령어로 입력하는 것은 귀찮기 때문에 셸 스크립트 등으로 자동화를 해줍시다. 아래는 예시 파일인 `transfer.sh`입니다.
```bash
#!/bin/sh  
gcc exploit.c -o exploit  
mv exploit rootfs  
cd rootfs; find . -print0 | cpio -o --null --format=newc > ../debugfs.cpio  
cd ../  
  
qemu-system-x86_64 \
    -m 64M \
    -nographic \
    -kernel bzImage \
    -append "console=ttyS0 loglevel=3 oops=panic panic=-1 nopti nokaslr" \
    -no-reboot \
    -cpu qemu64 \
    -smp 1 \
    -monitor /dev/null \
    -initrd debugfs.cpio \
    -net nic,model=virtio \
    -net user   \
    -s
```

동작 과정은 단순히 gcc로 `exploit.c` 파일을 컴파일한 후 생성된 exploit 파일을 cpio에 추가하고 qemu를 기동하는 스크립트입니다. 원본 파일인 `rootfs.cpio`가 깨지지 않도록 `debugfs.cpio`라고 지정해서 사용하지만, 취향에 따라 변경해도 상관 없습니다.  
또한 cpio 파일을 만들 때 root 권한이 아니면 파일의 권한이 바뀌기 때문에 `transfer.sh`는 root 권한으로 실행해주세요.

`exploit.c`에 아래와 같은 코드를 작성 후 `transfer.sh`를 실행해 봅시다.
```c
#include <stdio.h>  
  
int main() {  
	puts("Hello, World!");  
	return 0;  
}
```

그러면 아래와 같이 오류가 납니다. 왜 그럴까요?
![](https://i.imgur.com/tcP6MP2.png)

사실 이번에 배포한 이미지는 통상적인 libc가 아니라 uClibc라는 컴팩트한 라이브러리를 사용하고 있습니다. 당연히 exploit을 컴파일한 여러분의 환경에서는 GCC, 즉 일반적인 libc를 사용하기 때문에 동적 링크에 실패해서 exploit은 작동하지 않습니다.  
따라서 qemu 상에서 exploit을 실행할 때는 static 링크를 해주세요
```bash
gcc exploit.c -o exploit -static
```

위처럼 `transfer.sh`를 변경하고 실행해보면 프로그램이 정상적으로 작동할 것입니다.
![](https://i.imgur.com/E4Ev1Oj.png)
# 원격 머신 실행 : musl-gcc 이용
지금까지 무사히 exploit 파일을 qemu 상에서 실행하는 방법을 알았습니다. 이번에 배포한 환경은 네트워크 접속이 가능하도록 설정되어 있기 때문에 원격으로 실행하고 싶은 경우에는 qemu에서 wget 명령어 등을 이용하여 exploit을 전송할 수 있습니다.  
그러나 CTF 등 일부 작은 환경에서는 네트워크를 이용하지 못 할 수도 있습니다. 이런 경우에는 busybox에 있는 명령어를 이용해서 원격으로 바이너리를 전송해야 합니다. 일반적으로는 base64가 사용되는데 GCC에서 빌드한 파일은 수백 KB에서 수십 MB가 되기 때문에 전송에 많은 시간이 소요됩니다. 크기가 커지는 것은 외부 라이브러리(libc)의 함수를 static 링크했기 때문입니다.  
gcc에서 크기를 줄이고 싶다면 libc를 사용하지 않게 하고 read나 write 등은 시스템 콜(인라인 어셈블리)을 사용해야 합니다. 물론 이것은 매우 귀찮은 작업입니다.  
그래서 많은 CTF 유저들은 Kernel Exploit을 목적으로 musl-gcc라고 불리는 C 컴파일러를 이용합니다. 아래 링크에서 다운로드해서 설치해주세요.

https://www.musl-libc.org/
```
# ./config
# make ARCH=x86_64
# make install
```

설치가 완료되면 아래와 같이 `transfer.sh`의 컴파일 부분을 고쳐 봅시다. musl-gcc의 경로는 각자 설치한 곳의 디렉토리를 지정해주세요. 더 작게 하고 싶은 경우 strip 등으로 디버깅 심볼을 삭제해도 됩니다.

> 일부 헤더 파일(Linux 커널 계열)은 musl-gcc에 없기 때문에 include 경로를 설정하거나 gcc로 컴파일 할 필요가 있습니다. 그럴 때는 어셈블리를 경유해서 빌드하면 gcc 기능을 사용하면서 파일 크기를 줄일 수 있습니다.
> `gcc -S sample.c -o sample.S`
> `musl-gcc sample.S -o sample.elf`
{: .prompt-tip}

여기까지 완료되면 원격으로 (nc를 통해) base 64를 사용하여 바이너리를 전송하는 스크립트를 작성하도록 하겠습니다. 이 업로드 스크리트는 CTF에서 매번 사용하기 때문에 템플릿으로 자신의 것을 만들어 두는 것을 추천드립니다.

```python
from ptrlib import *
import time
import base64
import os

def run(cmd):
    sock.sendlineafter("# ", cmd) # root = #, usr = $
    sock.recvline()

with open("./rootfs/exploit", "rb") as f:
    payload = bytes2str(base64.b64encode(f.read()))

#sock = Socket("HOST", PORT) # remote
sock = Process("./run.sh")

run('cd /tmp')

logger.info("Uploading...")
for i in range(0, len(payload), 512):
    print(f"Uploading... {i:x} / {len(payload):x}")
    run('echo "{}" >> b64exp'.format(payload[i:i+512]))
run('base64 -d b64exp > exploit')
run('rm b64exp')
run('chmod +x exploit')

sock.interactive()
```

실행하고 잠시 후에 아래와 같이 업로드가 완료됩니다.

![](https://i.imgur.com/mcXJlAT.png)
