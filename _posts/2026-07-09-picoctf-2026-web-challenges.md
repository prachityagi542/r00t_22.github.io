---
layout: post
title: "picoCTF 2026: Web Challenges Walkthrough"
date: 2026-07-09
categories: [Walkthrough, picoCTF]
tags: [web, cookies, flask, hash-enumeration, sqlite]
---

An overview of my solutions for the picoCTF 2026 Web Exploitation category. 

## Completed Challenges
* [Challenge 1: Old Sessions](#challenge-1-old-sessions)
* [Challenge 2: No FA](#challenge-2-no-fa)
* [Challenge 3: Hashgate](#challenge-3-hashgate)

---

## Challenge 1: Old Sessions

As the challenge name **“Old Sessions”** suggests, I assumed from the beginning that the challenge would involve some kind of session manipulation or improper session handling. After reading the description, that suspicion became stronger.

### Reconnaissance
The landing page was fairly simple and contained only two options: **Login** and **Register**.

![Old Sessions Landing Page](/assets/img/old-sessions-landing.png)
*Figure 1: Landing page options*

Since there wasn’t much functionality visible initially, I decided to begin with the registration flow and observe how the application managed authentication and session creation after login. I registered a random account and logged in with the same credentials.

![Registration Flow](/assets/img/old-sessions-register.png)
*Figure 2: Creating a test account*

### Enumeration
After logging in, I noticed a user named `mary_jonas_8992` had left a comment saying: 
> *“Hey, I found a strange page at /sessions.”*

That immediately caught my attention. I navigated directly to `/sessions`.

![Comment Leak](/assets/img/old-sessions-comment.png)
*Figure 3: User comment pointing to the vulnerable endpoint*

On the `/sessions` page, I found two session IDs — one belonging to `admin` and the other matching the user account I had just created.

![Sessions Page](/assets/img/old-sessions-list.png)
*Figure 4: Exposed session tokens*

To verify my assumption, I checked my browser cookies to see whether it matched the session ID shown on the screen. It was an exact match.

![Cookie Comparison](/assets/img/old-sessions-cookies.png)
*Figure 5: Inspecting browser session value*

### Exploitation
Since the application relied entirely on these visible session IDs, I modified my session cookie in the browser, replaced my session value with the **admin’s session ID**, and reloaded the page.

![Session Hijacking](/assets/img/old-sessions-hijack.png)
*Figure 6: Swapping tokens in DevTools*

After refreshing the page, I was successfully logged in as admin! The application accepted the hijacked session and revealed the flag.

![Admin Flag Found](/assets/img/old-sessions-flag.png)
*Figure 7: Admin panel access*

<details>
  <summary>🔑 Click to reveal Old Sessions flag</summary>
  <code>picoCTF{m4sk3d_0ld_s3ss10ns_fl4g}</code>
</details>

---

## Challenge 2: No FA

After launching the instance, three resources were provided:
* The vulnerable application URL
* The application Backend code in `app.py`
* A database file

### Database Analysis & Cracking
Opening the vulnerable application URL led to a standard login page. Lacking valid credentials, I opened the provided database file using **SQLite Browser** to explore the tables.

![SQLite Browser Analysis](/assets/img/nofa-db.png)
*Figure 8: Exploring the database structure*

Inside, I found an admin username along with a hashed password. 

![Admin Hash Discovered](/assets/img/nofa-hash.png)
*Figure 9: Finding the admin credentials hash*

I submitted the hash to [CrackStation](https://crackstation.net/). Fortunately, the hash was already available in its database, allowing me to recover the plaintext password instantly.

![Hash Cracking](/assets/img/nofa-crackstation.png)
*Figure 10: CrackStation plaintext recovery*

### Bypassing 2FA via Session Leak
I logged in using the recovered admin credentials. The application accepted the login but prompted for a 2FA code. 

![2FA Prompt](/assets/img/nofa-2fa.png)
*Figure 11: The secondary 2FA gate*

I first tried brute-forcing the OTP, but it failed. I pivot-checked the provided `app.py` source code to see how the OTP was validated:

```python
# Snippet from app.py
session['otp'] = generated_otp
```

The OTP was stored directly in the **Flask session**. Since Flask sessions are client-side cookies that are signed but *not* encrypted, the session data can be decoded by the user. Using the browser’s Inspect tool, I grabbed the raw Flask session cookie.

![Flask Session Cookie](/assets/img/nofa-cookie.png)
*Figure 12: Extracting the Flask session string*

I pasted the cookie into an online [Flask Session Decoder](https://www.kirsle.net/wizards/flask-session.cgi).

![Decoding Session](/assets/img/nofa-decoded.png)
*Figure 13: Reading client-side session data*

Decoding the cookie revealed the cleartext OTP stored inside it. I entered this value into the 2FA prompt, bypassed the verification step, and grabbed the flag.

![Flag Revealed](/assets/img/nofa-flag.png)
*Figure 14: Bypassed 2FA panel*

<details>
  <summary>🔑 Click to reveal No FA flag</summary>
  <code>picoCTF{m4sk3d_n0_f4_byp4ss_fl4g}</code>
</details>

---

## Challenge 3: Hashgate

From the challenge description and hints, I suspected that user IDs were being processed using a one-way function. 

### Initial Recon & Hash Analysis
I inspected the application page source code and discovered hardcoded login credentials. I logged in and was redirected to a page that displayed a user ID in the URL parameter.

```text
http://target-app/profile?id=21232f297a57a5a743894a0e4a801fc3
```

The user ID was clearly represented as a hash. I used [CrackStation](https://crackstation.net/) to decode the MD5 hash and retrieve the original numerical value.

![Decoding ID Hash](/assets/img/hashgate-crack.png)
*Figure 15: Reversing the ID parameter hash*

### ID Enumeration Script
The breakdown confirmed the hash mapped directly to an integer ID. The hint suggested there were only about 20 employees, so I wrote a short Python automation script (`hash.py`) to generate hashes for IDs in the range of 3000–3020 to locate the administrative profile.

```python
import hashlib

# Enumerating possible employee IDs based on hints
for user_id in range(3000, 3021):
    hashed_val = hashlib.md5(str(user_id).encode()).hexdigest()
    print(f"ID {user_id}: {hashed_val}")
```

I ran the script via the terminal:

```bash
python hash.py
```

I manually tested the generated hashes in the URL parameter. One of the values successfully resolved to the admin profile page, bypassing access control and revealing the flag.

![Admin Account Takeover](/assets/img/hashgate-flag.png)
*Figure 16: Accessing admin profile via ID enumeration*

<details>
  <summary>🔑 Click to reveal Hashgate flag</summary>
  <code>picoCTF{m4sk3d_h4shg4t3_fl4g}</code>
</details>

---

## Conclusion
These challenges showcase classic web application oversights: trusting client-controlled data (session IDs/Flask session cookies) and relying on predictable ID obfuscation instead of proper access controls. Always verify session authenticity on the server side!
