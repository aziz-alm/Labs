# HackTheBox - SolidState

## Nmap Scan

Starting off with a full scan using `nmapAutomator` to get a comprehensive view of what's running.

```
./nmapAutomator.sh 10.129.96.167 -t All

Running all scans on 10.129.96.167

Host is likely running Linux


---------------------Starting Port Scan-----------------------



PORT     STATE SERVICE
22/tcp   open  ssh
8443/tcp open  https-alt



---------------------Starting Script Scan-----------------------



PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 fc:fb:90:ee:7c:73:a1:d4:bf:87:f8:71:e8:44:c6:3c (RSA)
|   256 46:83:2b:1b:01:db:71:64:6a:3e:27:cb:53:6f:81:a1 (ECDSA)
|_  256 1d:8d:d3:41:f3:ff:a4:37:e8:ac:78:08:89:c2:e3:c5 (ED25519)
8443/tcp open  ssl/http Golang net/http server
|_http-title: Site doesn't have a title (application/json).
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=minikube/organizationName=system:masters
| Subject Alternative Name: DNS:minikubeCA, DNS:control-plane.minikube.internal, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:kubernetes.default, DNS:kubernetes, DNS:localhost, IP Address:10.129.96.167, IP Address:10.96.0.1, IP Address:127.0.0.1, IP Address:10.0.0.1
| Not valid before: 2026-06-28T20:45:48
|_Not valid after:  2029-06-28T20:45:48
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: 58b72975-c8e7-4538-a3c4-81350d74f064
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 8914d77e-677e-4100-9ede-5a05f57c9f0f
|     X-Kubernetes-Pf-Prioritylevel-Uid: c45a50ba-4b84-4687-8b35-38a070d41a45
|     Date: Mon, 29 Jun 2026 20:48:30 GMT
|     Content-Length: 212
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/nice ports,/Trinity.txt.bak"","reason":"Forbidden","details":{},"code":403}
|   GetRequest: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: 30151b30-d40e-4f34-9446-c34f05e66099
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 8914d77e-677e-4100-9ede-5a05f57c9f0f
|     X-Kubernetes-Pf-Prioritylevel-Uid: c45a50ba-4b84-4687-8b35-38a070d41a45
|     Date: Mon, 29 Jun 2026 20:48:27 GMT
|     Content-Length: 185
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/"","reason":"Forbidden","details":{},"code":403}
|   HTTPOptions: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: 731d6f12-45dc-4e2c-bd85-b6043bd9eef5
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 8914d77e-677e-4100-9ede-5a05f57c9f0f
|     X-Kubernetes-Pf-Prioritylevel-Uid: c45a50ba-4b84-4687-8b35-38a070d41a45
|     Date: Mon, 29 Jun 2026 20:48:29 GMT
|     Content-Length: 189
|_    {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot options path "/"","reason":"Forbidden","details":{},"code":403}
|_ssl-date: TLS randomness does not represent time
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel




---------------------Starting Full Scan------------------------
                                                                                                                                                                                                                                            


PORT      STATE SERVICE
22/tcp    open  ssh
2379/tcp  open  etcd-client
2380/tcp  open  etcd-server
8443/tcp  open  https-alt
10250/tcp open  unknown
10256/tcp open  unknown



Making a script scan on extra ports: 2379, 2380, 10250, 10256
                                                                                                                                                                                                                                            


PORT      STATE SERVICE          VERSION
2379/tcp  open  ssl/etcd-client?
| tls-alpn: 
|_  h2
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=steamcloud
| Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.129.96.167, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
| Not valid before: 2026-06-29T20:45:49
|_Not valid after:  2027-06-29T20:45:50
2380/tcp  open  ssl/etcd-server?
| ssl-cert: Subject: commonName=steamcloud
| Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.129.96.167, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
| Not valid before: 2026-06-29T20:45:49
|_Not valid after:  2027-06-29T20:45:50
| tls-alpn: 
|_  h2
|_ssl-date: TLS randomness does not represent time
10250/tcp open  ssl/http         Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
| ssl-cert: Subject: commonName=steamcloud@1782765951
| Subject Alternative Name: DNS:steamcloud
| Not valid before: 2026-06-29T19:45:51
|_Not valid after:  2027-06-29T19:45:51
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
10256/tcp open  http             Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
```

So we've got SSH (22), a Kubernetes API on 8443, etcd on 2379/2380, Kubelet on 10250, and kube-proxy health check on 10256. Quite a few services to poke at. Let's start with the web side.

---

## Port 80

The website is just a static page for "Solid State Security" nothing dynamic or exploitable at first glance.

![Port 80 — Solid State Security homepage](../HTB_imgs/Pasted%20image%2020260630232859.png)

### Feroxbuster

Ran feroxbuster with the DirBuster medium wordlist to look for hidden directories or files.

![Feroxbuster output — nothing interesting](../HTB_imgs/Pasted%20image%2020260630233554.png)

Nothing useful came back just static assets and images. Let's move on and come back if needed.

---

## Port 4555

Port 4555 is running **JAMES Remote Administration Tool 2.3.2**  Apache James is a mail server. I connected with `nc` and tried the default credentials `root:root`, and they worked straight away.

![JAMES Admin Tool — default creds worked, listing users and resetting passwords](../HTB_imgs/Pasted%20image%2020260630234824.png)

From here I listed all users (`listusers`) and found 5 accounts: james, thomas, john, mindy, and mailadmin. Since we're root on the admin tool, we can use `setpassword` to reset any user's password. I reset all of them so I could log into POP3 and read their emails.

---

## POP3

Connected to POP3 on port 110 using `telnet` and started checking each user's inbox. Most had nothing interesting, but **mindy's** mailbox had an email from mailadmin with her SSH credentials in plaintext.

![POP3 — mindy's email containing SSH credentials](../HTB_imgs/Pasted%20image%2020260630235119.png)

The email contained `username: mindy` and `pass: P@55W0rd1!2@`. It also mentioned that her access is restricted  good to know for later.

---

## SSH

Used the creds from the email to SSH in as mindy, and it worked.

![SSH login as mindy](../HTB_imgs/Pasted%20image%2020260630235318.png)

---

# User Flag

![User flag](../HTB_imgs/Pasted%20image%2020260630235439.png)

---

# Privilege Escalation

## Transferring Tools

First thing I do after landing on a box is transfer some enumeration tools. Spun up a Python HTTP server on my Kali machine pointing to my Linux privesc toolkit.

![HTTP server hosting privesc tools](../HTB_imgs/Pasted%20image%2020260630235715.png)

## Restricted Shell (rbash)

However, I quickly realised we're in an **rbash** (restricted bash) shell. Commands like `wget`, `curl`, and `cd` are all blocked. This is exactly what the email warned about "your access is restricted."

![rbash — restricted environment, most commands blocked](../HTB_imgs/Pasted%20image%2020260701000101.png)

## Escaping rbash

Tried several bypass techniques but the one that worked was passing `bash --noprofile` as the command during SSH login. This skips the restricted profile and drops us into a proper bash shell:

```bash
ssh mindy@10.129.107.137 -t "bash --noprofile"
```

![rbash bypass — SSH with bash --noprofile](../HTB_imgs/Pasted%20image%2020260701000514.png)

Now we can `cd` freely and use all commands normally.

## Finding the Cron Job

After transferring **pspy32** (a tool that monitors processes without needing root), I could see that `/opt/tmp.py` is being executed by root (UID=0) every minute via a cron job.

![pspy32 output — root running /opt/tmp.py every minute](../HTB_imgs/Pasted%20image%2020260701004807.png)

## Checking Permissions

Checked the permissions on `/opt/tmp.py` and it's **world-writable** (`-rwxrwxrwx`). This means we can modify a file that root executes every minute — that's game over.

![/opt/tmp.py — world-writable, owned by root](../HTB_imgs/Pasted%20image%2020260701004916.png)

## Crafting the Payload

I wrote a simple Python script that sets the SUID bit on `/bin/bash`, so we can run bash as root:

```python
import os
os.system("chmod +s /bin/bash")
```

![Payload script — chmod +s /bin/bash](../HTB_imgs/Pasted%20image%2020260701005456.png)

Transferred the script to the target and overwrote `/opt/tmp.py` with it.

![Transferring and overwriting tmp.py with our payload](../HTB_imgs/Pasted%20image%2020260701005917.png)

## Waiting for Execution

Now we just wait for the cron job to fire. After about a minute, checked the permissions on `/bin/bash` and the SUID bit was set (`-rwsr-sr-x`).

![/bin/bash now has SUID bit set](../HTB_imgs/Pasted%20image%2020260701010229.png)

## Root

Ran `/bin/bash -p` to get a root shell and grabbed the root flag.

![Root shell — whoami returns root](../HTB_imgs/Pasted%20image%2020260701010330.png)
