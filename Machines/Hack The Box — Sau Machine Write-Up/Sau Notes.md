
## Data
- Users:
	- puma:
	- maltrail -> admin:changeme!
- Machine : Linux, Ubuntu

## Services
1. 22 ssh 
	Notes:
		- version : OpenSSH 8.2p1 Ubuntu 4ubuntu0.7

2. 55555 http
	Notes:
		- Port is running the request-baskets app version 1.2.1 
		- The version 1.2.1 is vulnerable to SSRF in /api/baskets/{name} the vulnerability has an CVE of CVE-2023-27163
		- [[scripts#Automate the SSRF]] for the SSRF
		- After the SSRF we found that the port 80 is open hosting `Maltrail  v0.53` 
		- Maltrail v0.53 is vulnerable to command injection CVE-2023-27163
		- Using [[scripts#Automated Command Injection]] with the a URL that points to the login page created by [[scripts#Automate the SSRF]] you can get a shell in the target
		- I made a script to automate everything [[scripts#Combining Both]]

### Inside the machine

- We get a shell as the `puma` user
-  In the `/opt/maltrail/maltrail.conf` file we found that the password of admin in the application is `changeme!` the default 
- We have sudo privileges to use systemctl status trails.service
- we get a root shell if we append !sh to less
