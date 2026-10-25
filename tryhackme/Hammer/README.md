Hammer CTF — Write-up
Platform: TryHackMe
Difficulty: Not specified in the supplied notes
Focus areas: Web enumeration, password-reset logic, rate-limit bypass, JWT tampering, privilege escalation
Status: Both requested objectives were completed in the supplied notes.

Objectives
1. Find the flag displayed after logging in to the dashboard.
2. Read the contents of /home/ubuntu/flag.txt.
Lab safety: This write-up describes activity against an authorized CTF target. Replace the example target IP with the IP assigned to your own lab instance. Do not run these techniques against systems without explicit permission.

Summary
The initial web enumeration identified port 1337 as the web application. Directory discovery revealed application paths and an exposed error log containing the account email tester@hammer.thm. The password-reset flow used a four-digit recovery code with a short validity window and rate limiting. In the lab, the reset could be completed by automating code attempts while varying the X-Forwarded-For header as accepted by the target.

After signing in, the dashboard displayed the first flag. The dashboard also exposed a command-execution feature with restrictions and session behavior. Intercepting its request showed a JWT bearer token. The notes describe finding a signing key in an exposed file, then modifying the token's role and key identifier (kid) to obtain administrator access. With that access, the command endpoint was used to read /home/ubuntu/flag.txt.
1. Reconnaissance and service discovery
Step 1 — Scan the target
Run a basic port scan against the IP assigned to the TryHackMe machine:
nmap -sC -sV -oN nmap-initial.txt <TARGET_IP>
- -sC runs Nmap's default scripts.
- -sV attempts to identify service versions.
- -oN saves the results to a text file.
Observation: The supplied notes report three open ports. Port 1337 stood out as the likely web application, so it was investigated further.
Step 2 — Open the web application
Browse to:
http://<TARGET_IP>:1337/
The application presented a login page and a Forgot your password? link. The page's source referenced resource paths beginning with hmr_, which provided a useful lead for content discovery.
2. Directory discovery and information disclosure
Step 3 — Enumerate web paths
Use a directory-discovery tool against the lab web service. For example:
dirsearch -u http://<TARGET_IP>:1337/ -w <WORDLIST_PATH>
The notes report discovery of paths including hmr_css, hmr_images, hmr_js, and hmr_logs.
Step 4 — Inspect discovered resources
Browse the discovered paths and review any files that are exposed. In this challenge, hmr_logs/error.logs was reported to contain application error messages and the email address:
tester@hammer.thm
Why this mattered: The log disclosure revealed a valid account identifier, which could be used in the password-reset workflow. Error logs should not be publicly accessible in a production application.
3. Password-reset weakness
Step 5 — Start the password-reset flow
Return to the login page, select Forgot your password?, and submit the discovered account email:
tester@hammer.thm
The reset page requested a four-digit recovery code and showed a countdown of roughly 180 seconds.
Step 6 — Inspect reset behavior
Submit a test code and inspect the request in the browser's Developer Tools Network panel. The request was handled by reset_password.php. The supplied notes report a rate-limit allowance of nine attempts.

This indicated two constraints to investigate in the authorized lab:
- The recovery code had a short validity period.
- Repeated guesses were rate limited.
Step 7 — Automate the recovery-code test in the lab
The notes describe a Python script that submitted candidate four-digit codes and varied the X-Forwarded-For request header within the 127.0.0.0/8 range. The target accepted this header for rate-limit accounting, allowing the challenge's rate limit to be bypassed. The script completed before the recovery-code window expired and set the new password to Password123.

Result reported in the notes: The script found the recovery code, and the password was reset successfully.
Security lesson: X-Forwarded-For is user-controllable unless a trusted reverse proxy removes and rewrites it. Applications must not trust arbitrary client-supplied forwarding headers to determine rate-limit identity. Apply limits using a trusted client identity and account-level controls, and add monitoring and lockout protections appropriate to the threat model.

For a public write-up, avoid publishing a reusable brute-force script or a live recovery code. Explain the flaw and the observed result, and keep any exploit implementation limited to the isolated CTF environment.
Step 8 — Sign in and capture the first flag
Return to the login page and authenticate using:
- Email: tester@hammer.thm
- Password: Password123 (the password set during the lab workflow)
The dashboard welcomed the user as Thor and displayed the first flag:
THM{AuthBypass3D}
Record the flag as the result for objective 1.
4. JWT analysis and administrator access
Step 9 — Inspect the dashboard command request
The dashboard contained a command input. The notes report that some commands were restricted and that the session appeared to log out quickly after a command was submitted.

Use the browser's Network panel or an intercepting proxy such as Burp Suite to inspect the request sent to execute_command.php. The observed request used a bearer JWT in the Authorization header and JSON data containing a command field.

The notes also report that changing the request's persistentSession value to yes made the lab session more stable. Treat this as a challenge-specific observation, not a general authentication recommendation.
Step 10 — Review the exposed key file
The command endpoint's response exposed a directory listing that included 188ade1.key. The notes state that the file contained a signing key and that the key was saved as a text file for inspection.

Finding: A signing key should never be exposed through a web-accessible path or a command endpoint. Anyone who obtains a JWT signing secret may be able to forge tokens if the application trusts tokens signed with that secret.

For a public repository, do not commit the actual signing key, a complete live JWT, session cookies, or unredacted proxy captures.
Step 11 — Decode and inspect the JWT
The notes used a JWT decoder to inspect the token's header and payload. The payload included user data and a role claim; the header included an algorithm and a kid (key identifier).

Conceptually, the relevant claims looked like this:
{
  "data": {
    "email": "tester@hammer.thm",
    "role": "user"
  }
}
The example above is illustrative and intentionally omits token timestamps, user identifiers, signature material, and the actual key.
Step 12 — Identify the token-validation weakness
The notes report that the token was modified so the role claim became admin, and the kid was changed to reference the discovered key. The modified token was then signed using the exposed key and substituted for the original token in the intercepted request.

Result reported in the notes: The application accepted the modified token and granted administrator access.

This is a combination of key disclosure and improper trust in client-presented JWT claims. A role claim must not be trusted merely because it appears in a decoded token; the server must validate the signature and claims correctly, and signing keys must remain secret.
5. Read the second flag
Step 13 — Test command execution with the elevated token
With the modified token accepted in the authorized lab, the notes changed the JSON command value from a directory-listing command to a file-read command targeting:
/home/ubuntu/flag.txt
The request was sent to execute_command.php, and the response returned the file contents.

The supplied notes do not include the literal value of this second flag, so it is intentionally not guessed here. Add the exact value from your successful response before publishing if you want the repository to document both flags.
Results
Objective	Result
Flag after dashboard login	THM{AuthBypass3D}
Contents of /home/ubuntu/flag.txt	Retrieved according to the notes; literal flag not included in the supplied text


Vulnerabilities and lessons learned
1. Information disclosure: Web-accessible logs exposed a valid account email and useful application details.
2. Weak recovery-code defenses: A short numeric code and a client-spoofable rate-limit identity made automated guessing feasible in the lab.
3. Trusting forwarding headers: The target's rate limiting could be bypassed by manipulating X-Forwarded-For.
4. Signing-key exposure: The key file was accessible through the application workflow.
5. JWT privilege escalation: The application accepted a modified role claim after token re-signing with the exposed key.
6. Overpowered command feature: A dashboard command endpoint allowed sensitive file access once administrator authorization was bypassed.
Remediation recommendations
- Keep logs, configuration files, and signing keys outside the web root; deny direct web access to diagnostic files.
- Remove sensitive values from error messages and rotate any secret that may have been exposed.
- Use sufficiently unpredictable, short-lived recovery tokens; enforce per-account and per-source throttling on the server, add abuse monitoring, and invalidate reset tokens after use.
- Trust X-Forwarded-For only when it is inserted by a configured trusted proxy. Do not let arbitrary clients choose their rate-limit identity.
- Store JWT signing secrets securely, rotate compromised keys, and use a strict allowlist of accepted algorithms.
- Validate JWT signatures and relevant claims server-side. Resolve roles from trusted server-side authorization data rather than trusting a client-editable role claim.
- Restrict command execution to an explicit allowlist of safe operations, run with least privilege, and avoid exposing arbitrary operating-system commands through a web dashboard.
- Add regression tests for rate-limit bypass, unauthorized role changes, key-file access, and command endpoint authorization.
Evidence checklist for GitHub
Evidence screenshots uploaded to screenshots/ (original capture filenames retained). Review each image for redaction before publishing:
- [x] Nmap results showing the discovered services.
- [x] Directory-discovery results showing the relevant hmr_ paths.
- [x] The reset page and relevant request/response behavior.
- [x] The dashboard showing the first flag.
- [x] The JWT header/payload with tokens and signatures redacted.
- [x] The successful second flag response.
Uploaded files:
- screenshots/01-login-page.jpg
- screenshots/02-directory-enumeration.jpg
- screenshots/03-error-log-discovery.jpg
- screenshots/04-dashboard-first-flag.jpg
- screenshots/05-command-execution-flag.jpg
- screenshots/Screenshot From 2026-10-24 22-47-18.jpg
- screenshots/Screenshot From 2026-10-24 22-48-52.jpg
- screenshots/Screenshot From 2026-10-24 23-19-26.jpg
- screenshots/Screenshot From 2026-10-25 00-52-36.jpg
- screenshots/Screenshot From 2026-10-25 01-59-53.jpg
- screenshots/Screenshot From 2026-10-25 02-00-42.jpg
- screenshots/Screenshot From 2026-10-25 02-01-16.jpg
- screenshots/Screenshot From 2026-10-25 02-01-38.jpg
- screenshots/Screenshot From 2026-10-25 02-17-30.jpg
- screenshots/Screenshot From 2026-10-25 02-20-36.jpg
- screenshots/Screenshot From 2026-10-25 02-36-55.jpg
- screenshots/Screenshot From 2026-10-25 02-37-41.jpg
- screenshots/Screenshot From 2026-10-25 02-37-49.jpg
- screenshots/Screenshot From 2026-10-25 02-38-07.jpg
Redact before upload: bearer tokens, session cookies, the contents of 188ade1.key, recovery codes, private IPs if you prefer not to disclose them, and any other reusable credentials. Do not publish a working session token or signing secret.
Suggested repository layout
hammer-ctf/
├── README.md
└── screenshots/
    ├── 01-nmap.png
    ├── 02-directory-discovery.png
    ├── 03-password-reset.png
    ├── 04-dashboard-first-flag.png
    ├── 05-jwt-analysis-redacted.png
    └── 06-second-flag-redacted.png
Disclaimer
This report documents a TryHackMe-style training exercise using the supplied notes. Use the techniques only in systems for which you have explicit authorization. Verify all target-specific details and add the second flag from your own lab evidence before publishing.
