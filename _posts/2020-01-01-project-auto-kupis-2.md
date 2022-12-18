---
layout: post
title:  "Project AutoKupis2"
date:   2020-01-01 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Python
tags:
   - Python
   - Toy Project
toc: true
toc_sticky: true
---
# Project AutoKupis2

Project AutoKupis2, Auto-sugang GUI version of AutoKupis project(private).

This is a simple reqeust sender for Konkuk University class registering site. The project name comes from the registration server URL(*kupis.konkuk.ac.kr*). 

Unlike AutoKupis project, all sources are open to public, thus anyone can use, recompile, modify and redistribute. But please mark the original author when you use it.

## Use

You can compile the source by yourself. You can compile it with **pyinstaller**. The command I used was, 

<!--more-->

```
pyinstaller --onefile --noconsole ak2.py --icon=C:\Users\acoustikue\Documents\Git\auto_kupis_2\ui\auto_kupis_2.ico
```

Or you can just download compiled file in **build** folder in this repository. The directory contains ak2.exe file and geckodriver.exe. **geckodriver file must be located under the webdriver folder**, otherwise the program will not work.

![demo](/assets/posts/2020-01-01-project-auto-kupis-2/capture.png)

## Update log
- 0.1.0va: Initial commit, extremely experimental
- 0.1.4v : Fixed thread error. Added ak2_worker module.


