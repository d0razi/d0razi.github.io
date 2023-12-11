---
title: "우분투에 ZSH(oh my zsh) 설치하기"
author: d0razi
date: 2023-07-30 19:00
categories: [OS, Linux]
tags: [linux, Set up]
image: /assets/img/media/banner/linux.jpg
---

# ZSH 설치하기

아래 명령어로 zsh을 설치해줍니다.

```bash
sudo apt install zsh
```

설치된 경로는 아래 명령어로 알 수 있습니다.

```bash
which zsh
```

![Untitled](/assets/img/media/post_img/setting&custom/zsh-install/Untitled.png)

아래 명령어로 기본 쉘을 **bash**에서 **zsh**로 변경해줍니다.

```bash
chsh -s $(which zsh)
```

# Oh My ZSH 설치

Oh My ZSH를 설치하기 위해서는 'git' 그리고 'curl'이나 'wget'라는 유틸리티가 설치되어 있어야 합니다. 먼저 필요한 유틸리티들을 아래 명령어로 설치해줍니다.

```bash
sudo apt install git wget curl
```

아래 명령어로 wget을 사용해서 Oh My ZSH를 설치해줍니다.

```bash
sh -c "$(wget https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
```

![Untitled](/assets/img/media/post_img/setting&custom/zsh-install/Untitled%201.png)

설치가 잘 된 모습입니다.