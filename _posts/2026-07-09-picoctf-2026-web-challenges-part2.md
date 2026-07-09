---
layout: post
title: "picoCTF 2026: Web Challenges Walkthrough (Part 2)"
date: 2026-07-09
categories: [Walkthrough, picoCTF]
tags: [web, sql-injection, postgresql, sqlmap, hash-cracking]
---

Welcome to Part 2 of my picoCTF 2026 Web Exploitation series. In this post, we shift our focus toward exploiting poor database interactions and breaking weak validation mechanisms.

## Completed Challenges
* [Challenge 4: Secret Box](#challenge-4-secret-box)
* [Challenge 5: Sql Map1](#challenge-5-sql-map1)

---

## Challenge 4: Secret Box

Once I launched the challenge, I noticed two links: one for the web application and another for the source code. My usual approach is to interact with the application first to understand how it works, then use the source code to validate my assumptions or look for hidden flaws. 

The challenge also provided a hint: **“How to use SQL Injection.”** This strongly suggested the intended attack vector, so I kept that in mind while exploring the application.

### Initial Interaction
The application initially presented a **Login** and **Signup** page. Since I didn’t have any valid credentials, I created a new account through the signup functionality and logged in. After authentication, I was redirected to a panel where I could create a new secret message.

![Secret Box Panel](/assets/img/secret-box-dashboard.png)
*Figure 1: Secret creation dashboard*

### Fuzzing & SQL Error Leak
Since the challenge hint mentioned SQL Injection, that was the first vector I wanted to test. To see how the application handled raw user input, I entered a single quote (`'`) into the input field and submitted the request.

![Triggering SQL Error](/assets/img/secret-box-error.png)
*Figure 2: Submitting a single quote to force an unhandled exception*

Submitting the single quote (`'`) successfully triggered a verbose database error, confirming a potential SQL Injection vulnerability! 

The error output leaked essential backend structure: the application was utilizing a PostgreSQL database with a table called `secrets` containing fields named `owner_id` and `content`. 

### Source Code Audit & Payload Crafting
While I now knew the schema, I still didn’t know the administrator's specific `owner_id`. I opened up the provided backend source code to look for clues.

![Reviewing Source Code](/assets/img/secret-box-source.png)
*Figure 3: Finding administrative parameters within the source files*

While reviewing the source code, I successfully located the static admin `owner_id` and verified that target flag secrets were kept within the `content` column. 

Because the application used PostgreSQL, I could leverage its native string concatenation operator (`||`) to string-inject the result of an arbitrary subquery back into the visible application output interface.

Using the discovered administrative ID (`e2a66f7d-2ce6-4861-b4aa-be8e069601cb`), I crafted the following data-exfiltration SQL Injection payload:

```sql
' || (SELECT content FROM secrets WHERE owner_id='e2a66f7d-2ce6-4861-b4aa-be8e069601cb') || '
```

### Flag Extraction
I dropped this payload right into the secret submission field. The application evaluated the malicious input string, processed the inline query, and cleanly output the admin's secret right on the page. 🎉

![Flag Extracted](/assets/img/secret-box-flag.png)
*Figure 4: The application printing the admin's secret*

<details>
  <summary>🔑 Click to reveal Secret Box flag</summary>
  <code>picoCTF{m4sk3d_s3cr3t_b0x_fl4g}</code>
</details>

---

## Challenge 5: Sql Map1

The challenge description hinted at weak coding practices and outdated password storage mechanisms. The provided hints pointed directly toward the app's internal search functionality, explicitly highlighted MD5 hashes, and even referenced CrackStation. This gave me a clear roadmap combining SQL Injection automation and credential cracking.

### Initial Discovery
After launching the instance, I was presented with a baseline Login and Register page. I quickly registered a test account and logged in to find a search field that returned several fake flags. 

Given that the hints specifically referenced the search parameter and even provided a boilerplate automation template, I suspected this field was fully vulnerable to SQL Injection.

![Inspecting Search Parameter](/assets/img/sqlmap1-search.png)
*Figure 5: Target search field*

### Automated Database Dumping
Rather than mapping the injection points manually, I opted to run `sqlmap` against the vulnerable parameter to systematically extract underlying table schemas. 

The challenge hinted that we needed to act as a *legitimate user* to retrieve the secret flag. I enumerated the database tables and mapped out three distinct relations: `users`, `flags`, and `sqlite_sequence`.

```bash
sqlmap -u "http://<TARGET_URL>/vuln.php?q=test" --cookies="PHPSESSID=234er435er6586750e" -p q --batch --tables
```

![Running sqlmap Enumeration](/assets/img/sqlmap1-tables.png)
*Figure 6: Automated extraction of the database schema*

Because legitimate user credentials are consistently maintained inside user profiles, I shifted my focus to dumping data columns straight out of the `users` table:

```bash
sqlmap -u "http://<TARGET_URL>/search?q=test" --cookies="PHPSESSID=234er435er6586750e" -p q --batch -T users --dump
```

![Dumping Table Data](/assets/img/sqlmap1-dump.png)
*Figure 7: Exfiltrating table contents*

The exfiltrated database data revealed several distinct platform usernames along with corresponding legacy password hashes. 

One specific MD5 hash had already been auto-resolved by a public lookup table, revealing the plaintext password string: `dyesebel`. 

### Privilege Escalation & Flag Recovery
I initially attempted to log in as the default `admin` account using this string, but authentication failed. I then pivoted to trying the recovered user profile credentials pairing:

* **Username:** `ctf-player`
* **Password:** `dyesebel`

The login request passed successfully! Authenticating into the web terminal as a verified internal employee completely bypassed restriction filters, dropped me onto the main user interface dashboard, and displayed the flag. 🎉

![Flag Retrieved](/assets/img/sqlmap1-flag.png)
*Figure 8: Accessing the system to display the final flag*

<details>
  <summary>🔑 Click to reveal Sql Map1 flag</summary>
  <code>picoCTF{m4sk3d_sql_m4p1_fl4g}</code>
</details>

---

## Happy Hacking! 🚀
