---
title: "[Pawnyable] Kernel Exploit에 대해"
author: d0razi
date: 2024-02-08 11:19
categories: [InfoSec, Pwn]
tags: [kernel, Pawnyable]
image: /assets/img/media/banner/pawnyable_background.png
---

# 커널 exploit 이란
'일반 바이너리 pwn은 공부했지만, 커널은 어려울 것 같다' 라는 분들이 많습니다. 하지만 사실 커널 exploit에서는 경우에 따라 공격이 매우 간단합니다.
이 글에서는 일반 바이너리 exploit과 커널 exploit의 차이점, 환경 구축 등에 대해 설명합니다.
## 공격 대상
유저영역 exploit과 커널 exploit의 가장 큰 차이점은 목적에 있습니다.
유저영역 exploit에서는 대부분 'RCE(임의 명령어 실행)'이라는 목표를 향해 exploit을 합니다. 한편 커널 exploit에서는 일반적으로 'Privilege Escalation(권한 상승)'을 위해 exploit을 합니다. 공격 대상의 머신에 침입할 수 있다는 가정하에 공격자는 커널 exploit을 이용하여 추가로 root 권한을 획득합니다(SMB Ghost처럼 머신 외부에서 커널 exploit을 하는 공격도 있습니다). 이와 같이 로컬 머신에서의 권한 상승을 **LPE**(Local Privilege Escalation)라고 부릅니다.
물론 유저영역에서 권한을 상승할 수 있는 취약점도 있지만, 그건 공격 대상 바이너리가 SUID가 설정되어 있기 때문입니다. 커널 exploit의 경우 공격 대상은 주로 다음 두 가지가 있습니다.
1. 리눅스 커널
2. 커널 모듈
리눅스 커널 내부 코드(syscall이나 file system)는 root 권한으로 동작하고 있기 때문에 커널 자체에 버그가 있는 경우 LPE로 연결될 가능성이 있습니다.
다른 하나는 장치 드라이버 등의 커널 모듈에 포함된 취약점입니다. 디바이스 드라이버는 유저 공간에서 외부 기기(프린터, 키보드 등)와의 교환을 간단하게 하기 위한 인터페이스입니다. 디바이스 드라이버도 root 권한으로 동작하기 때문에 버그가 있을 경우 LPE로 연결됩니다.

>>> 파일 시스템이나 Character module은 보통 커널 모듈에서 구현되지만 FUSE, CUSE 같은 기능이 등장하면서 유저 공간에서도 구현할 수 있게 되었습니다.
## 공격 방법
유저영역 exploit의 경우 일반적으로 공격 대상 서비스에 입력을 주는 것으로 exploit을 합니다. 그렇기 때문에 Python 등의 언어로 exploit 코드를 작성합니다.
한편, 커널 exploit의 경우는 대상이 OS 혹은 드라이버이빈다. 이러한 작업은 로우 레벨 작업이기 때문에 C언어를 사용하여 exploit을 작성하는 것이 주류입니다. 물론 Python 등에서도 쓸 수 있지만, 공격 대상 머신(특히 CTF나 실험 환경에서 준비되는 작은 Linux) 위에 Python이 존재하는 경우가 적습니다.
pawnyable에서도 C언어로 exploit을 작성합니다.
## 자원 공유
커널 exploit의 또 다른 특징은 자원이 공유된다는 점입니다.
유저영역에서는 보통 공격 대상 프로세스가 하나 존재하고, 그 프로세스를 exploit함으로써 쉘을 획득하는 등의 공격을 합니다. 한편, 리눅스 커널이나 디바이스 드라이버와 같은 프로그램은 OS를 이용하는 모든 프로세스에 공유되어 있습니다. System call은 누구나 원할 때 사용할 수 있고, 장치 드라이버도 누가 언제 조작할지 모릅니다. 즉, 커널 공간에서 작동하는 코드를 쓸 때는 항상 멀티 스레드로 프로그래밍하지 않으면 쉽게 취약점을 갖게 됩니다.
## 힙 영역 공유
커널의 힙 영역은 모든 드라이버나 커널에서 공유된다는 특징이 있습니다.
지금까지 유저영역 exploit에서는 프로그램마다 힙이 따로 있기 때문에 그 프로그램에서 Heap Overflow가 발생한다고 해서 다른 프로그램까지 exploit은 안됩니다. 하지만 예를 들어 디바이스 드라이버에서 한번 Heap Overflow가 발생하면 다른 디바이스 드라이버나 리눅스 커널이 힙에 확보한 메모리까지 오염됍니다.
공격자 관점에서 보면 이 특징에는 장단점이 있습니다. 
장점은 힙 주위의 작은 취약점만으로도 LPE로 연결될 가능성이 매우 높다는 점입니다. 예를 들면 함수 포인터를 가진 객체는 리눅스 커널에 많이 존재하기 때문에 그 중 하나를 이용해서 RIP를 쉽게 잡을 수 있습니다.
단점은 전체 프로그램의 영향을 받기 때문에 힙 상태를 예측할 수 없습니다. 유저영역의 프로그램은 유저가 입력한 값이 힙 메모리 상태에 전부였기 때문에 복잡한 Heap Exploit(House of XXX)이 가능했습니다. 하지만 커널에선 Heap Overflow가 발생하는 청크 뒤에 어떤 데이터가 존재하는지, Use After Free로 free 된 후 누가 그 주소를 사용하는지 알 수 없습니다.
취약점 자체는 유저영역과 크게 다르지 않습니다. Stack Overflow나 Use After Free 등은 커널 영역에도 존재할 수 있습니다. 또한 디바이스 드라이버 스택에도 보호 기법으로 Stack Canary를 사용할 수 있습니다. 하지만 커널 공간 특유의 취약점이 있기 때문에 나중에 다루도록 하겠습니다.
# qemu 이용
리눅스 커널 exploit을 할 때는 디버깅을 위해 에뮬레이터 위에서 커널을 작동시킵니다. VM이라면 뭐든 상관 없지만 qemu가 일반적이기 때문에 이 강의에서도 qemu를 사용하겠습니다

독자 분들은 자신의 환경에 맞게 qemu를 설치해주시기 바랍니다.
```bash
apt install qemu-system
```
# 디스크 이미지
qemu에서 머신을 실행할 때, 리눅스 커널과는 별도로 루트 디렉토리로서 마운트 되는 디스크 이미지가 필요합니다.
디스크 이미지는 일반적으로 ext 등의 파일 시스템의 바이너리 또는 cpio라고 불리는 형식으로 작성 및 배포됩니다.
파일 시스템의 경우는 mount 커맨드로 마운트하면 안의 파일을 편집할 수 있습니다.
```
# mkdir root
# mount rootfs.img root
```

이 강의에서 다루는 예제는 일반적으로 CTF에서 사용되는 cpio 형식을 사용합니다.
cpio 명령어를 사용하여 다음과 같이 파일을 전개합니다.
```
# mkdir root
# cd root; cpio -idv < ../rootfs.cpio
```

파일을 추가하거나 편집하면 다음과 같이 cpio 파일로 정리합니다.
```
# find . -print0 cpio -o --format=newc --null > ../rootfs_updated.cpio
```

cpio가 또한 gz로 압축되어 있는 경우도 있으므로 그때는 적절하게 압축 해제•재압축 해주시면 됩니다.
또한 cpio는 권한 정보도 부여하기 때문에 파일 시스템 편집에서는 적절히 파일 소유자를 root에 할당해야 합니다. 상기 커맨드는 모두 root 권한으로 실행하기 때문에 문제는 없겠지만, 귀찮은 경우에는 `--owner=root` 옵션을 주고 압축해도 상관없습니다.
```
$ mkdir root
$ cd root; cpio -idv < ../rootfs.cpio
...
$ find . -print0 | cpio -o --format=newc --null --owner=root > ../rootfs_updated.cpio
```