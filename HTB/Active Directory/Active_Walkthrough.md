# HackTheBox — Active

Active is a Windows Active Directory box that covers two classic AD attack techniques: **GPP password decryption** and **Kerberoasting**. A great intro to AD pentesting.

## Nmap

Standard nmap scan shows this is a Windows Domain Controller — ports like 88 (Kerberos), 389 (LDAP), 445 (SMB), and others typical of a DC. Domain is `active.htb`.

---

## SMB Enumeration

Ran `smbmap` without credentials to see what shares are accessible anonymously.

![smbmap — Replication share has READ ONLY access](../HTB_imgs/Pasted%20image%2020260703025100.png)

Only `Replication` is readable without auth. Connected with `smbclient` and started digging through the directory structure. It mirrors the SYSVOL layout — Group Policy folders with GUIDs.

![Browsing Replication share — navigating to Groups.xml](../HTB_imgs/Pasted%20image%2020260703025631.png)

After browsing through the policy directories, found `Groups.xml` under `\active.htb\Policies\{...}\MACHINE\Preferences\Groups\`. This file is a **Group Policy Preferences (GPP)** artifact — it contains the username `active.htb\SVC_TGS` and an AES-encrypted password (`cpassword`).

![Groups.xml — SVC_TGS username and cpassword hash](../HTB_imgs/Pasted%20image%2020260703025803.png)

The thing about GPP passwords is that Microsoft published the AES key on MSDN, so they're trivially decryptable. Kali has a built-in tool for this:

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

![gpp-decrypt — password is GPPstillStandingStrong2k18](../HTB_imgs/Pasted%20image%2020260703030018.png)

Password: `GPPstillStandingStrong2k18`

---

## Authenticated SMB Access

Re-ran `smbmap` with the `SVC_TGS` credentials and now we have READ ONLY access to `NETLOGON`, `SYSVOL`, and `Users` as well.

![smbmap with SVC_TGS creds — more shares accessible](../HTB_imgs/Pasted%20image%2020260703032522.png)

The `Users` share maps to `C:\Users\` and we could grab the user flag from here. But instead of digging deeper into SMB, since we now have a valid domain account, let's check for **Kerberoasting**  requesting TGS tickets for service accounts and cracking them offline.

---

## Kerberoasting

Used Impacket's `GetUserSPNs.py` to request TGS tickets for any SPNs associated with user accounts in the domain:

```bash
impacket-GetUserSPNs -request -dc-ip 10.129.108.47 active.htb/SVC_TGS -save -outputfile GetUserSPNs.out
```

The Administrator account has an SPN registered (`active/CIFS:445`), so we got a TGS ticket hash for it.

![GetUserSPNs — Administrator TGS hash retrieved](../HTB_imgs/Pasted%20image%2020260703033514.png)

Cracked it with `hashcat` using mode 13100 (Kerberos 5 TGS-REP) against `rockyou.txt`:

```bash
hashcat -m 13100 GetUserSPNs.out /usr/share/wordlists/rockyou.txt
```

![hashcat — cracked: Ticketmaster1968](../HTB_imgs/Pasted%20image%2020260703033816.png)

Password: `Ticketmaster1968`

---

## Getting a Shell

Since we have the Administrator password, we can use Impacket's `psexec.py` to get a SYSTEM shell. It works by uploading a binary to a writable share (`ADMIN$`), creating a service, and executing it.

```bash
impacket-psexec active.htb/administrator@10.129.108.47
```

![psexec — SYSTEM shell](../HTB_imgs/Pasted%20image%2020260703034121.png)

We're in as `nt authority\system`.

---

## User Flag

![User flag](../HTB_imgs/Pasted%20image%2020260703034335.png)

## Root Flag

![Root flag](../HTB_imgs/Pasted%20image%2020260703034428.png)
