# About this document
This document is an open source markdown document that can be contributed to via github.
If you see a typo, an improvment to add, or a vector that I missed please send a pull request to the master brunch via the 
[repo link](https://github.com/AddaxSoft/OSWindowsPrivEscalation) and I will review it and approve if approperiate.

This document is meant for pen-testers, red teams, and the like.

** Needless to state: You're responosible for what you're doing :-)

# Contributors
- AK | [amAK.xyz](https://imAK.xyz)

------
Let's get to it!

# OS Enumurations
In this stage you want to learn as much as possible about the operating system.
Note any odd things and investigate them until you hit a dead-end, then do the next thing.

## Windows Version and Configuration
What Windows is it, what version?

    systeminfo | findstr /B /C:"OS Name" /C:"OS Version"

What architecture? x86 or x64?

    wmic os get osarchitecture
    echo %PROCESSOR_ARCHITECTURE%

List all env variables

    set
    
## Users Enumuration
Get current username

    echo %USERNAME%
    whoami

List all users 

    net user
    whoami /all

List logon requirments; useable for bruteforcing

    net accounts

Get details about a user (i.e. administrator, admin, current user)

    net user administrator
    net user admin
    net user %USERNAME%

List all local groups

    net localgroup 

Get details about a group (i.e. administrators)

    net localgroup administrators


## Network Enumuration
You will want to know how this host is connected; what kind of protocls and services are running, and finally maybe even tap into one of the interfaces and learn what's going on


List all network interfaces

    ipconfig /all

List current routing table

    route print

List the ARP table

    arp -A

List all current connections

    netstat -ano

List firware state and current configuration

    netsh advfirewall firewall dump

List all network shares

    net share

# Looting any clear text passwords
Many admins will store clear-text passwords on the file system. Your target is usually xml, txt, xls files that have the word pass/password on them.

## Searching in files
Quick peek into common password files

note: If you found encrypted contents; decrypt them with gpprefdecrypt.py
encoded passwords are decoded using base64

    TYPE c:\sysprep.inf
    TYPE c:\sysprep\sysprep.xml
    TYPE %WINDIR%\Panther\Unattend\Unattended.xml
    TYPE %WINDIR%\Panther\Unattended.xml

Search for file contents

    cd C:\ & findstr /SI /M "password" *.xml *.ini *.txt

Search for a file with a certain filename

    dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*

## Searching in Registery
Search the registery for key names

    REG QUERY HKLM /F "password" /t REG_SZ /S /K
    REG QUERY HKCU /F "password" /t REG_SZ /S /K
    
Search the registery for any clear text passwords in key values

note: value of each key will be printed out too

    REG QUERY HKLM /F "password" /t REG_SZ /S
    REG QUERY HKCU /F "password" /t REG_SZ /S

Read a vlue of a certain sub key

    REG QUERY "HKLM\Software\Microsoft\FTH" /V RuleList


## Processes Enum
What processes are running?

    tasklist /v

Which processes are running as "system"

    tasklist /v /fi "username eq system"


-----

# Special Thanks & Original Inspirations 
- This document wouldn't be here if I didn't get some inspirations:
  - fuzzysecurity's ultimate guide for Windows Privilge escalation, which can be found under this [link](http://www.fuzzysecurity.com/tutorials/16.html).
  - g0tmi1k's Basic Linux Privilege Escalation which can be found under this [link](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)
  - Peter Kim's Hackers Playbook 2 - Zero to Hero section [link](http://thehackerplaybook.com/dashboard/)
- Offensive Security, which pushed me really hard beyod my limitations during the many hours of training. 
