#web #lfi #gobuster #ffuf #tryhackme 
room url: https://tryhackme.com/room/support
# Scenario
A new internal **Support Operations Platform** has been deployed to assist IT and helpdesk teams. The application handles user management, internal APIs, and system-level operations. However, security was not the primary focus during development. Several features rely on user-controlled input and weak trust boundaries.

# Summary Attack Chain
1. Enumerate the web application and discover exposed files.
2. Brute-force the helpdesk account credentials.
3. Bypass authorization by modifying an MD5-based role cookie.
4. Exploit a BOLA/IDOR vulnerability to retrieve the administrator's email.
5. Abuse a Local File Inclusion vulnerability to read `config.php` and obtain the administrator password.
6. Exploit command injection in the admin panel to obtain remote code execution and retrieve the final flag.

---
# 1. Reconnaissance and Enumeration

## 1.1 [Nmap](Nmap.md) Scan
ran `nmap -sV -sC -oN scan.nmap TARGET_IP` to discover open ports
![[nmap-scan.png]]
The scan identified two open ports:
- TCP/22 - SSH
- TCP/80 - HTTP

Browsing to the HTTP service presented a login page.
![[Pasted image 20260807122955.png]]
I tested common SQL injection authentication bypass payloads such as:
`' OR 1=1;--` and: `admin@support.thm'--`
but none succeeded, suggesting the login form was not vulnerable to a simple SQL injection bypass.
## 1.2 Gobuster Directory Enumeration
To discover hidden files and directories, I performed directory enumeration using Gobuster. 
```bash
gobuster dir -u http://10.113.166.52/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -t 50 -x php,html,txt,bak,js
```

![[Pasted image 20260807123401.png]]

## 1.3 Mapping Application Structure
```text
/
├── api.php
├── config.php
├── dashboard.php
├── footer.php
├── index.php
├── info.php
├── logout.php
├── includes/
│   ├── header.php
│   └── skin.php
├── js/
│   └── bootstrap.bundle.min.js
├── layout/
│   └── bootstrap.min.css
└── skins/
    ├── blue.php
    ├── default.php
    ├── green.php
    └── red.php
```
Before authentication, none of the discovered files exposed sensitive information. Viewing the page source also did not reveal any useful endpoints or client-side secrets.

---

# 2. Brute Forcing Login
1. Intercept a POST request with invalid credentials using Burp Suite to get the body parameters and note how the app indicates an error
2. The login page explicitly displayed the helpdesk email address: `help@support.thm`. Since the username was known, only the password needed to be brute-forced.
3. Use ffuf to fuzz the password 

![[Pasted image 20260807135448.png]]

![[Pasted image 20260807135822.png]]

---
# 3. Initial Access
After authentication, a new value appeared in browser storage:
`isITUser: 68934a3e9455fa72420237eb05902327`
The value resembled an MD5 hash, so I submitted it to [CrackStation](https://crackstation.net/).
![[Pasted image 20260807140449.png]]

Since the cookie represented the MD5 hash of `false`, I generated the MD5 hash of `true` and replaced the cookie value.
![[Pasted image 20260807140655.png]]

# 4. BOLA/IDOR
After replacing the cookie, I'm presented with an API endpoint `http://10.113.166.52/user/3`
![[Pasted image 20260807141018.png]]

When I change '3' to '1', I get the admin's email
![[Pasted image 20260807142801.png]]
The API did not perform authorization checks to verify whether the requesting user was allowed to access another user's information.
# 5. Local File Inclusion (LFI)
Visiting the home page presents a dropdown menu to select a theme, and it's reflected in the URL
![[Pasted image 20260807143020.png]]
The dashboard allowed users to select a visual theme through the `skin` parameter. For example: `dashboard.php?skin=red`. 

During enumeration, I had already discovered the following file: `skins/red.php` 

Since the URL referenced `red` rather than `red.php`, it suggested that the backend automatically appended the `.php` extension before including the file.

Assuming the application performed something similar to:
```php
include("skins/" . $_GET['skin'] . ".php");
```

Testing path traversal: `dashboard.php?skin=../config`
This caused the application to include: `config.php`, which exposed admin password
![[Pasted image 20260807144216.png]]

---
# Command Injection
Upon using the admin's credentials, I find the admin flag. 
After authenticating as the administrator, additional functionality became available. One feature allowed the administrator to retrieve the system date and time.
![[Pasted image 20260807155802.png]]

Upon viewing the page source, I notice this sends a POST request
![[Pasted image 20260807155947.png]]
Inspecting the request revealed a parameter named `sys`, suggesting the backend executed system-level commands based on user input.

Intercepting a response and modifying it using burp suite:
![[Pasted image 20260807160322.png]]
so we bypass by running `date` and then we end the command using a `;` to terminate the date command and then inject the `cat` command: 
![[Pasted image 20260807160415.png]]

# Lessons Learned

## 1. Always Inspect the Basics Before Making Assumptions

### Mistake:
I focused on backend clues and theories instead of fully checking the information already provided by the application.

Example:
- `phpinfo()` showed the server running as `webmaster@localhost`.
- I assumed the login username would be `webmaster@support.thm`.
- However, the page itself displayed the correct contact email:
```
help@support.thm
```

### Lesson:
Direct application evidence is usually more valuable than indirect assumptions about the backend.

Before trying a theory, ask:
> "What evidence do I actually have that this is correct?"

Always test the obvious possibilities before the clever ones.

---

## 2. Always Check Page Source

### Mistake:
I did not consistently inspect the source code of every page.
### Lesson:
Always check `view-source:` for every page during enumeration.

Look for:
- Hidden comments
- Hardcoded credentials
- JavaScript endpoints
- Hidden parameters
- API routes
- Client-side logic

A simple source inspection can reveal information that is not visible in the UI.

---

## 3. Do Not Ignore API Methods

### Mistake:
I mainly focused on GET requests and normal browser interactions.

### Lesson:
When discovering API endpoints, test different HTTP methods:

- GET
- POST
- PUT
- PATCH
- DELETE

When interacting with APIs, do not rely only on the frontend. The frontend may hide functionality that is still accessible through the backend.

---

## 4. Investigate How the Backend Processes Input

### Mistake:
I noticed the `skin` parameter controlled files, but I did not immediately consider how the backend transformed the value.

The application accepted:
```
?skin=red
```

while the actual file was:
```
red.php
```

This indicated that the backend was likely appending the extension automatically.
### Lesson:
Whenever an input controls files, templates, or resources, investigate:
- Does the backend add extensions?
- Does it modify paths?
- Is input sanitized?
- Can directory traversal bypass restrictions?

Small transformations can completely change the attack surface.

---

## 5. Do Not Get Tunnel Vision on Interesting Clues

### Mistake:
I saw:
```
allow_url_fopen = On
```

in `phpinfo()` and focused heavily on RFI.

However, there was no clear vulnerable endpoint or code path that allowed file inclusion.

### Lesson:
A configuration setting is not automatically a vulnerability.
When you find a possible file inclusion: 
1. Identify the parameter controlling the file. 
2. Understand how the backend processes the input. 
3. Test the simplest and most likely cases first: 
	- Local File Inclusion (LFI): 
		- Can I read local files using path traversal? 	
	- Remote File Inclusion (RFI): 
		- Can I make the application include a remote resource?

Always ask:
> "Do I have a way to reach this feature?"

A vulnerability requires both:
1. A dangerous capability.
2. User-controlled input reaching that capability.

---

## 6. Keep Multiple Hypotheses Alive

### Mistake:
I committed too quickly to certain ideas:
- `webmaster@support.thm` as the username.
- RFI because of `allow_url_fopen`.
### Lesson:
Do not treat hypotheses as facts.

Instead:
```
Possible username:
- help@support.thm
- webmaster@support.thm
- admin@support.thm
```

Test multiple reasonable possibilities instead of forcing one theory.

---
# General Rules

- Enumerate systematically before exploiting.
- Always inspect source code.
- Always monitor requests with Burp.
- Test API endpoints with different HTTP methods.
- Prioritize direct evidence over assumptions.
- When stuck, revisit observations instead of only creating new theories.
- Ask:
  - "What did I observe?"
  - "What assumption am I making?"
  - "How can I verify it?"