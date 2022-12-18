---
title:  "아치(Arch) 리눅스 설치기 (업데이트)"
date:   2021-03-29 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Linux
tags:
   - Linux
   - Arch
toc: true
toc_sticky: true
breadcrumbs: true
---

# 아치(Arch) 리눅스 설치기

아치 리눅스는 설치가 까다롭기로 유명합니다. 일반적인 배포판과는 달리 CLI 환경에서 사용자가 직접 설정 해주어야 하기 때문이 아닐까 싶습니다. 그러나 공식 아치 위키(Arch Wiki) 사이트에서 아주 친절하게 설명해주고 있어 영어 장벽만 없다면 무난하게 설치가 가능합니다. 

이 포스트에서는 Arch와 KDE Plasma 조합으로 설치한 경험을 기록하고자 합니다. 참고한 자료는 공식 [ArchWiki](https://wiki.archlinux.org/index.php/installation_guide)와 [WooHyung Jeon님의 한글 번역 가이드](https://whjeon.com/arch-install/)를 참고하였습니다.

<!--more-->

## Arch Installation
### EFI 부팅 확인

UEFI 부팅을 지원하는 경우 파일 리스트가 존재합니다.
```bash
$ ls /sys/firmware/efi/efivars
```

### 인터넷 연결 확인

```bash
$ ping archlinux.org
```

Wi-Fi 무선 인터넷을 사용하는 경우,

```bash
# Install the iwd package. 

$ iwctl # Interactive prompt에 연결
[iwd]$ device list # 본인의 Wireless 드라이버가 설치되어 로드되는지 확인
[iwd]$ station device scan # 네트워크 스캔
[iwd]$ station device get-networks # 사용 가능한 네트워크 리스트 확인
[iwd]$ station (device) connect (SSID) # 네트워크 연결

$ ping archlinux.org # 연결 확인
```

인증 방식을 지정해야 한다면 [iwd 사용법](https://wiki.archlinux.org/index.php/Iwd#iwctl)을 참고하시면 됩니다.



### 네트워크 시간 설정

네트워크 시간과 시스템 시간을 동기화하도록 설정합니다.

```bash
$ timedatectl set-ntp true
$ timedatectl status # Sync와 NTP가 켜져있는지 확인
```

### 파티션 계획

제가 사용한 Thinkpad T540p의 경우 아래와 같이 파티션을 계획했습니다.

| Mount point |	Partition |	Partition type | size |
|---|---|---|---|
| /mnt/efi | /dev/sda1 | EFI system partition | 1G |
| [SWAP] | /dev/sda2 | Linux swap | 24G |
| /mnt | /dev/sda3 | Linux x86-64 root (/) | Remainder |

아래 명령어를 통해 내 드라이브로 어떤 것들이 있는지 확인합니다. 

```bash
$ lsblk
```

결과로 sda, hda와 같은 대용량 저장 장치가 확인됩니다. hda는 기존의 IDE방식 커넥션으로 연결된 하드디스크일 가능성이 크고, sda는 최근의 SCSI/SATA방식으로 연결된 하드/SDD입니다.

대용량 장치가 여러개일 경우 sda,sdb,sdc순으로 출력됩니다.

```bash
$ cfdisk /dev/sda # 경로 지정
$ cfdisk # 또는 이렇게만 입력해도 가능합니다
```

이후 아래와 같이 선택합니다.

```
1. New
2. 용량 크기 지정 : ex. 1G
3. 모든 파티션 Primary로 지정
4. Type 지정 : EFI (+Bootable), Linux Swap, default (Linux Filesystem)
5. Write
6. Quit
```

UEFI환경에서 부팅 파티션은 FAT32 파일시스템을 지정합니다. 아까 EFI시스템으로 파티션을 계획했던 sda1은 fat32로 포맷을 할 것이고, 기본 파일 시스템으로 쓰기로 했던 sda3는 최근 가장 범용으로 쓰이는 ext4파일 시스템으로 포맷합니다. 스왑 파티션은 바로 다음 단계에서 포맷합니다. /dev/sda2를 포맷하지 않습니다.

```bash
$ mkfs.ext4 /dev/sda3
$ mkfs.fat -F32 /dev/sda1 

$ mkdwap /dev/sda2
$ swapon /dev/sda2

# 마운트
$ mount /dev/sda3 /mnt
$ mkdir /mnt/efi
$ mount /dev/sda1 /mnt/efi
```

### 패키지 설치

우선 가장 기본적인 패키지를 설치합니다.

```bash
$ pactrap /mnt base base-devel linux linux-firmware networkmanager sudo vim man-db man-pages
```

### 아치 리눅스 (Root) 진입

```bash
$ genfstab -U /mnt >> /mnt/etc/fstab
```

파일 시스템이 어떤 드라이브와 마운트 되어있는지를 알 수 있는 파일 시스템 테이블 파일을 생성합니다. 시스템 사용중에도 마운트 된 디바이스, UUID를 확인할 때 사용하는 파일입니다.

```bash
$ arch-chroot /mnt
```

시간을 설정합니다.
```bash
$ ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime
$ hwclock --systohc
```

루트 비밀번호를 생성합니다.

```bash
$ passwd
```

시스템의 이름을 설정하기 위해 /etc/hostname을 수정합니다. 

```bash
$ echo "mypc" > /etc/hostname
```

시스템의 로컬 라우팅 테이블을 설정합니다. /etc/hosts 파일에 아래와 같이 수정합니다.

```
127.0.0.1	localhost
::1		localhost
127.0.1.1	[mypc].localdomain	[mypc]
```


일반 사용자 설정
```bash
$ useradd -m -g users -G wheel -s /bin/bash (USER)
$ passwd (USER)

# Sudo가 설치되지 않은 경우 설치
$ pacman -S sudo
$ vim /etc/sudoers
```

관리자 권한에 진입하기 위해 설정한 사용자 이름으로 아래와 같이 추가합니다.
```
USER ALL=(ALL) ALL
```

### Bootloader 설치

```bash
$ pacman -Sy
$ pacman -S grub efibootmgr

$ grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=arch
$ grub-mkconfig -o /boot/grub/grub.cfg
```

### 네트워크 사용 설정

재부팅 시 네트워크 연결이 가능하도록 설정합니다.

```bash
$ systemctl enable NetworkManager.service
```

### 재부팅

```bash
$ exit
$ reboot
```

## KDE Installation

```bash
# Option 1 Full Installation
# $ pacman -S --needed xorg sddm
# $ pacman -S --needed plasma kde-application

# Option 2 Lightweight Installation
$ pacman -Syu
$ pacman -S xorg-server plasma konsole dolphin ark firefox gwenview spectacle

# Dev tools
$ pacman -S git kdevelop visual-studio-code-bin octave

# Korean Font
$ pacman -S adobe-source-han-sans-kr-fonts
$ pacman -S fcitx-im fcitx-hangul kcm-fcitx

# Office tools
$ pacman -S okular libreoffice-still

# Java
$ pacman -S jdk8-openjdk

# Telegram
$ pacman -S telegram-desktop

$ systemctl enable sddm
$ reboot
```

## 추가 유틸리티 설치 (pacman)

아래 유틸리티는 제가 유용하게 사용하는 프로그램 모음입니다.

### Microcode
```bash
$ pacman -S intel-ucode
$ grub-mkconfig -o /boot/grub/grub.cfg
```

### Gwenview (이미지 뷰어)
```bash
$ pacman -S gwenview
```

### Latte Dock
```bash
$ pacman -S latte-dock
```

### Darktable & GIMP (사진 편집 프로그램)
```bash
$ pacman -S darktable gimp
```

### NordVPN
```bash
$ yay -S nordvpn-bin
$ groupadd -r nordvpn
$ gpasswd -a [USERNAME] nordvpn

$ nordvpn login
$ nordvpn set technology nordlynx # Enable NordLynx(Wireguard)
$ nordvpn connect [[country]/[server]/[country_code]/[city] or [country] [city]]
# for example: nordvpn connect Japan

$ nordvpn status

$ nordvpn disconnect
$ nordvpn logout
```

### Kdenlive & OBS Studio & Spectacle
```bash
$ pacman -S obs-studio
$ pacman -S kdenlive
$ pacman -S spectacle # 스크린샷
```
### Docker
```bash
$ pacman -S docker
$ sudo useradd -aG docker (사용자 이름)

$ systemctl start docker.service
$ systemctl enable docker.service
```

## AUR
### yay 설치 
```bash
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -si
```

### Brave Browser
```bash
$ yay -S brave-bin
```

### Bitwarden
```bash
$ yay -S bitwarden-bin
```

### Freetube
```bash
$ yay -S freetube-bin
```

### pCloud Drive
```bash
$ yay -S pcloud-drive
```

### Timeshift
```bash
$ git clone https://aur.archlinux.org/timeshift.git
$ cd timeshift
$ makepkg -si

$ sudo timeshift –create –comments “Fresh install”
$ sudo timeshift –restore
```

### Nerd Font
```bash
$ yay -S nerd-fonts-jetbrains-mono
```
가장 좋아하는 font 입니다.

그러면 험난한 아치 리눅스 설치가 완료됩니다.

![screenshot](/assets/posts/2021-03-29-arch-installation/installed2.png)