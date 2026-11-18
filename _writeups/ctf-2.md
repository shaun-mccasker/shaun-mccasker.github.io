---
layout: writeup
title: TryHackMe - Support
date: 2026-10-28
platform: TryHackMe - OCT 2026
category: Web Exploitation
difficulty: Medium
description: Information/Directory exposure, Insecure Credential Storage, BOLA, LFI, CMD Injection
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

- The assessment began with reconnaissance using Nmap and Gobuster, identifying SSH and an Apache web server, along with potentially interesting directories and files such as `/includes, /skins, /info.php and /config.php`.

- The initial login page exposed a valid `email: help@support.thm`. Testing revealed no apparent brute-force protection, allowing Hydra to identify the password `snoopy`. After logging in, the session `cookie isITUser` was found to contain an `MD5 hash of the value false`. Replacing it with the MD5 hash of true allowed access to additional functionality.

- Further investigation uncovered a Broken Object Level Authorization (BOLA) vulnerability in the user API. By changing the user ID in the API URL, it was possible to access another user's information, including the administrator's `email: specialadmin@support.thm`.

- A Local File Inclusion (LFI) vulnerability was then discovered through the dashboard's theme selection parameter. This allowed access to internal PHP files, including `config.php`, which exposed the administrator's `password:support@110`. After correcting the password format `password: support110`, administrator access was successfully obtained ADMIN FLAG `THM{I_AM_ADMIN999}`

- Finally, the administrator dashboard provided a `system date` functionality that was vulnerable to command injection. By manipulating the sys parameter, it was possible to execute additional system commands and retrieve the contents of `/home/ubuntu/user.txt` FLAG: `THM{GOT_THE_FLAG001}`.

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
- info.php   - Revealed webserver info and existence of config.txt
- skins          -  Useful later on 
- config.txt    -  Useful later on 

It's worth noting `config.txt` could be loaded but nothing was rendered to the page or in the page source.

## Credential brute force attack
When loading the homepage we are presented with a login form. Within the form and below the form it is indicated that there is an account with
email/username `help@support.thm`.

Trying some login attempts we can see the error renders is `Invalid Credentials`. Notably it did not disclose whether the username was valid/invalid.

Using burp suite I automated some attacks and received `200 OK` error `Invalid Credentials`. It's worth noting that repeating this attack did not return a 429 indicating no rate limiting. Knowing this a Hydra payload was created.

```
hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt \
10.49.154.66 http-post-form '/:email=^USER^&password=^PASS^:Invalid credentials' -f -V
```
Hydra succeeded returning credentials `email: help@support.thm password: snoopy`

Logging in using these credentials we get to the dashboard.

## Cookie manipulation

Within the dashboard all we can see a change themes button, this allows us to change the colour from red, green, blue and default. Inspecting local storage revealed our session is stored in as cookies with values, isITUser and PHPSESSID.
`IsITUser: 68934a3e9455fa72420237eb05902327:false`. Using crackstation it’s found to be an md5 hash of the word false.

Changing the cookie value to `IsITUser: b326b5062b2f0e69046810717534cb09:true` revealed an 'IT Admin Panel' and granted access to `/api.php`

## BOLA 
`/api.php` reveals an Api Request/Response

```
API REQUEST
Get /user/3

API RESPONSE
{
    "email": "help@support.thm",
    "2FA": false,
    "admin": false
}
```

This was indicative of a potential Bola. In the URL I tried `10.49.153.228/user/3` which revealed my current user information.
Next `10.49.153.228/user/1` revealed user 1 information
```
email	"specialadmin@support.thm"
2FA	false
admin	true
```

We now have an admin email/username `specialadmin@support.thm`

## File Inclusion 
Going back to the themes. Loading a theme loads changes the url to   
`10.49.159.168/dashboard.php?skin=red`. When inspecting the `skins` directory from earlier it revealed a directory listing with files `blue.php, default.php, green.php, red.php`.  
With this information it can be seen changing themes loads the relevant skin file stripping the `.php` from the url.
Testing for file inclusion I tried `10.49.159.168/dashboard.php?skin=../dashboard`. This loaded the dashboard twice to the UI indicating success. When inspecting the page source we could see internal code which is not usually visible.

With this knowledge I went back to the enumerated directories and began loading them into the interface via file inclusion. When loading `config.txt` as `10.49.159.168/dashboard.php?skin=../config` we can see the source code for this file notably containing `$MASTER_PASSWORD = 'support@110';`.

We now had credentials `username: specialadmin@support.thm password: support@110`
NOTE: `support110` is the actual password, no idea why.

Attempting to login with these credentials revealed ADMIN FLAG `THM{I_AM_ADMIN999}`

## Command Injection

The only different thing about the admin account is a dropdown to select date or time to display. Intercepting this request in burp suite reveals a parameter called `‘sys’` with value `date`.

This looked like a potential command injection so I inputted who am I which returned the error `‘Only date is allowed’`. The logic was filtering the sys input. A simple check by inputting ‘date; whoami’ revealed it was only checking to see if the date or time value existed returning 

```
Tue OCT 28 22:56:18 UTC 2026
www-data
```

This indicated a successful command injection. The task said the flag is located at `/home/ubuntu/user.txt` therefore I injected `date; cat /home/ubuntu/user.txt` which revealed the final FLAG: `THM{GOT_THE_FLAG001}`
