---
title: "[Lazenca] Hello world!"
author: d0razi
date: 2023-12-12 18:12
categories: [Lazenca, Kernel, Dev Kernel Module]
tags: [linux, kernel]
image: /assets/img/media/banner/pwnable.jpg
---

# Hello world!
## OS Information
![](https://i.imgur.com/xO0T6SF.png)
## Installing the Development Environment
* **Linux Kernel Module**을 만들기 위해서는 아래 패키지들을 설치해야합니다.

```bash
sudo apt install build-essential make
```
## Simple Linux Kernel Module
* 아래 코드는 간단한 **Kernel Module** 코드입니다.
	* init_module( ) 함수는 Module이 Kernel에 삽입될 때 동작하는 코드입니다.
	* cleanup_module( ) 함수는 Module이 Kernel에서 제거될 때 동작하는 코드입니다.
	* 해당 코드에서 printk( ) 함수를 이용하여 Module이 삽입, 제거될 때 메시지를 출력합니다.

**sample.c**
```c
#include <linux/module.h> /* Needed by all modules */
#include <linux/kernel.h> /* Needed for KERN_INFO */

int init_module(void) {
    printk(KERN_INFO "Hello world - Lazenca0x0.\n");
    return 0;
}

void cleanup_module(void) {
    printk(KERN_INFO "Goodbye world - Lazenca0x0.\n");
}
```

* 아래와 같이 **Makefile**을 이용하여 **Module**을 빌드 할 수 있습니다.

**Makefile**
```Makefile
obj-m += sample.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

* 아래와 같이 **make**를 실행하면 **Kernel module**인 **".ko"** 확장자를 가진 파일이 생성됩니다.
	* Kernel module : sample.ko

**make**
```bash
❯ make
make -C /lib/modules/4.15.0-213-generic/build M=/home/d0razi/lazenca/Hello_world! modules
make[1]: Entering directory '/usr/src/linux-headers-4.15.0-213-generic'
  CC [M]  /home/d0razi/lazenca/Hello_world!/sample.o
  Building modules, stage 2.
  MODPOST 1 modules
WARNING: modpost: missing MODULE_LICENSE() in /home/d0razi/lazenca/Hello_world!/sample.o
see include/linux/module.h for more information
  CC      /home/d0razi/lazenca/Hello_world!/sample.mod.o
  LD [M]  /home/d0razi/lazenca/Hello_world!/sample.ko
make[1]: Leaving directory '/usr/src/linux-headers-4.15.0-213-generic'
❯ ls
Makefile  modules.order  Module.symvers  sample.c  sample.ko  sample.mod.c  sample.mod.o  sample.o
```

* **modinfo** 명령어를 이용하여 생성된 **Kernel module**의 정보를 확인할 수 있습니다.

**modinfo ./sample.ko**
```bash
❯ modinfo ./sample.ko
filename:       /home/d0razi/lazenca/Hello_world!/./sample.ko
srcversion:     9749A26456CFFEE86CBB1E2
depends:
retpoline:      Y
name:           sample
vermagic:       4.15.0-213-generic SMP mod_unload modversions
```
## insmod - simple program to insert a module into the Linux Kernel
* **insmod** 명령어를 이용하여 **Linux Kernel**에 **Module**을 등록할 수 있습니다.
* **lsmod** 명령어를 이용하여 **Linux Kernel**에 등록된 **Module**들을 확인할 수 있습니다.

**sudo insmod ./sample.ko**
```bash
❯ sudo insmod ./sample.ko
[sudo] password for d0razi:
❯ lsmod | grep sample
sample                 16384  0
```

* **dmesg** 명령어를 이용하여 **Module**에서 출력하는 메시지를 확인 할 수 있습니다.
	* "sample.ko" Module이 Kernel에 삽입된 후 "Hello world - Lazenca0x0." 이라는 메시지가 출력되었습니다.

**dmesg | tail**
```bash
❯ dmesg | tail
[    9.557488] NET: Registered protocol family 40
[   11.063918] IPv6: ADDRCONF(NETDEV_UP): ens33: link is not ready
[   11.073318] e1000: ens33 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: None
[   11.074795] IPv6: ADDRCONF(NETDEV_CHANGE): ens33: link becomes ready
[   13.298099] new mount options do not match the existing superblock, will be ignored
[ 1005.345317] sample: loading out-of-tree module taints kernel.
[ 1005.345320] sample: module license 'unspecified' taints kernel.
[ 1005.345320] Disabling lock debugging due to kernel taint
[ 1005.345361] sample: module verification failed: signature and/or required key missing - tainting kernel
[ 1005.346603] Hello world - Lazenca0x0.
```
## rmmod - simple program to remove a module from the Linux Kernel
```bash
❯ sudo rmmod sample
❯ dmesg | tail
[   11.063918] IPv6: ADDRCONF(NETDEV_UP): ens33: link is not ready
[   11.073318] e1000: ens33 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: None
[   11.074795] IPv6: ADDRCONF(NETDEV_CHANGE): ens33: link becomes ready
[   13.298099] new mount options do not match the existing superblock, will be ignored
[ 1005.345317] sample: loading out-of-tree module taints kernel.
[ 1005.345320] sample: module license 'unspecified' taints kernel.
[ 1005.345320] Disabling lock debugging due to kernel taint
[ 1005.345361] sample: module verification failed: signature and/or required key missing - tainting kernel
[ 1005.346603] Hello world - Lazenca0x0.
[ 1153.123745] Goodbye world - Lazenca0x0.
```
