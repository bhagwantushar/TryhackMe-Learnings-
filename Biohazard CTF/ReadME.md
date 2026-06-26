# TryHackMe — Biohazard CTF Writeup

**Difficulty:** Medium  
**Tags:** Web Enumeration, Steganography, Encoding/Decoding, FTP, Privilege Escalation

> *"This room doesn't hand you a vulnerability on a silver platter. It hides clues inside clues, and punishes you for rushing."*

---

## Before I Start — What Kind of Room Is This?

Most CTFs I've done follow a pretty predictable path — scan, exploit, escalate. Biohazard is different. It's built like a puzzle game where you're literally walking through a haunted mansion, collecting items and unlocking doors one by one. The story is pulled straight from Resident Evil, and honestly the theming made it more fun to solve.

That said, it also made it easy to get lost. There are multiple rooms, encoded strings everywhere, and the clues you need are often buried in HTML comments or hidden inside image files. If you try to brute-force your way through this one, you'll hit a wall fast.

The mindset that worked for me: **treat every page like it might be hiding something**. Check the source code. Click every link. Read every file.

---

## Step 1 — Enumeration

As always, I started with an Nmap scan:

```bash
nmap -sC -sV 10.49.176.213
```

Three ports came back open:

| Port | Service | Detail |
|------|---------|--------|
| 21   | FTP     | vsftpd |
| 22   | SSH     | OpenSSH |
| 80   | HTTP    | Apache |

The web server was the obvious starting point. Visiting the IP brought up a creepy mansion intro page — July 1998, the STARS alpha team, zombie dogs. Classic RE1 opening. There's a link to the `/mansionmain/` page, and from there the rabbit hole begins.

---

## Step 2 — Exploring the Mansion (Web Enumeration)

This is where the room really opens up. Each page of the mansion is its own little puzzle. The key habit I developed early: **always view the page source before clicking anything**. Most of the clues in this room are hidden in HTML comments, not visible on the page itself.

**Main Hall** — The source had a comment pointing to the `/diningRoom/`. Easy enough.

**Dining Room** — A hidden comment in the source code had a Base64 string. Decoding it gave a hint: visit the `/teaRoom/`. There was also an emblem slot on the wall — I needed to find an emblem before I could use it.

```bash
echo "SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=" | base64 -d
# How about the /teaRoom/
```

**Tea Room** — This one said to visit the `/artRoom/`. Following the trail.

**Art Room** — Found a paper on the wall. It contained a map of the entire mansion, listing all the room paths:

```
/diningRoom/
/teaRoom/
/artRoom/
/barRoom/
/diningRoom2F/
/tigerStatusRoom/
/galleryRoom/
/studyRoom/
/armorRoom/
/attic/
```

This was the moment the room clicked for me. Instead of guessing URLs, I now had a complete roadmap. I worked through each one methodically.

---

## Step 3 — Collecting Items (The Puzzle Layer)

Each room had something to pick up or a lock to open. Here's how it played out:

**Bar Room** — The door was locked and needed a `lock_pick`. I'd found that earlier from the `/teaRoom/master_of_unlock.html` page. Entered it, got redirected inside. Inside the bar room there was a music note — a Base32 encoded string. Decoded it:

```bash
echo "NV2XG2LDL5ZWQZLFOR5TGNRSMQ3TEZDFMFTDMNLGGVRGIYZWGNSGCZLDMU3GCMLGGY3TMZL5" | base32 -d
# music_sheet{362d72deaf65f5bdc63daece6a1f676e}
```

There was also a gold emblem on the wall. Picked it up — `gold_emblem{58a8c41a9d08b8a4e38d02a4d7ff4843}`.

I tried entering the gold emblem in the dining room slot first — nothing happened. Turns out I needed to paste it into the **bar room's** emblem slot instead. After that, the dining room emblem slot accepted it and gave me a Vigenere-encoded string — which I'd need to decode later.

**Tiger Status Room (via armorRoom)** — The armor room had a note with crest 3 encoded in Base64 (triple-encoded, the hints warned). I decoded it in CyberChef. The tiger room needed the `blue_jewel` — which I'd found by decoding a ROT13 string from the attic that pointed to `/diningRoom/sapphire.html`.

**Attic** — After unlocking it with the `shield_key` found at `/diningRoom/the_great_shield_key.html`, Jill fought a snake and found a note. The note was ROT13 encoded. Decoded it in CyberChef — it pointed to the blue gem location.

---

## Step 4 — Decoding All Four Crests

Throughout the mansion I collected four crest fragments, each encoded differently:

- **Crest 1** — Base64 (encoded twice) — found in the Tiger Status Room
- **Crest 2** — Base64 (encoded twice) — found in the Gallery Room  
- **Crest 3** — Base64 (encoded three times) — found in the Armor Room
- **Crest 4** — Base64 (encoded twice) — found in the Attic note

The room told me to combine all four and decode the result. I concatenated them and ran the whole thing through CyberChef's From Base64. What came out was:

```
FTP user: hunter
Password: you_cant_hide_forever
```

Just like that, after all that web crawling, I had FTP credentials.

---

## Step 5 — FTP Access

Logged in as hunter:

```bash
ftp 10.49.176.213
```

The FTP server had several image files and an `important.txt`. The note from Barry told me the helmet key was inside one of the files but he couldn't decrypt it — and there was a `/hidden_closet/` door that was locked.

I downloaded everything:

```bash
get important.txt
get 001-key.jpg
get 002-key.jpg
get 003-key.jpg
get helmet_key.txt.gpg
```

---

## Step 6 — Steganography on the Key Images

This is where tools like `steghide` and `exiftool` became essential.

**001-key.jpg** — Used `steghide` to extract a hidden file. It asked for a passphrase (which turned out to be empty — just hit Enter). Got `key-001.txt` with the first part of a key.

```bash
steghide --info 001-key.jpg
steghide extract -sf 001-key.jpg
cat key-001.txt
```

**002-key.jpg** — Ran `exiftool` and found a string hidden in the Comment field of the image metadata. That was the second key fragment.

```bash
exiftool 002-key.jpg
# Comment: 5fYmVfZGVzdHJveV9
```

**003-key.jpg** — Used `binwalk` to extract a zip archive hidden inside the image. Inside was `key-003.txt` with the third fragment.

```bash
binwalk -e 003-key.jpg
cat _003-key.jpg.extracted/key-003.txt
```

Combining all three key fragments and decoding through CyberChef (From Base64) gave me the GPG passphrase: `plant42_can_be_destroy_with_vjolt`

---

## Step 7 — Decrypting the GPG File

With the passphrase in hand:

```bash
gpg helmet_key.txt.gpg
# Enter passphrase: plant42_can_be_destroy_with_vjolt
cat helmet_key.txt
# helmet_key{45f6093193501d2b94bbab2e727f0db4b}
```

I used this key to unlock the `/hidden_closet/` door on the website, which led to the closet room — where Jill found Enrico, the STARS Bravo leader, who was shot dead before he could name the traitor. She found a MO Disk 1 and a wolf medal.

---

## Step 8 — The Final Decode Chain

Inside the closet, the MO Disk 1 had a Vigenere-encoded string:

```
wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk
```

I used an online Vigenere decoder (auto-solve mode) and it cracked it with the key `albert`:

```
weasker login password, stars_members_are_my_guinea_pig
```

The study room (unlocked with the helmet key) gave me a `doom.tar.gz` file, which extracted to `eagle_medal.txt`:

```
SSH user: umbrella_guest
```

---

## Step 9 — SSH and Final Escalation

Logged in via SSH as `umbrella_guest` using the password from the wolf medal file (`T_virus_rules`).

Inside the home directory I found a `.jailcell` folder with `chris.txt` — a conversation between Jill and Chris revealing the traitor and giving MO Disk 2 with the key: `albert`.

Using `albert` as the Vigenere key on MO Disk 1 confirmed the Weasker credentials:

```
weasker / stars_members_are_my_guinea_pig
```

Switched to Weasker and ran `sudo su` — and got root:

```bash
su weasker
sudo su
whoami
# root
cat /root/root.txt
```

The final scene played out in the terminal — Weasker monologuing, Jill stunning him, Brad blowing up the mansion with a rocket launcher. Room complete.

---

## What I Actually Learned

This room was genuinely different from anything I'd done before. The skills it tested weren't really about exploitation — they were about patience and attention to detail.

A few things that stuck with me:

**Source code is not optional.** Almost every room had a clue hidden in an HTML comment. If you just look at the page visually, you'll miss half the puzzle.

**Steganography comes in many forms.** Files can hide data in metadata (exiftool), embedded files (steghide, binwalk), or encrypted archives inside images. Knowing which tool to reach for takes practice.

**Encoding ≠ encryption.** Base64, Base32, ROT13, Vigenere — none of these are secure. They're just obfuscated. CyberChef is your best friend for chaining these operations together.

**CTF rooms can tell stories.** The Resident Evil theming wasn't just decoration — it gave every step context and made the puzzle feel like it had a reason. I enjoyed this more than I expected to.

---

## Tools Used

| Tool | What I used it for |
|------|--------------------|
| Nmap | Initial port and service enumeration |
| Browser DevTools | Reading HTML source code for hidden comments |
| CyberChef | Base64, Base32, ROT13, Vigenere decoding |
| steghide | Extracting hidden files from images |
| exiftool | Reading image metadata |
| binwalk | Extracting embedded archives from images |
| gpg | Decrypting the GPG-encrypted helmet key |
| FTP client | Accessing the hunter FTP account |
| SSH | Getting initial foothold on the machine |

---

*Written by Tushar — still learning, documenting as I go.*
