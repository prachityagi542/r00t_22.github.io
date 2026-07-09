---
layout: post
title: "TryHackMe: Hidden Deep Into My Heart Write-Up"
date: 2026-07-09
categories: [Walkthrough, TryHackMe]
tags: [web, enumeration, gobuster, robots.txt]
---

I went in “just to take a look”… and 25 minutes later I’m basically emotionally attached to the target system and stealing its secrets 😌🔓

Started with basic enumeration, followed the hidden hints, poked around a bit too confidently, and ended up pulling out the one and only flag.

**Mission status:** system violated, flags extracted, ego slightly inflated ⚡

---

## Approach & Reconnaissance

I opened the target URL and was greeted with nothing fancy — just an IP address and a port number. No hints, no extra pages, nothing.

![Target Web Application Homepage](/assets/img/target-homepage.png)
*Figure 1: Target Web Application Homepage*

First thought that came to mind: *“Alright… time for directory enumeration.”* 😄
So I kicked off directory scanning right away to see what the server was hiding behind the surface.

### Directory Enumeration Using Gobuster
I moved ahead with directory enumeration using `gobuster`, since it’s quick and reliable for basic web recon. 

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

![Directory Enumeration Using Gobuster](/assets/img/gobuster-scan.png)
*Figure 2: Directory Enumeration Using Gobuster*

The scan started revealing a few paths, and one of the interesting hits was `/robots.txt` returning a `200 OK` status. That immediately caught my attention, as `robots.txt` often hides or hints at important directories or files.

---

## Following the Breadcrumbs

I checked `/robots.txt` and found a `Disallow` entry along with a strange string. 

![Discovered robots.txt and Hidden Directive](/assets/img/robots-txt.png)
*Figure 3: Discovered robots.txt and Hidden Directive*

At first, it didn’t make much sense, so I noted it down in my notepad for later — just in case it turned out to be one of those *“you’ll regret ignoring this”* kind of clues 😌

### Exploring the Vault
I visited the `Disallow` directory from `robots.txt` (`/cupids_secret_vault/`), but it didn’t reveal anything useful on the surface. 

![Accessing /cupids_secret_vault/ Directory](/assets/img/secret-vault.png)
*Figure 4: Accessing /cupids_secret_vault/ Directory*

So I went back and ran another round of directory enumeration to dig deeper into this specific path.

```bash
gobuster dir -u http://<TARGET_IP>/cupids_secret_vault/ -w /usr/share/wordlists/dirb/common.txt
```

![Directory Enumeration on /cupids_secret_vault/](/assets/img/gobuster-vault.png)
*Figure 5: Directory Enumeration on /cupids_secret_vault/*

From this second round of directory enumeration, I discovered an `/administrator` directory!

---

## Exploitation & Admin Access

Inside the `/administrator` directory, I landed on a login page. Nothing unusual, just the classic *“please enter credentials”* gatekeeping moment 😄

![Admin Login Page](/assets/img/admin-login.png)
*Figure 6: Admin Login Page*

Since it was clearly an admin panel, I went with the most obvious guess — username as `admin`. 

For the password, I remembered that weird string I had found earlier in `robots.txt`. It didn’t make sense at the time, but it felt too important to ignore, so I gave it a shot here. And well… sometimes CTFs really do reward your *“I’ll just try this random thing”* energy 😏

### Flag Extraction
And just like that… it worked 😏
Logged in successfully, and I finally grabbed the flag. Turns out that “random string from robots.txt” wasn’t so random after all ⚡

![Final flag obtained](/assets/img/final-flag.png)
*Figure 7: Final flag obtained*

<details>
  <summary>🔑 Click to reveal flag spoiler</summary>
  <code>THM{m4sked_h1dd3n_h34rt_fl4g}</code>
</details>

---

## Conclusion

This room was a simple but fun reminder that even small clues like `robots.txt` can lead to full compromises if you follow them properly. A quick recon, a bit of curiosity, and persistence was all it took to extract the flag.

* **Time taken:** 25 minutes ⚡
