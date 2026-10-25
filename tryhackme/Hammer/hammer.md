# TryHackMe: Hammer

## 1. Room overview

**Hammer** is a TryHackMe web application challenge. The target is an HTTP service running on port `1337` of the assigned machine IP. The challenge involves web reconnaissance, password recovery abuse, rate-limit bypass, command execution, JWT analysis, and privilege escalation to read `/home/ubuntu/flag.txt`.

Two flags are verified in the evidence:

- Dashboard flag: `THM{AuthBypass3D}`
- Command-execution flag: `THM{RUNANYCOMMAND1337}`

## 2. Objective and lab setup

- Attacker machine: Kali Linux (zsh, Chromium, Burp Suite Community Edition).
- Target: `http://<target-ip>:1337/` (observed IPs in screenshots include `10.49.144.218`, `10.49.138.219`, `10.48.141.153`).
- Tools used: browser DevTools, `dirsearch`, `sed`, Burp Suite Repeater/Proxy, jwt.io, Python brute-force script.

## 3. Nmap enumeration and port 1337

The service of interest is the web application on port `1337`. (Exact nmap output is not included in the supplied notes; treat the port identification as an observed/assumed step.)

## 4. Web reconnaissance and directory discovery

A wordlist was prepared and used with `dirsearch`:

```bash
sed 's/^/hmr_/' big.txt > hmr_big.txt
dirsearch --url http://<target-ip>:1337 -w /home/shivam/hmr_big.txt
```

Discovered directories:

- `/hmr_css/`
- `/hmr_images/`
- `/hmr_js/`
- `/hmr_logs/`

The error log at `/hmr_logs/error.log` exposed Apache errors and references to paths such as `/restricted-area`, `/admin-login`, `/var/www/html/`, and `/home/hammer/test.php`.

## 5. Error logs and recovery email

The recovery flow is initiated from the login page via **Forgot your password?**. The reset page is `reset_password.php` and accepts a 4-digit recovery code within a time limit (observed values: 121s, 143s, 174s).

## 6. Password recovery and rate-limit behavior

The reset code was brute-forced with a Python script (`hammer.py`):

```text
[+] Sending the password reset request.
[+] Starting the code brute-force.
[+] Found the recovery code: 3466
[+] Sending the new password request.
[+] Password is set to Password123
```

Rate limiting was bypassed by rotating the `X-Forwarded-For` header across `127.0.0.1` to `127.0.255.255`. The response header `Rate-Limit-Pending` was observed (value `6` in one capture).

Recovered credentials:

- Email: `tester@hammer.thm`
- Password: `Password123`

## 7. Dashboard login and first flag

After login, the dashboard greets the user and displays the first flag:

- URL: `http://<target-ip>:1337/dashboard.php`
- Welcome: `Welcome, Thor! - Flag: THM{AuthBypass3D}`
- Role: `user`
- Command input accepts commands such as `ls` or `whoami`; restricted commands return `Command not allowed`.

**Flag 1:** `THM{AuthBypass3D}`

## 8. Command execution restrictions and session behavior

The endpoint `/execute_command.php` accepts JSON:

```json
{"command":"ls"}
```

Restricted commands are rejected with `Command not allowed`. A valid session (`PHPSESSID`) and JWT (`token` cookie / `Authorization: Bearer ...`) are required.

## 9. JWT analysis, signing-key discovery, and authorization weaknesses

The JWT header uses `alg: HS256` and a `kid` pointing to a local key file:

- Observed `kid`: `/var/www/mykey.key` and `/var/www/html/188ade1.key`

A key file `188ade1.key` (or `188ade1.txt`) contained a 32-hex-character secret:

```text
56058354efb3daa97ebab00fabd7a7d7
```

Using jwt.io, a new JWT was forged with `role: admin` and signed with the discovered secret. The forged token was then used in the `Authorization` header and `token` cookie, granting administrative command execution.

## 10. Administrative access and second flag

With the forged admin JWT, the following request was sent via Burp Repeater:

```http
POST /execute_command.php HTTP/1.1
Host: <target-ip>:1337
Content-Type: application/json
Authorization: Bearer <forged-admin-jwt>

{"command":"cat /home/ubuntu/flag.txt"}
```

Response:

```json
{"output":"THM{RUNANYCOMMAND1337}\n"}
```

**Flag 2:** `THM{RUNANYCOMMAND1337}`

## 11. Lessons learned and security recommendations

- Enforce robust, server-side rate limiting that cannot be bypassed via spoofed `X-Forwarded-For` headers.
- Use cryptographically strong, unpredictable recovery codes and secure delivery channels.
- Do not expose signing keys in web-accessible or predictable paths; rotate and protect keys.
- Validate JWTs strictly (algorithm, issuer, audience, expiry) and avoid trusting client-controlled `kid` without allow-listing.
- Enforce authorization server-side; do not rely on client-side role checks.
- Restrict command execution endpoints or remove them entirely; never pass user input directly to shell commands.
- Avoid leaking sensitive paths and errors in publicly accessible logs.

## 12. Evidence screenshots

- `evidence/01-dashboard-flag.jpg` — Dashboard showing `THM{AuthBypass3D}`.
- `evidence/02-login.jpg` — Login page.
- `evidence/03-recovery-code.jpg` — Recovery code entry with timer.
- `evidence/04-dirsearch-errorlogs.jpg` — Directory discovery and Apache error log.
- `evidence/05-command-execution-flag.jpg` — Burp response containing `THM{RUNANYCOMMAND1337}` (request tokens redacted).

## References

- TryHackMe Hammer room.
- Observed endpoints: `/`, `/login`, `/reset_password.php`, `/dashboard.php`, `/execute_command.php`, `/logout.php`, `/hmr_logs/error.log`.
path: /home/shivam/ctf-writeups/tryhackme/Hammer/hammer.md