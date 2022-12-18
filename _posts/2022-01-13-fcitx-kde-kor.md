---
layout: post
title:  "KDE Fcitx 한글 입력기 설정"
date:   2022-01-13 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Linux
tags:
   - Linux
   - Arch
toc: true
toc_sticky: true
---

# Fcitx 설정

Manjaro 리눅스를 설치하는 과정에서 한글 입력기에 대한 오류가 발생했습니다. 이전 Arch Linux 설치 포스트에서 설치했던 네 가지의 패키지를 설치했음에도 불구하고 한글이 정상적으로 입력되지 않았습니다.

조금 삽질한 과정과, 참고할 만한 다른 포스트를 공유합니다.

<!--more-->

## Manjaro (KDE Plasma 5) 환경

Manjaro KDE Plasma 5 (X11) 에서 설치된 환경은 아래와 같습니다.

```zsh
$ sudo pacman -Qs fcitx
local/fcitx 4.2.9.8-1 (fcitx-im)
    Flexible Context-aware Input Tool with eXtension
local/fcitx-hangul 0.3.1-3
    Hangul (Korean) support for fcitx
local/fcitx-qt5 1.2.6-1 (fcitx-im)
    Qt5 IM Module for Fcitx
local/kcm-fcitx 0.5.6-1
    KDE Config Module for Fcitx
```

이 [포스트](https://wnw1005.tistory.com/600) 에서는 `fcitx5` 를 사용하여 설정하고 있지만, 입력은 가능하나 일부 프로그램에서 한글이 깨지는 현상이 발생합니다. 아직까지는 `fcitx` 패키지가 KDE Plasma 5 와 상성이 맞는 것 같습니다. Arch Linux KDE에서 1년 동안 문제 없이 작동해 주었기 때문입니다.

## 환경변수 추가

Manjaro 에서는 아래와 같이 환경 변수를 추가해 주었습니다. 이것이 없을 때 로딩이 안되는 문제가 생기는 듯 합니다. 혹여나 다른 배포판에서 문제가 발생한다면 아래의 환경변수 추가를 시도해 보는 것이 좋을 것 같습니다.

```zsh
$ cat /etc/environment

#
# This file is parsed by pam_env module
#
# Syntax: simple "KEY=VAL" pairs on separate lines
#

GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

