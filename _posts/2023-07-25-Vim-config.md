---
title: "Vim 옵션 설정"
author: d0razi
date: 2023-07-25 19:00
categories: [OS, Linux]
tags: [linux]
toc: false
image: /assets/img/media/banner/linux.jpg
---

```bash
vi ~/.vimrc
```
위 명령어 사용 후 아래 내용 입력
```bash
set nu
set ai
set si
set expandtab	
set cindent
set autoindent
set smartindent
set sts=4
set ts=4
set shiftwidth=4
set wmnu
set laststatus=2
set ignorecase
set hlsearch
set nocompatible
set mouse=a
set ruler
set fileencodings=utf=8,euc-kr
set fencs=ucs=bom,utf-8,euc-kr
set showmatch
syntax on
filetype indent on
set bs=indent,eol,start
set title
set autoread
set autowrite
```