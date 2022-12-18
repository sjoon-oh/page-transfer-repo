---
title:  "아치 리눅스 블루투스 설정하기"
date:   2021-06-22 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Linux
tags:
   - Linux
   - Bluetooth
toc: true
toc_sticky: true
breadcrumbs: true
---

# 아치(Arch) Bluetooth 설정하기

이전 [포스트](https://sjoon-oh.github.io/archivers/arch-installation)에서 아치 리눅스 설치 과정을 다루었습니다. 이 포스트에서는 KDE 환경을 제 스타일에 맞게 꾸미는 과정을 다루었습니다. KDE 버전은 아래와 같습니다.

![sc1](/assets/posts/2021-04-08-arch-kde-themes/screen1.png)

<!--more-->

한동안 유선 이어폰만을 사용하다가 노트북으로 블루투스를 연결할 일이 생겼습니다.

## bluez 패키지 설치

```bash
$ sudo pacman -S bluez bluez-utils
$ reboot
```

이후 재시작을 해 줍니다.

## 모듈 불러오기

```bash
$ modprobe bluez # module load
$ modprobe -r bluez # module unload
```

## Service 시작

```bash
$ systemctl start bluetooth.service
$ systemctl enable bluetooth.service.
```