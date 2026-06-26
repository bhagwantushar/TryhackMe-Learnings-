# TryHackMe — Brooklyn Nine-Nine Writeup

**Difficulty:** Easy
**Tags:** Linux, FTP, Brute Force, Privilege Escalation

> *"I almost spent 45 minutes brute-forcing SSH when the answer was sitting on an FTP server the whole time."*

---

## Before I Start — A Honest Confession

When I first saw this room, I did what most beginners do — I saw port 22 open and immediately thought "let's brute-force SSH." I even fired up Hydra with the full rockyou.txt wordlist and watched it grind away at 5000 tries per minute, thinking I was making progress.

I wasn't.

The real entry point took me about 30 seconds once I actually slowed down and looked at my Nmap results properly. So if you're reading this before attempting the room yourself — don't be me. Read everything first.

---

## Step 1 — Enumeration

Every box starts the same way for me. I run Nmap and actually read what it tells me instead of just looking for SSH.

```bash
nmap -sC -sV 10.48.150.130
```

Here's what came back:

| Port | Service | Version |
|------|---------|---------|
| 21   | FTP     | vsftpd 3.0.3 |
| 22   | SSH     | OpenSSH 7.6p1 |
| 80   | HTTP    | Apache 2.4.29 |

The thing that jumped out was this line in the FTP result:

```
ftp-anon: Anonymous FTP login allowed
```

Anonymous FTP means you can log in with the username `anonymous` and no password. It's one of those misconfigurations that sounds minor but can completely unravel a system — because people leave files there without thinking about who can read them.

I also checked the webpage on port 80. It's a Brooklyn Nine-Nine fan page with a background image. Nothing in the source either. Honestly it was just vibes and no substance — moving on.

---

## Step 2 — FTP Access

I connected to FTP and logged in as anonymous:

```bash
ftp 10.48.150.130
```

Username: `anonymous`
Password: just hit Enter

```bash
ls
```

There was one file sitting there: `note_to_jake.txt`. I downloaded it:

```bash
get note_to_jake.txt
exit
```

The note was from Amy to Jake. It basically said his password was embarrassingly weak and that Holt would be furious if someone got in because of it.

That one note told me three things:
- The username is **jake**
- His password is weak enough to be in a wordlist
- There's also a user called **holt** on this machine

Sometimes the best recon isn't a fancy tool. It's just reading a text file someone left lying around.

---

## Step 3 — SSH Brute Force

With `jake` as my target, I used Hydra to brute-force his SSH password. One important thing I learned the hard way — SSH will throttle you if you throw too many threads at it. Keep it at 4:

```bash
hydra -l jake -t 4 -P /usr/share/wordlists/rockyou.txt 10.48.150.130 ssh
```

It found the password pretty quickly since it was near the top of rockyou.txt — exactly what you'd expect from a "weak" password.

I logged in:

```bash
ssh jake@10.48.150.130
```

Once inside, I poked around the home directories. Found the user flag sitting in `/home/holt/user.txt`. Read it, noted it down, and moved on to the part I was actually curious about — getting root.

---

## Step 4 — Privilege Escalation

The first thing I do after landing on any machine is check what I can run as root:

```bash
sudo -l
```

The output stopped me in my tracks:

```
User jake may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /usr/bin/less
```

Jake can run `less` as root — with no password. Now, `less` is just a file reader. It lets you scroll through files in the terminal. Harmless, right?

Not quite.

`less` has a little-known feature where you can run shell commands from inside it by typing `!` followed by the command. So if you open `less` as root and then use that `!` trick to spawn a shell — that shell inherits root's privileges.

I looked this up on [GTFOBins](https://gtfobins.github.io/gtfobins/less/#sudo) just to confirm, and sure enough, there it was under the Sudo section.

```bash
sudo less /etc/hosts
```

The file opened. From inside `less`, I typed:

```
!/bin/sh
```

Hit Enter. A new shell appeared. I typed:

```bash
whoami
```

```
root
```

Just like that.

```bash
cd /root
cat root.txt
```

Room complete.

---

## What Actually Stuck With Me

I made a mistake at the start of this room that I want to be upfront about. I ran Hydra against SSH immediately without properly reading my Nmap output, and I sat there watching it crawl through 14 million passwords thinking I was doing the right thing.

The FTP server with anonymous login, a note file with the target username and a hint about a weak password — all of that was right there from the beginning. I just wasn't paying attention.

The lesson I took from this room isn't really about Hydra or GTFOBins or `sudo -l`. It's about slowing down. When you're learning, it's tempting to just run tools and hope something pops. But the actual skill is reading what those tools tell you and connecting the dots before you start throwing wordlists at things.

Enumerate. Read. Think. Then attack.

---

## Tools Used

| Tool | What I used it for |
|------|--------------------|
| Nmap | Finding open ports and services |
| FTP  | Accessing the anonymous FTP server |
| Hydra | Brute-forcing Jake's SSH password |
| GTFOBins | Finding the `less` privilege escalation technique |

---

*Written by Tushar — still learning, documenting as I go.*
