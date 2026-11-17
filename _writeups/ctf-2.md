---
layout: writeup
title: TryHackMe - Support
date: 2026-10-28
platform: TryHackMe - OCT 2026
category: Web Exploitation
difficulty: Medium
description: Information/Directory exposure, Insecure Credential Storage, SSRF, SQLI 
---

# Overview
A new internal Support Operations Platform has been deployed to assist IT and helpdesk teams. The application handles user management, internal APIs, and system-level operations. However, security was not the primary focus during development. Several features rely on user-controlled input and weak trust boundaries.
What is the flag value after logging in as admin?
What is the content of the file /home/ubuntu/user.txt?

## TARGET
- IP 10.49.153.228
- Admin FLAG: THM{__________}  
- /home/ubuntu/user.txt FLAG: THM{__________}  

# Summary

The assessment began with reconnaissance using Nmap and Gobuster, identifying SSH and an Apache web server, along with potentially interesting directories and files such as /includes, /skins, /info.php and /config.php.

The initial login page exposed a valid username (help@support.thm). Testing revealed no apparent brute-force protection, allowing Hydra to identify the password snoopy. After logging in, the session cookie isITUser was found to contain an MD5 hash of the value false. Replacing it with the MD5 hash of true allowed access to additional functionality.

Further investigation uncovered a Broken Object Level Authorization (BOLA) vulnerability in the user API. By changing the user ID in the API URL, it was possible to access another user's information, including the administrator's email address.

A Local File Inclusion (LFI) vulnerability was then discovered through the dashboard's theme selection parameter. This allowed access to internal PHP files, including config.php, which exposed the administrator's password. After correcting the password format, administrator access was successfully obtained.

Finally, the administrator dashboard provided a system date functionality that was vulnerable to command injection. By manipulating the sys parameter, it was possible to execute additional system commands and retrieve the contents of /home/ubuntu/user.txt.

## Enumeration
It was already known this CTF would be focusing on session management, broken authentication, file inclusion, command injection and API testing.

Upon observing the home page I was presented with a login form.

**NMAP**
```
nmap -sC -sV 10.49.153.228
```
Running Nmap resulted in three
Nmap resulted in 2 ports 
- 22/tcp OpenSSH 9.6p1
- 80/tcp Apache 2.4.58.

**GOBUSTER**
```
gobuster dir -u 10.49.153.228 -w /usr/share/wordlists/dirb/directory-list-2.3-medium.txt
```
Gobuster Revealed 5 interesting directories that are accessible 
- includes      - Indicates possible local file inclusion
- info.php   - Revealed webserver info and existance of config.txt
- skins          -  Useful later on 

## Credential brute force attack
When loading the homepage we are presented with a login form. Within the form and below the form it is indicated that there is an account with
email/username `help@support.thm`.

Trying some login attempts we can see the error renders is `Invalid Credentials`. Notably it did not disclose whether the username was valid/invalid.

Using burp suite I automated some attacks and received `200 OK` error `Invalid Credentials`. It's worth noting that repeating this attack did not return a 429 indicating no rate limiting. Knowing this a hyrda payload was created.

```
hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt \
10.49.154.66 http-post-form '/:email=^USER^&password=^PASS^:Invalid credentials' -f -V
```
Hyrda succeeded returning credentials `email: help@support.thm password: snoopy`

Logging in using these credentials we get to the dashboard.

## Cookie manipulation
Within the dashboard we can see we can change themes, this allows us to change the colour from red, green, blue and default. We can also see out session is stored in the local storage as a cookie with isITUser and PHPSESSID.
IsITUser stands out. After a quick reverse hash check it’s found to be an md5 hash of the word false.
## Initial Access

Explain how you identified the vulnerability and gained access.

## Exploitation

Document the technical steps involved in solving the challenge.

## Conclusion

Summarise what you learned and the techniques involved.