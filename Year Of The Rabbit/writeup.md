## ENUMERATION

An nmap scan of the host reveals 3 open ports: 21,22 and 80. 80 is running Apache and 22 as usual, is running SSH.
`nmap -sC -sV -o nmap MACHINE-IP`

```
# Nmap 7.93 scan initiated Sun Jul 9 14:59:25 2023 as: nmap -sC -sV -o nmap 10.10.116.151
        Nmap scan report for 10.10.116.151
        Host is up (0.16s latency).
        Not shown: 997 closed tcp ports (conn-refused)
        PORT STATE SERVICE VERSION
        21/tcp open ftp vsftpd 3.0.2
        22/tcp open ssh OpenSSH 6.7p1 Debian 5 (protocol 2.0)
        | ssh-hostkey: 
        | 1024 a08b6b7809390332ea524c203e82ad60 (DSA)
        | 2048 df25d0471f37d918818738763092651f (RSA)
        | 256 be9f4f014a44c8adf503cb00ac8f4944 (ECDSA)
        |_ 256 dbb1c1b9cd8c9d604ff198e299fe0803 (ED25519)
        80/tcp open http Apache httpd 2.4.10 ((Debian))
        |_http-title: Apache2 Debian Default Page: It works
        |_http-server-header: Apache/2.4.10 (Debian)
        Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
        # Nmap done at Sun Jul 9 14:59:59 2023 -- 1 IP address (1 host up) scanned in 34.13 seconds
```

## ACTIVE ENUMERATION

Let's investigate the webserver now. It's the default Apache landing page. Nothing out of the ordinary could be found even from the page source.
Let's use gobuster to find some hidden directories:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/gobuster.png)

Let's check out the assets directory. There we find a style.css file which gives us `/sup3r_s3cret_fl4g.php`. Navigating to it , we get:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20164949.png)

To turn javascript off in firefox: open a new tab , type `about:config` and hit enter. Search for javascript and you'll find the option. Continuing ..... and we get rickrolled. Was this why we had to turn the audio on? Finding nothing else , time to move on to good ol' burp.
After setting up burp and going back to /sup3r_s3cr3t_fl4g.php again , we find :

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20165300.png)

We get a hidden directory. Navigating to it, we get:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20165440.png)

Downloading the file - an image , `Hot_Babe.png`
Normally , with images we try to find some hidden messages in there using binwalk or Exiftool , but this time the answer is much simpler. Using strings on the file:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20165627.png)

Yay! We got a wordlist! Saving all those passwords in a file and using hydra to bruteforce the ftp login we get:
Command : `hydra -l ftpuser -P pass.txt ftp://MACHINE-IP`

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20165802.png)

Some sweet credentials! Login to ftp and download the file Eli's_Creds using the `get` command
Now I didn't know what the hell this was but it's an esoteric language called Brainfuck. Using an online decoder we get Eli's credentials.

## FOOTHOLD
SSH using Eli's creds:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20170042.png)

In search of a `s3cr3t` file we got Gwendoline's credentials! The user flag is not in Eli's home directory so let's look in gwendoline's now.
Command : `su gwendoline`
~[](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20170738.png)
Sweet! Got the user flag

## PRIVILEGE ESCALATION
Checking which commands can user gwendoline run: `sudo -l`
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20170856.png)

This means that we are allowed to run vi to open the user flag as any user except the root. If it had been ALL instead of !root , this would have been easier (just open vi and type !/bin/sh). However there is a vulnerability (CVE-2019-14287) we can exploit. If we give a parameter user id of -1 , the command will run as root. Command: `sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt`
This will open the vi editor. Type `:` and gain the command line. Now type `!/bin/sh`, hit enter and there you have the root shell.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20Of%20The%20Rabbit/images/Screenshot%202023-07-09%20171410.png)





