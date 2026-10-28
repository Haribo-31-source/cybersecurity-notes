# 🏴 DutyBound — HTB Write-up

> **Difficulty:** Medium  
> **OS:** Linux  
> **Focus:** Web Application Security, X11 Injection, Privilege Escalation  
> **Key Takeaways:** BCBP parsing, PostgreSQL RCE, Xlib fake input, systemd.path abuse

---

## 📋 Overview

DutyBound is a Linux-based Hack The Box machine that simulates a **self-checkout kiosk** system for an airport duty-free shop. The challenge is designed to teach three important attack vectors in sequence:

1. **Web exploitation** through a strict barcode parser
2. **Lateral movement** via X11 display server manipulation  
3. **Privilege escalation** through a misconfigured systemd service

---

## 🔍 Phase 1: Reconnaissance

### Initial Enumeration

A standard full-port scan reveals three open TCP ports:
- **22** — OpenSSH (not directly exploitable)
- **80** — HTTP (redirects to HTTPS)
- **443** — HTTPS (the real application)

### Key Observation 🔑

Pay close attention to the **SSL certificate** and the **HTTP redirect target**. They both reveal a custom hostname that the web application expects. 

👉 **Tip:** Without properly configuring your `/etc/hosts` file, you will never reach the actual application interface.

### UDP Scan?

> ⚠️ Skip UDP scanning on this machine. The attack surface is entirely TCP-based and web-focused.

---

## 🌐 Phase 2: Web Application Analysis

### The Kiosk Interface

The web application mimics an **airport self-checkout kiosk** where travelers can purchase snacks. The checkout flow requires a **boarding pass barcode** to be scanned.

### The Critical Endpoint 🔑

Intercepting traffic with a proxy tool reveals a POST endpoint at `/checkout/boarding-pass/verify` that accepts a JSON payload with a `raw` field. This is where the scanner would normally send barcode data.

**Important:** The scanner hardware is marked as "out of service," but the API still accepts user-supplied input. This is your entry point.

### Understanding the BCBP Format

The barcode follows the **IATA BCBP (Barcoded Boarding Pass)** standard. Key characteristics to discover:

- 📏 **Mandatory fields** (first 60 bytes) — strict format validation
- 🔢 **Length indicator** — the last two bytes of the mandatory section specify the hex length of the conditional tail
- 🛡️ **Validation rules** — certain fields only accept specific characters (A-Z, numeric, etc.)
- ⚠️ **One field is unsanitized** — find which part of the barcode gets passed directly to the database without filtering

### Testing Your Understanding

Try sending payloads of varying lengths and formats. The error messages returned by the API will reveal:
- Which format is expected (look for "M" as the first character)
- Minimum length requirements
- Which fields have strict character validation
- Where the parser stops reading the input

👉 **Tip:** Manually crafting valid BCBP payloads with dynamic lengths is error-prone. Consider writing a small helper script to automate length calculations and proper formatting.

---

## 💉 Phase 3: SQL Injection Discovery

### Identifying the Vulnerability

Once you find the unsanitized field, test for SQL injection using a single quote (`'`). A successful injection will trigger a **500 Internal Server Error** with a database syntax error message.

### Key Characteristics

- The backend database is **PostgreSQL**
- The connection runs with elevated privileges
- **Stacked queries** are supported, allowing you to execute additional SQL commands after closing the original query

👉 **Tip:** Before attempting full exploitation, test if the database executes your injected commands using a **time-based blind SQL injection** (e.g., sleep functions). This confirms code execution capability.

---

## 🚪 Phase 4: Gaining Foothold (Reverse Shell)

### PostgreSQL RCE

PostgreSQL has a built-in capability to execute operating system commands directly through a specific SQL statement. 

**Key considerations:**
- The command requires appropriate database privileges
- Long-running commands may cause HTTP timeouts, but the command will still execute
- Proper quote escaping is critical when nesting shell commands inside SQL strings

### Important: JSON Payload Gotcha 🚨

Since you're sending a JSON payload, **double quotes inside your command must be escaped** with backslashes (`\"`). This is a common source of failure:

```
❌ Wrong: 'bash -c 'inner command''      (premature quote closure)
✅ Right: 'bash -c \"inner command\"'     (proper escaping)
```

---

## 👤 Phase 5: User Privilege Escalation (dutyfree-kiosk)

### The Problem

After getting a shell, you'll find yourself as the `postgres` user. However, the `user.txt` flag belongs to the `dutyfree-kiosk` user, and direct privilege escalation paths (like SUID binaries) don't yield results.

### The X11 Discovery 🔑

Enumerate the `/tmp` directory carefully. You'll find:
- A `.X11-unix` directory with **world-writable** permissions
- A `serverauth` file belonging to the kiosk user
- An `.X0-lock` file indicating an active X display

**What this means:** The machine is running a graphical desktop environment with the kiosk user logged in, and the X11 socket allows any user to connect without authentication.

### The Attack Vector

This machine uses a minimal window manager with a **global keyboard shortcut** configured to open a terminal. By connecting to the X display server, you can:

1. **Inject fake keyboard events** to trigger the shortcut
2. **Type commands** into the newly opened terminal
3. **Press Enter** to execute those commands as the kiosk user

### Tools for X11 Injection

Several approaches work:
- **`xdotool`** — Command-line X11 automation tool
- **Python's `python-xlib` library** — Programmatic input injection

👉 **Tip:** Timing is important. After sending the keyboard shortcut, wait briefly for the terminal window to open before injecting the typed command.

### Getting the User Shell

Your injected command should establish a reverse shell back to your machine. This gives you a stable session as the `dutyfree-kiosk` user, where you can read the user flag.

---

## 👑 Phase 6: Root Privilege Escalation

### Group Membership Discovery

Run `id` or `groups` to discover the `dutyfree-kiosk` user's group memberships. One particular group will stand out — it has write access to specific directories that standard users cannot modify.

### Finding Writable Files

Use `find` to locate files and directories owned by this group that are writable:

```bash
find / -group <group_name> -writable 2>/dev/null
```

👉 **Tip:** Focus on application directories like `/opt` where custom services often reside.

### The Systemd Path Trigger 🔑

You'll discover a **CSV file** containing duty rates for various product categories. More importantly, a `systemd.path` unit is configured to watch this file and trigger a service whenever it's modified.

The service runs a shell script **as root** that:
1. Reads the CSV file line by line
2. Validates certain columns (numeric checks)
3. Uses a dangerous shell command to assign values to variables
4. Updates a PostgreSQL table with the new rates

### The Vulnerability

Examine the shell script closely. Notice:
- ✅ Columns 2 and 3 are validated as numeric only
- ❌ **Column 1 has no validation at all**
- ⚠️ Column 1 is passed to a dangerous shell builtin that executes strings as commands

### Crafting the Payload

Your payload must:
1. ✅ Preserve the required product categories (the script validates their presence)
2. ✅ Maintain valid 3-column CSV format
3. ✅ Contain no commas in the injected command (because `,` is the field separator)
4. ✅ Trick the shell builtin into executing your command before the `=` assignment

👉 **Tip:** Use command chaining operators to execute your reverse shell command before the variable assignment happens. The assignment will still occur (preventing errors) but your command will run first.

### Triggering the Exploit

After modifying the CSV file, the systemd path unit may need the file's modification time to be updated to trigger. A simple `touch` command on the file is usually sufficient.

Listen on your attacker machine, modify the file, and wait for the root reverse shell to connect.

---

## 🎯 Key Learnings

### Security Concepts

1. **Parser bypass techniques** — Understanding format specifications (like BCBP) can reveal injection points
2. **Length-based validation** — Many parsers use length fields to determine where data ends
3. **X11 security model** — World-writable X sockets allow any local user to inject inputs
4. **Eval injection** — Shell scripts using `eval` with unsanitized input are extremely dangerous
5. **systemd.path** — File watch mechanisms can be weaponized for privilege escalation

### Tools & Techniques

- Burp Suite for HTTP interception
- Python xlib for X11 automation
- PostgreSQL `COPY TO PROGRAM` for RCE
- JSON payload escaping
- Bash command chaining

---

## 🛡️ Mitigations (Defensive Perspective)

If you were securing this application:

- ✅ Implement proper input validation using parameterized queries
- ✅ Restrict X11 socket permissions to the display owner only
- ✅ Avoid `eval` in shell scripts; use associative arrays instead
- ✅ Validate all columns in CSV processing, not just numeric ones
- ✅ Use `xauth` properly and disable `-nolisten tcp` alternatives

---

## 📚 References

- [IATA BCBP Standard Documentation](https://www.iata.org/en/publications/standards/bar-boarding-pass/)
- [PostgreSQL COPY TO PROGRAM](https://www.postgresql.org/docs/current/sql-copy.html)
- [Python Xlib Documentation](https://python-xlib.sourceforge.io/)
- [systemd.path Man Page](https://www.freedesktop.org/software/systemd/man/systemd.path.html)

---

<div align="center">

**Hope you enjoyed solving DutyBound! Feel free to reach out if you have questions.** ⭐

*Happy hacking!* 💻🔒

</div>
