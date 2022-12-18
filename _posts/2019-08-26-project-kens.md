---
title:  "Project KENS/KNS"
date:   2019-08-26 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Python
tags:
   - Python
   - Toy Project
toc: true
toc_sticky: true
breadcrumbs: true
---

# Project KENS

This is a web-scraping project for Konkuk University pages. KNS is a sub-project which was not planned to be made. KNS scraps the main homepage of Konkuk University, where as KENS scraps Electronic Engineering homepage in particular.

## Install

This project was designed for environment of home-server, especially Raspberry-Pi. KENS is a simple mini project, thus it does not provide any special functions, since it does not run on 24-hour running server, but runs by cron scheduler. This section explains how to install the project.

<!--more-->

First download the project to **Desktop/git**

```bash
pi@raspberrypi:~ $ cd Desktop
pi@raspberrypi:~ $ git clone https://github.com/sjoon-oh/project_kens/
```

Make directory below Desktop folder, naming as project_kens_log. This is where log files will be stored. This is highly suggested.

```bash
pi@raspberrypi:~ $ cd Desktop
pi@raspberrypi:~ $ mkdir project_kens_log
```

Now, logs named in the form of 'kens-[year]-[Month]-[Date]-[Hour]-[Minute]' or 'kns-[year]-[Month]-[Date]-[Hour]-[Minute]' will be piled up here.


## Environment Settings

KENS and KNS project are tested on Python 3.6 and 3.7 environment. Raspberry-Pi has Python as default with Raspbian, but make sure your Raspberry-Pi has Python installed.

KENS/KNS uses parsing process with BeautifulSoup4 library. Thus install BeautifulSoup4 with Requests library.

```bash
root@raspberrypi:~# pip3 install bs4
root@raspberrypi:~# pip3 install requests
```

KENS/KNS sends notification to users by Telegram. To run Telegram API without any errors, there should be python-telegram-bot module.

```bash
root@raspberrypi:~# pip3 install python-telegram-bot
```




## Project Settings

Only some script files need to be registered in crontab, not every scripts. The scripts are, **/Rpi/KensRunRpi.py** and **/Rpi/KnsRunRpi.py** .

Open crontab to set those files like below. Doing this by superuser account is highly suggested.

```bash
pi@raspberrypi:~ $ sudo su
root@raspberrypi:~# crontab -e
```

And add the script to run. This is an example of running scripts in every 30 minutes. Set if you like to write logs. In this example, actions of removing logs at a specific time every day was added.

```
# added 19.08.25.

0 * * * * /usr/bin/python3 /home/pi/Desktop/git/project_kens/Rpi/KensRunRpi.py > /home/pi/Desktop/project_kens_log/kens-`date +\%Y-\%m-\%d_\%H:\%M`
2 * * * * /usr/bin/python3 /home/pi/Desktop/git/project_kens/Rpi/KnsRunRpi.py > /home/pi/Desktop/project_kens_log/kns-`date +\%Y-\%m-\%d_\%H:\%M`

30 * * * * /usr/bin/python3 /home/pi/Desktop/git/project_kens/Rpi/KensRunRpi.py > /home/pi/Desktop/project_kens_log/kens-`date +\%Y-\%m-\%d_\%H:\%M`
32 * * * * /usr/bin/python3 /home/pi/Desktop/git/project_kens/Rpi/KnsRunRpi.py > /home/pi/Desktop/project_kens_log/kns-`date +\%Y-\%m-\%d_\%H:\%M`

0 5 * * * rm /home/pi/Desktop/project_kens_log/kens*
1 5 * * * rm /home/pi/Desktop/project_kens_log/kns*
2 5 * * * rm /home/pi/Desktop/project_kens_log/cron*

*/20 * * * * /usr/sbin/service cron status | /bin/grep -E 'Active|Memory' > /home/pi/Desktop/project_kens_log/cron-`date +\%Y-\%m-\%d_\%H:\%M`

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



## How does KENS/KNS stores data?

KENS/KNS is not designed for heavy data handling, and they surely does not have to be. **They don't need databases.** It gets information of the first page each only, thus each function does not handle a large amount of data. Maximum number would be about 10 to 20, so it just stores as files.

Of course, linux system does not have any meaning in extensions, thus the names are just for distinguishing projects.

For example, if some files are named in a pattern of **'*.kens'**, then the files are used for KENS scripts. Same does for KNS scripts.

Now, there will be several texts stored with nothing written. In 0.1.3vb version, there are no exception handling codes, thus **there must be files exactly named as written in the scripts.** The files necessary are listed below.

Files necessary for KENS scripts are, 

- notice.kens
- user_telegram.kens

and for KNS scripts,

- notice_haksa.kns
- notice_janghak.kns
- notice_changup.kns
- notice_guukjae.kns
- notice_sanhak.kns
- notice_ilban.kns

Files named as **'notice_*'** are for the comparing process. In these files, previously scraped information(title, number, date, etc) are stored. The script first opens each files and stores them in _lists_, waiting for the new information to be compared. 

If updated information is found, then it automatically stores the entire list in those files. Note that it ignores any other information like numbers, the numbers of posts seen that are not really worth to send notifications.


## Memory status using cron

Figure below describes memory use checked by *cron service status* command.

![kens_kns_memory_use](/assets/posts/2019-08-26-project-kens/2019-08-26-00.jpg)


## Update logs

2019.08.21. <b>Version: 0.1.0va</b>
- Project KENS initiated, author(SukJoon Oh, acoustikue)

2019.08.24. <b>Version: 0.1.1va</b>
- Telegram ID saving function added. No more receiving only recent ID's, preventing skipping sending messages for random ID's.
- Administrator notification service module added.
- Fixed giving alert every hour, even there's no updated information in the notice board.

2019.08.25. <b>Version: 0.1.3vb</b>
- KNS on KENS project successfully tested. Version upgraded to BETA.
- Hello KNS! New service has been launched. This project is named KNS(Konkuk Univ Notice board Scraper). It is mainly focused on scraping main homepage notice board, where as KENS is only for the EE homepage.
- 0.1.3vb is beta version, so it only sends notification of Haksa and Janghak section, but other tabs will be supported in the later version.
- Directory scanning process added, for convenience in launching server.

2019.08.27. <b>Version: 0.1.6vb</b>
- KNS on KENS project updated! Now more tabs are supported.
- Guukjae, haksaeng, ilban tab support was added. Other remaining tabs need a slight different algorithm, so please wait for later update.
- Note that this service is still BETA, so there might be unexpected actions, for instance KNS catches even a slight modifications of a string in a single title. This will be fixed in the later update.