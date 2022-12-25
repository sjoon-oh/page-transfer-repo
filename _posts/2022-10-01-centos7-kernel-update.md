---
title:  "[노트] CentOS7 Kernel Update & Boot"
date:   2022-10-01 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
   show_overlay_excerpt: false
categories: 
   - Research
tags:
   - CentOS7
   - Operating System
toc: true
toc_sticky: true
breadcrumbs: true
---

# [노트] CentOS7 Kernel Update

*이 포스트는 OSLab 연구 활동에서 작성한 개인 노트의 일부입니다. 공부 목적으로 작성된 내용이므로 많이 부족할 수 있습니다.*

## Installation

이 글에서는 RPM 으로 제공되는 커널 설치에 대해 다루도록 한다.

### Repository Add

아래의 명령을 실행하여 `elrepo-kernel` 레포지토리를 추가한다. 지원하는 커널의 종류는 다음과 같다.

- kernel-lt : Long-term Support 커널
- kernel-ml : Mainline 커널 (Stable)

```bash
$ sudo yum -y install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
$ sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

다음은 설치 가능한 커널을 확인한다.

```bash
$ sudo yum --disablerepo="*" --enablerepo="elrepo-kernel" list available | grep kernel-ml
```

### Mirror Installation

다음과 같은 버전의 커널을 설치하기 위해서 아래 과정을 따른다.

```bash
kernel-ml.x86_64                    5.11.1-1.el7.elrepo          @elrepo-kernel
kernel-ml-devel.x86_64              5.11.1-1.el7.elrepo          @elrepo-kernel
kernel-ml-headers.x86_64            5.11.1-1.el7.elrepo          @elrepo-kernel
```

```bash
$ wget https://linux.cc.iitk.ac.in/mirror/centos/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-5.11.1-1.el7.elrepo.x86_64.rpm
$ wget https://linux.cc.iitk.ac.in/mirror/centos/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-devel-5.11.1-1.el7.elrepo.x86_64.rpm
$ wget https://linux.cc.iitk.ac.in/mirror/centos/elrepo/kernel/el7/x86_64/RPMS/kernel-ml-headers-5.11.1-1.el7.elrepo.x86_64.rpm
$ sudo yum localinstall *rpm
```

`kernel-headers` 패키지와 충돌이 발생하는 경우 `kernel-ml-header`는 제외하고 설치한다.

참고: [https://elrepo.org/tiki/kernel-ml](https://elrepo.org/tiki/kernel-ml)

### Grub2 Entries

Grub2의 기본 설정 파일은 `/etc/default/grub2` 에 존재한다.

```
GRUB_TIMEOUT=5
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

이 영역에 대한 수정을 반영하기 위해서는 다음을 실행한다.

```bash
$ grub2-mkconfig -o /boot/grub2/grub.cfg ## Legacy BIOS
$ grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg ## UEFI System
```

Grub2 에 등록된 메뉴 엔트리를 출력하기 위해서는 다음을 실행한다.

```bash
$ grep "^menuentry" /boot/grub2/grub.cfg | cut -d "'" -f2
$ grep "^menuentry" /boot/efi/EFI/centos/grub.cfg | cut -d "'" -f2
## UEFI 시스템에서는 경로 명이 다를 수 있다.

## 또는,
$ awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
```

현재 기본으로 등록된 엔트리는 다음으로 확인한다.

```bash
$ grub2-editenv list
saved_entry=CentOS Linux (3.10.0-1160.11.1.el7.x86_64) 7 (Core)
```

위에서 확인한 인덱스 기반으로 새 엔트리를 설정한다.

```bash
$ grub2-set-default "CentOS Linux (5.11.1-1.el7.elrepo.x86_64) 7 (Core)"
$ grub2-editenv list
```

<!-- ## MLNX_OFED Driver Reinstallation

---

우선 다음과 같이 기존 설치된 드라이버를 제거한다.

```bash
$ sudo /usr/sbin/ofed_uninstall.sh
```

CentOS 계열 el Release 커널은 `--add-kernel-support` 옵션을 지정해야 한다. 해당 드라이버를 [다운로드](https://www.mellanox.com/products/infiniband-drivers/linux/mlnx_ofed) 후 아래와 같이 인스톨러를 실행한다.

```bash
$ sudo ./mlnxofedinstall --add-kernel-support [--without-fw-update]

## 재부팅 후 드라이버 로드
$ sudo /etc/init.d/openibd restart
```

설치 진행이 안되는 경우 다음을 시도한다.

- `GCC` 버전이 지원되는지 로그를 통해 확인.
    
    e.g. 5.11-1 커널은 `GCC 9` 를 필요로 한다.
    
- `--force` 옵션 시도. -->

## Reference

---

- [https://www.tecmint.com/install-upgrade-kernel-version-in-centos-7/](https://www.tecmint.com/install-upgrade-kernel-version-in-centos-7/)
- [https://computingforgeeks.com/install-linux-kernel-5-on-centos-7/](https://computingforgeeks.com/install-linux-kernel-5-on-centos-7/)
- [https://fasterdata.es.net/host-tuning/linux/recent-tcp-enhancements/elrepo-kernel/](https://fasterdata.es.net/host-tuning/linux/recent-tcp-enhancements/elrepo-kernel/)
- [https://linux.cc.iitk.ac.in/mirror/centos/elrepo/kernel/el7/x86_64/RPMS/](https://linux.cc.iitk.ac.in/mirror/centos/elrepo/kernel/el7/x86_64/RPMS/)
- [https://m.blog.naver.com/hymne/221000701554](https://m.blog.naver.com/hymne/221000701554)
- [https://wiki.centos.org/HowTos/Grub2](https://wiki.centos.org/HowTos/Grub2)