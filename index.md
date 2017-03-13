# About this document
This document is an open source markdown document that can be contributed to via github.
If you see a typo, an improvment, or something to add please send a pull request to the master brunch and I will add it.
[repo link](https://github.com/AddaxSoft/OSWindowsPrivEscalation)
This document is meant for pen-testers, red teams, and the like.
Needless to state: You're responosible for what you're doing :-)

# Special Thanks
- This document wouldn't be here if I didn't get some inspirations:
  - fuzzysecurity's ultimate guide for Windows Privilge escalation, which can be found under this [link](http://www.fuzzysecurity.com/tutorials/16.html).
  - g0tmi1k's Basic Linux Privilege Escalation which can be found under this [link](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)
- Offensive Security, which pushed me really hard beyod my limitations during the many hours of training. 

# Contributors
- AK | [amAK.xyz](https://imAK.xyz)

------

# Let's get to it!

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

## Processes Enum
What processes are running?

    tasklist /v

Which processes are running as "system"

    tasklist /v /fi "username eq system"
    
## Users Enumuration
Get current username

    whoami
    echo %USERNAME%

List all users 

    net user

Get details about a user

    net user administrator

## Network Enumuration
List all network interfaces

    ipconfig /all

List all current connections

    netstat -ano
