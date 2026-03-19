## Flags
- **User Flag:** `86567d62e8554eabef93d9f6020505d6`
- **Root Flag:** `04a0ab5bcf5f64b6fab38d2b03ecec19`

---

## Target Information
- **Operating System:** Linux
- **Found Credentials:**
  - `dwight : Queenofblad3s!23`
  - `rocket : my$ecretPass`

---

## Initial Enumeration

### 1. Open Services
The following ports were open:
- `22` — SSH
- `80` — HTTP
- `443` — HTTPS

The web server was Apache running on a CentOS system.

---

### 2. Web Server (Port 80)
- **Language:** PHP 7.2.24
- A custom header revealed an internal domain:
  - `X-Backend-Server: office.paper`
- Directory scanning found:
  ```
  /manual
  /cgi-bin/
  /.npm
  ```

---

## Web Application Enumeration

### 3. Domain: `office.paper`
- The site runs **WordPress 5.2.3**
- Found WordPress users:
  - `nick`
  - `prisonmike`

  ```bash
  curl http://office.paper/index.php/wp-json/wp/v2/users
  ```

- Found a subdomain: `chat.office.paper`
- A Post give us a hint about draft posts
- WordPress was vulnerable to **CVE-2019-17671**
- This bug allows reading draft posts without login
- One draft post leaked a secret chat registration link:
  ```
  http://chat.office.paper/register/8qozr226AhkCHZdyY
  ```

---

### 4. Subdomain: `chat.office.paper`
- Meteor‑based chat application
- Written in JavaScript
- Registration was locked but worked using the leaked link
- After registering, internal employee chats were visible

---

## Chat Bot Abuse

### 5. Recyclops Bot
- Employees talked about a bot called **recyclops**
- Bot commands were discovered using:
  ```
  recyclops help
  ```

#### Bot Features
- List files
- Read files
- Show system time
- Access limited to the **Sales** folder

- The bot leaked the system username:
  - `dwight`
- The bot allowed reading files on disk
- Found credentials in:
  ```
  /home/dwight/habot/.env
  ```

  Password:
  ```
  Queenofblad3s!23
  ```

- Used these credentials to log in via SSH as `dwight`

---

## Privilege Escalation

### 6. User Access
- **User Flag:** `86567d62e8554eabef93d9f6020505d6`
- Further checks revealed MongoDB credentials:
  ```
  mongodb://rocket:my$ecretPass@localhost:27017/local
  ```

- Database access did not lead to root access

---

### 7. Root Exploit
- System was vulnerable to **CVE-2021-3560 (Polkit / pwnkit)**
- Vulnerability confirmed using `linpeas`
- Used a public exploit:
  - https://github.com/secnigma/CVE-2021-3560-Polkit-Privilege-Esclation
- Successfully escalated privileges to root

---

✔ Full system compromise complete.

---

## Summary
This machine was compromised by chaining:
- An old WordPress vulnerability
- Information leaks
- Weak access controls in a chat bot
- Credential reuse
- A known Linux privilege escalation bug

Keeping everything updated would have stopped us, this is the importance of updating what you use  
