---
title: "OMZ(Oh My ZSH) 커스텀(테마, 플러그인)"
author: d0razi
date: 2023-07-31 19:00
categories: [OS, Linux]
tags: [linux]
toc: false
image: /assets/img/media/linux.jpg
---

# 테마

저는 **agnoster**테마를 사용하겠습니다.

해당 테마는 별도의 설치없이 zshrc 문서에서 테마만 수정해주면 됩니다.

아래 명령어로 문서 수정을 해줍니다.

```bash
vi ~/.zshrc
```

```bash
ZSH_THEME="agnoster"
```

![Untitled](/assets/img/media/post_img/omz-custom/Untitled.png)

**robbyrussell**을  **agnoster**로 변경해주시면 됩니다.

아래 명령어를 입력해주시면 테마가 변경된 걸 확인할 수 있습니다.

```bash
source ~/.zshrc 
```

## 멀티라인 설정

```bash
vi ~/.oh-my-zsh/themes/agnoster.zsh-theme
```

위 명령어를 사용하고 맨 밑에 아래 코드를 붙여넣기 하시면 멀티라인이 설정됩니다.
[Untitled](/_posts/mutiline.md)

# 플러그인

## **Auto Suggestions**

터미널의 입력 history를 기반으로 추천을 해주는 플러그인입니다. 

![Untitled](/assets/img/media/post_img/omz-custom/Untitled%201.png)

위와 같이 뜰때 오른쪽 방향키를 눌러주면 완성이 됩니다.

1. 아래 명령어로 레포지토리 다운로드

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

1. ~/.zshrc 파일에 plugin 리스트에 zsh-autosuggestions를 추가

![Untitled](/assets/img/media/post_img/omz-custom/Untitled%202.png)

## **Syntax Highlighting**

zsh에서 해당 command가 존재하는 명령어인지 색상을 통해 알려줍니다.

![Untitled](/assets/img/media/post_img/omz-custom/Untitled%203.png)

1. 설치

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

1. 추가

특정 부분을 수정해야 하는게 아니라 그냥 파일 맨 마지막에 추가만 하면 됨으로 아래 명령어를 사용

```bash
echo "source ~/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${HOME}/.zshrc
```

## fzf

터미널에서 빠른 퍼지 파일 검색을 해주는 유틸리티입니다.

```bash
$ git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
$ cd ~/.fzf/
$ ./install
```

컨트롤 T를 누르고 원하는 파일명을 검색하시면 됩니다.