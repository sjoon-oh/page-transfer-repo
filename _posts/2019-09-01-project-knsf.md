---
title:  "Project KNSF"
date:   2019-09-01 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
   show_overlay_excerpt: false
categories: 
   - Python
tags:
   - Python
   - Toy Project
toc: true
toc_sticky: true
breadcrumbs: true
---

# Project KNSF

Project KNSF, KENS/KNS extended version.

This is a web-scraping project for Konkuk University pages. KNSF stands for **K**onkuk University **N**otice Board **S**craper with **F**CM. This includes previous project KENS and KNS which was based on Telegram-bot. 

KNS scraps the main homepage of Konkuk University, where as KENS scraps Electronic Engineering homepage in particular. KNSF refactored many of the source codes.

![demo](/assets/posts/2019-09-01-project-knsf/demo.gif)

Specially thanks to Su Chan Kim for testing.

<!--more-->

## Install

This project was designed for environment of home-server, especially Raspberry-Pi. KENS is a simple mini project, thus it does not provide any special functions, since it does not run on 24-hour running server, but runs by cron scheduler. This section explains how to install the project.



First download the project to any directory.

```bash
pi@raspberrypi:~ $ cd Desktop
pi@raspberrypi:~ $ git clone https://github.com/sjoon-oh/knsf
```

Make any directory for logs. This is an option.

```bash
pi@raspberrypi:~ $ cd Desktop
pi@raspberrypi:~ $ mkdir knsf_log
```

<!-- Now, logs named in the form of 'kens-[year]-[Month]-[Date]-[Hour]-[Minute]' or 'kns-[year]-[Month]-[Date]-[Hour]-[Minute]' will be piled up here. -->



## Environment Settings

Along with KENS and KNS project, KNSF is also tested on Python 3.6 and 3.7 environment. Raspberry-Pi has Python as default with Raspbian, but make sure your Raspberry-Pi has Python installed.

KNSF uses parsing process with BeautifulSoup4 library. Thus install BeautifulSoup4 with Requests library.

```bash
root@raspberrypi:~# pip3 install bs4
root@raspberrypi:~# pip3 install requests
```

KNSF sends notification by FCM service. It uses pyfcm library. So install it by,

```bash
root@raspberrypi:~# pip3 install pyfcm
```

or
```bash
root@raspberrypi:~# pip install git+https://github.com/olucurious/PyFCM.git
```



## Project Settings

Unlike KENS/KNS, the script files are located under **/knsf_module/run_script** directory. Also each script scraps a single page, whereas KNS scrapes five different pages in a single script-execution.

Since KNSF is also designed for a small-scale home server, KNSF will run repeatedly by cron service. Open crontab to set those files like below. Doing this with superuser is highly suggested.

```bash
pi@raspberrypi:~ $ sudo su
root@raspberrypi:~# crontab -e
```

And add the script to run. This is an example of running scripts in every 30 minutes. Set if you like to write logs. In this example, actions of removing logs at a specific time every day was added.

```
# added 19.08.31.
0/10 * * * * /usr/bin/python3 /home/pi/Desktop/git/knsf/knsf_module/run_script/KnsfKensRun.py > /home/pi/Desktop/knsf_log/knsf-kens-`date +\%Y-\%m-\%d_\%H:\%M`
0/10 * * * * /usr/bin/python3 /home/pi/Desktop/git/knsf/knsf_module/run_script/KnsfKnsGuukjaeRun.py > /home/pi/Desktop/knsf_log/knsf-kns-guukjae-`date +\%Y-\%m-\%d_\%H:\%M`
0/10 * * * * /usr/bin/python3 /home/pi/Desktop/git/knsf/knsf_module/run_script/KnsfKnsHaksaengRun.py > /home/pi/Desktop/knsf_log/knsf-kns-haksaeng`date +\%Y-\%m-\%d_\%H:\%M`
0/10 * * * * /usr/bin/python3 /home/pi/Desktop/git/knsf/knsf_module/run_script/KnsfKnsHaksaRun.py > /home/pi/Desktop/knsf_log/knsf-kns-haksa-`date +\%Y-\%m-\%d_\%H:\%M`
0/10 * * * * /usr/bin/python3 /home/pi/Desktop/git/knsf/knsf_module/run_script/KnsfKnsIlbanRun.py > /home/pi/Desktop/knsf_log/knsf-kns-ilban-`date +\%Y-\%m-\%d_\%H:\%M`
0/10 * * * * /usr/bin/python3 /home/pi/Desktop/git/knsf/knsf_module/run_script/KnsfKnsJanghakRun.py > /home/pi/Desktop/knsf_log/knsf-kns-janghak-`date +\%Y-\%m-\%d_\%H:\%M`

0 5 * * * rm /home/pi/Desktop/knsf_log/*

*/20 * * * * /usr/sbin/service cron status | /bin/grep -E 'Active|Memory' > /home/pi/Desktop/knsf_log/cron-`date +\%Y-\%m-\%d_\%H:\%M`

```

Now run the cron service.

```bash
root@raspberrypi:~# service cron start
```

To check the service whether it is running,

```bash
root@raspberrypi:~# service cron status
```


If you want to stop the script, then write the command like below.

```bash
root@raspberrypi:~# service cron stop
```

Examine whether the logs are piling up.



## How does KNSF stores data?

Like KENS/KNS, KNSF is also not designed for heavy data handling, and they surely does not have to be. **They don't need databases.** It gets information of the first page each only, thus each function does not handle a large amount of data. Maximum number would be about 10 to 20, so it just stores as files.

In KENS/KNS project, there were some errors in file handling. If there were no files in exact directory which was hard-coded in the source, it stopped with error logs. Unlike KENS/KNS, KNSF handles files at the initial state of executing scripts. If there are no files detected, it makes blank file, so that there are always right files in the directory. 

Files KNSF are located under **/knsf_module/db** and names are as follows:

- DB_CHANGUP.KNSF
- DB_EE.KNSF
- DB_GUUKJAE.KNSF
- DB_HAKSA.KNSF
- DB_HAKSAENG.KNSF
- DB_ILBAN.KNSF
- DB_JANGHAK.KNSF
- DB_SANHAK.KNSF

These files holds parsed information from each pages in **JSON** form. 
Information about FCM service is stored in the following:

- FCM_MESSAGES.KNSF
- DB_FCM_SERVER_KEY.KNSF
- DB_FCM_USER.KNSF


Files named as **'DB_*'** under under **/knsf_module/db** are for the comparing process. In these files, previously scraped information(title, number, date, etc) are stored. The script first opens each files and stores them in customized container object, which works like DAOs in a common Spring project.

If updated information is found, then it automatically stores the entire information in those files. Note that it ignores any other information like numbers, the numbers of posts seen that are not really worth to send notifications.

Information essential for FCM push is saved in DB_FCM_SERVER_KEY.KNSF and DB_FCM_USER.KNSF. Be sure to write server API key to **DB_FCM_SERVER_KEY.KNSF** and user tokens in **DB_FCM_USER.KNSF** file. They are not in the JSON form, but a plain texts.

Note that all files are encoded in UTF-8.



## Update logs

2019.08.28. <b>Version: 0.1.0va</b>
- Project KNSF initiated, author(SukJoon Oh, acoustikue)
