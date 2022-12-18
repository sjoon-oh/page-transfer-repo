---
layout: post
title:  "아치(Arch) 리눅스 KDE 데스크탑 환경 설정하기"
date:   2021-04-08 09:00:00
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

# 아치(Arch) KDE 데스크탑 환경 설정기 (업데이트)

이전 [포스트](https://sjoon-oh.github.io/archivers/arch-installation)에서 아치 리눅스 설치 과정을 다루었습니다. 이 포스트에서는 KDE 환경을 제 스타일에 맞게 꾸미는 과정을 다루었습니다. KDE 버전은 아래와 같습니다.

![sc1](/assets/posts/2021-04-08-arch-kde-themes/screen1.png)

<!--more-->

# System Settings: Appearance

## Global Themes

개인적으로는 기본 테마가 가장 미니멀하고 간단해서 좋아합니다. 그래서 Global Themes 탭에서 기본 Breeze로 설정합니다.

![sc2](/assets/posts/2021-04-08-arch-kde-themes/screen2.png)

## Application Style

### Lightly 설치하기

```bash
$ git clone --single-branch --depth=1 https://github.com/Luwx/Lightly.git

$ cd Lightly && mkdir build && cd build
$ cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=lib -DBUILD_TESTING=OFF ..
$ make
$ sudo make install
```

### Lightly 설정하기

![sc11](/assets/posts/2021-04-08-arch-kde-themes/screen11.png)

![sc12](/assets/posts/2021-04-08-arch-kde-themes/screen12.png)

Transparency를 살짝 조정해줍니다.

## Colors

앞서 설치했던 Lightly를 선택하고 세부 조정은 Default로 설정해줍니다.

![sc13](/assets/posts/2021-04-08-arch-kde-themes/screen13.png)

## Windows Decorations

앞서 설치했던 Lightly 로 설정합니다.

![sc14](/assets/posts/2021-04-08-arch-kde-themes/screen14.png)

## Icons

Tela Circle Icon으로 설정하겠습니다. 가장 무난하면서도 괜찬더라구요.

![sc3](/assets/posts/2021-04-08-arch-kde-themes/screen3.png)
![sc4](/assets/posts/2021-04-08-arch-kde-themes/screen4.png)

이 외의 설정은 건드리지 않습니다. 참고로 한글 폰트가 설치되어있지 않은 경우 깨짐 문제가 발생할 수 있으니 폰트를 설치 해 줍니다.

```bash
# Korean Font
$ pacman -S adobe-source-han-sans-kr-fonts
$ pacman -S fcitx-im fcitx-hangul kcm-fcitx # 입력기 설정
```


# Workspace Behavior

## General Behavior

기본 애니메이션 빠르기는 조금 답답하게 느껴집니다. 그래서 조금 빠르게 설정하였습니다. 추가적으로, Clicking files or folders 옵션을 Selects them으로 설정 해 줍니다. 이 설정이 켜져있지 않다면 Dolphin에서 파일을 클릭만 하면 열리거나 실행되므로 주의해야 합니다.


![sc5](/assets/posts/2021-04-08-arch-kde-themes/screen5.png)

## Desktop Effects

제가 기본 설정에서 바꾼 옵션들은 다음과 같습니다.

### Appearance
- Wobbly Windows (Enable, Sub-setting: Default)
- Magic Lamp (Enable, Sub-setting: Default)

![sc6](/assets/posts/2021-04-08-arch-kde-themes/screen6.png)

### Focus
- Dialog Parent (Enable)
- Dim Inactive (Enable)
- Slide Back (Enable)

![sc7](/assets/posts/2021-04-08-arch-kde-themes/screen7.png)

### Window Open/Close Animation

- Scale (Enable)

### Screen Edges

모서리 옵션 체크를 모두 해제합니다.

![sc8](/assets/posts/2021-04-08-arch-kde-themes/screen8.png)

## Window Management
### Task Switcher

Main - Visualization 종류를 Cover Switch 로 세팅하고 세부 설정에서 Animation Duration을 300ms 으로 설정하겠습니다.

![sc9](/assets/posts/2021-04-08-arch-kde-themes/screen9.png)

![sc10](/assets/posts/2021-04-08-arch-kde-themes/screen10.png)



# Desktop 설정

아래와 같이 Desktop을 설정해 보겠습니다.

![sc15](/assets/posts/2021-04-08-arch-kde-themes/screen15.png)

## Latte Dock

### Latte Dock 설치

yay 가 설치되어 있지 않다면 먼저 설치 해 줍니다. Floating Dock을 만들기 위해서는 Latte Dock 0.10 버전 이상이 필요합니다만 21.04.08. 기준으로는 pacman에 올라가 있는 버전은 0.9 버전입니다. 따라서 직접 컴파일 하는 방식을 사용하겠습니다.

```bash
$ cd ~/Downloads
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
```

아래와 같이 설치합니다.

```bash
$ yay -S latte-dock-git
```

### ### Latte Dock 설치 (New)

21.11.02. 기준 확인해보니 pacman 으로 0.10 이상이 업데이트 되었습니다. 위의 과정은 단순하게 pacman으로 설치해 주시면 됩니다.
### 하단 Latte Dock

Behavior 탭에서는 Dodge Maximized 선택하고 그 외 설정은 건들지 않습니다. 

![ld1](/assets/posts/2021-04-08-arch-kde-themes/latte1.png)

Appearance 탭에서는 아래와 같은 설정을 적용합니다.

- Items - Zoom on hover: 50%
- Length - Maximum: 80%
- Margins - Thickness: 8%
- Margins - Screen edge: 15px
- Background - Size: 100%
- Background - Opacity: 65%
- Background - Radius: 38%
- Background - Blur (Enable)
- Background - Shadows (Enable)

![ld2](/assets/posts/2021-04-08-arch-kde-themes/latte2.png)

Edit mode에서 Dock 형식으로 지정합니다.


### 상단 Latte Dock

Edit mode에서 Panel 형식으로 지정합니다. 그리고 아래 스크린샷과 같이 Pannel에서 Global Menu 위젯을 추가합니다. Global Menu 위젯은 KDE에 기본적으로 포함되어 있습니다.

추가적으로 우측 상단에 Netspeed Widget 을 추가해줍니다.

![ld3](/assets/posts/2021-04-08-arch-kde-themes/latte3.png)

Behavior 탭에서는 Dodge Maximized 선택하고 그 외 설정은 건들지 않습니다. 

Appearance 탭은 아래와 같이 설정합니다.

- Items - Absolute size: 40px
- Items - Zoom on hover: 0%
- Length - Maximum: 98%
- Margins - Thickness: 11%
- Margins - Screen edge: 15px
- Background - Size: 100%
- Background - Opacity: 75%
- Background - Radius: 50%
- Background - Blur (Enable)
- Background - Shadows (Enable)
