---
layout: post
title:  "[메모] PyTorch Windows 10 Anaconda 에서 사용하기"
date:   2020-11-24 13:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Python
tags:
   - Python
toc: true
toc_sticky: true
---

# PyTorch - Anaconda/Jupyter 에서 사용하기

## 가상 환경 설정

파이썬 3.8 기준, CPU 모드로 설정하는 경우 입니다. CUDA를 이용하는 경우 공식 홈페이지에 자세히 설치 방법이 나와있으니 참고하시면 될 것 같습니다. 

```bash
(base) path> conda create -n pytorch python=3.8  
(base) path> conda activate pytorch  
(pytorch) path> conda install pytorch torchvision cpuonly -c pytorch 
```

## 필요 모듈 설치

```bash
(pytorch) path> conda install jupyter pandas matplotlib  
(pytorch) path> conda install -c conda-forge jupyter_contrib_nbextensions   
(pytorch) path> conda install scikit-learn  
(pytorch) path> jupyter notebook  
```

## 가상 환경 나가기

```bash
(pytorch) path> conda deactivate 
```


