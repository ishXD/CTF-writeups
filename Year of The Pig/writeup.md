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
 For wrong credentials the following message appears:
 `Remember that passwords should be a memorable word, followed by two numbers and a special character`
 Take a note of that.
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
You can use [CeWL](https://digi.ninja/projects/cewl.php) for this
 

