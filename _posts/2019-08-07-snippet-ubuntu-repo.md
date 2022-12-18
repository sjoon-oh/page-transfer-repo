---
title:  "구름IDE 초기 설정"
date:   2019-08-07 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
   show_overlay_excerpt: false
categories: 
   - Snippet
tags:
   - Snippet
   - Ubuntu
toc: true
toc_sticky: true
breadcrumbs: true
---

# Ubuntu Repository 

### 구름 IDE repository?

우분투 16.04 LTS 컨테이너를 생성하면 왠지 모르게 업그레이드가 안되는 애들이 있습니다다. 몇 번 삽질한 결과 다음과 같은 방법으로 해결하면 될 듯 합니다. 아래를 추가합시다.

<!--more-->


```bash
vi /etc/apt/sources.list
```

위의 파일을 열고 다음을 추가합니다.

```bash
deb http://archive.ubuntu.com/ubuntu bionic main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-security main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-updates main restricted universe multiverse
```

그리고 갱신합니다.

```bash
sudo apt-get update
sudo apt-get upgrade
```


### 콘솔 상의 Selenium을 위한 Firefox Headless 세팅 19.08.10.

```bash
sudo apt-add-repository ppa:mozillateam/firefox-next
```

다음을 설치합니다.

```bash
sudo apt-get install firefox xvfb
```

가상 디스플레이를 설정해줍니다.


```bash
Xvfb :10 -ac &
export DISPLAY=:10
```

파이어폭스 실행 후 프로세스가 돌아가는 것을 확인합니다.
```bash
firefox
```


