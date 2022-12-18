---
layout: post
title:  "Project YNSF"
date:   2020-03-22 09:00:00
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

# Project YNSF

Project YNSF, KNSF extended, and refactored version.

This is a web-scraping project for Yonsei University pages. YNSF stands for **Y**onsei University **N**otice Board **S**craper with **F**CM. This is based on KNSF project, and added some additional functions.

YNSF is mainly targeted for students who major in Electrical and Electronic Engineering(EE). It scraps the main homepage of Yonsei University, EE page, and sends notification if there is any change or update in notice.

![demo](/assets/posts/2020-03-22-project-ynsf/ynsf_screenshot.jpg)

<!--more-->

## Install

This project was designed for environment of AWS Lightsail. This does not provide any complicated functions, and must be set via cron service.

This is a private project, thus environment settings will not be opened here.


## Environment Settings

This is a private project, thus environment settings will not be opened here.


## Database

Unlike KNSF or some other previous projects, YNSF has been updated to use the actual database, especially MongoDB. MongoDB stores data in json form, 

```python
# Database design(main site)
# 1. site: "https://www.yonsei.ac.kr/sc/support/notice.jsp"
# 2. title: "2020학년도 1학기 선택교양 일몰 교과목 재수강 지정 목록 및 신청 안내"
# 3. sign: "신촌/국제 학부대학 2020.02.04"
# 4. url: "https://www.yonsei.ac.kr/sc/support/notice.jsp?mode=view&article_no=181932&board_wrapper=%2Fsc%2Fsupport%2Fnotice.jsp&pager.offset=0&board_no=15"

{
    "site": "https://www.yonsei.ac.kr/sc/support/notice.jsp", 
    "title": "2020학년도 1학기 선택교양 일몰 교과목 재수강 지정 목록 및 신청 안내", 
    "sign": "신촌/국제 학부대학 2020.02.04", 
    "url": "https://www.yonsei.ac.kr/sc/support/notice.jsp?mode=view&article_no=181932&board_wrapper=%2Fsc%2Fsupport%2Fnotice.jsp&pager.offset=0&board_no=15"}

```

This is how to control MongoDB service.

```bash
$ service mongod start # start db
$ service mongod status # show status
$ service mongod stop # stop db
$ service mongod restart # restart db
```

When accessing to MongoDB console, 

```bash
$ mongo -u username -p 'your_password'
```

When you're new to MongoDB, first create user, in order to control MongoDB with pymongo library.

```bash
$ use admin
$ db.createUser( { user: "username", pwd: "your_password", roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] })
```

Now to add or remove elements from database through console, 

```bash
$ use ynsf
$ db.notice.find({})
$ db.notice.remove({}) # remove by giving conditons
$ db.notice.insert({}) # insert your element

```


## Update logs

2020.01. <b>Version: 0.1.0va</b>
- Project YNSF initiated, author(SukJoon Oh, acoustikue)
