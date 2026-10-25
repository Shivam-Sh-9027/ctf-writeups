---
name: ctf-writeup-generator
description: Generate professional CTF writeups from solving notes.
version: 0.1.0
author: Shivam Sharma (Shivam-Sh-9027), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [ctf, writeup, markdown, security, documentation]
    related_skills: [bug-bounty-writeups, report-generation]
---

# CTF Writeup Generator

Turns raw solving notes into a professional, publish-ready CTF writeup: detects the
flag format, categorizes the challenge, and emits structured markdown with metadata,
step-by-step solution, code blocks, tools, flag, and takeaways. Covers Jeopardy CTFs,
HackTheBox, TryHackMe, OffSec, and picoCTF. Does not do the solving itself — it
documents a solution you already have, so the quality of the output is bounded by the
notes you feed it.

## When to Use

- User says "generate a CTF writeup" / "document my CTF solution" / "writeup for [challenge]".
- User pastes solving notes, terminal output, or an exploit and wants it formatted.
- User asks to extract and format flags from their notes.
- User wants a portfolio writeup for a CTF or a training-material writeup.

Don't use for: reporting a vulnerability you found in someone else's system (use
`bug-bounty-writeups`), or QA/security test reports (`report-generation`).

## Prerequisites

- The solving notes themselves: challenge name, platform, category, and the steps taken.
- `terminal` available for any command verification you run yourself.
- Optional: pandoc for PDF export (`pandoc -V geometry:margin=2cm`).
- Ask for anything missing that changes the writeup (platform, difficulty, points) rather
  than inventing it — a guessed platform produces a wrong flag-format check.

## Procedure

1. **Gather information.** Completion: you have challenge name, platform/CTF, category
   (or enough description to detect it), difficulty/points, and the solving steps.
   Ask once, in one message, for whatever is absent.
2. **Detect the flag.** Completion: flag matches one of the known formats below, or you
   have flagged the format as custom/unconfirmed to the user.
3. **Categorize.** Completion: exactly one category from the table below, chosen by
   keywords and tools, not by the challenge's own label.
4. **Extract the technical content.** Completion: tools and commands are enumerated and
   dead ends are separated from the working path.
5. **Generate the writeup.** Completion: markdown matching the template below, saved as
   `[platform]_[challenge-name]_writeup.md` (lowercase, hyphens for spaces).
6. **Validate.** Completion: flag format checked against platform, every code block has a
   language tag, and no credentials or internal hostnames leaked into the text.

## Flag Format Detection

Match case-sensitively first; only fall back to case-insensitive if nothing hits.

| Platform | Format | Example |
|---|---|---|
| Generic | `CTF{...}`, `FLAG{...}`, `flag{...}` | `CTF{th3_fl4g}` |
| HackTheBox | `HTB{...}` | `HTB{...}` |
| TryHackMe | `THM{...}` | `THM{...}` |
| picoCTF | `picoCTF{...}` | `picoCTF{...}` |
| Shaastra | `SHAASTRA{...}` | `SHAASTRA{...}` |
| Custom | user-supplied regex | — |

Validate the closing brace is present and the payload is non-empty. If the note says
"the flag was" and gives text that matches no known format, report it as unconfirmed
instead of silently rewriting it into a known format.

## Challenge Categories

Pick by keywords/tools in the notes:

| Category | Signals |
|---|---|
| Web Exploitation | SQLi, XSS, CSRF, auth bypass, SSRF, file upload, Burp, sqlmap, ffuf |
| Binary Exploitation | buffer overflow, ROP, format string, heap, GDB, pwntools |
| Reverse Engineering | decompilation, obfuscation, Ghidra, IDA, radare2, strings |
| Cryptography | ciphers, RSA/ECC, hash cracking, padding oracle, CyberChef, hashcat |
| Forensics | steganography, memory/disk imaging, pcap, Volatility, binwalk, Wireshark |
| OSINT | usernames, social media, WHOIS, metadata, geolocation |
| PWN | shellcode, privilege escalation, kernel, exploit dev |
| Miscellaneous | anything mixed or unclassifiable |

## Output Template

```markdown
# [Challenge Name] - [Platform] CTF Writeup

**Author**: Shivam-Sh-9027
**Date**: [YYYY-MM-DD]
**Category**: [Category]
**Difficulty**: [Easy/Medium/Hard]
**Points**: [points, if applicable]

## Summary
[2-3 sentences: what the challenge was and the core idea of the solution.]

## Challenge Description
[Original description, quoted or paraphrased.]

## Reconnaissance
[Enumeration, ports, files, endpoints, first observations.]

## Solution

### Step 1: [Phase name]
[Explanation with commands.]

```bash
# command used
```

### Step 2: [Next phase]
[Continue until the flag. Include dead ends as "What didn't work" where instructive.]

## Tools Used
- **Tool**: purpose

## Flag
```
FLAG{...}
```

## Key Takeaways
- [Learning point]

## References
- [Link]
```

Platform-specific additions: HackTheBox — machine IP, OS, user/root flag sections, attack
path summary. OffSec/OSCP — enumeration methodology, privilege-escalation chain, proof
screenshot references. Jeopardy — point value, solve time, category.

Writeup styles: **Academic** (theory background), **Speedrun** (essential steps only),
**Tutorial** (beginner-friendly, extra explanation), **Portfolio** (professional framing
for job applications). Default to Tutorial unless the user says otherwise.

## Pitfalls

- Never fabricate a step, command, or output to fill a gap — mark it `[not captured]`
  and tell the user what to supply.
- Never invent a flag. If the notes contain no flag, omit the Flag section and say so.
- Respect competition rules: a running CTF forbids publishing. Add a spoiler warning or
  hold the writeup until the CTF ends.
- Scrub credentials, API keys, internal hostnames/IPs, and absolute paths under your
  home directory before writing the file.
- A flag format that matches the platform's but looks "off" (wrong case, stray spaces)
  is a transcription error, not a new format — check before reporting.

## Verification

- File exists at the expected path: `read_file(path=...)`.
- Frontmatter/structure present: title line, metadata block, Summary, Solution, Flag,
  Key Takeaways all present in that order.
- Every fenced code block has a language tag (`bash`, `python`, `sql`, `c`, `asm`).
- Flag block is non-empty and matches the platform format.
- No secrets or `/home/...` paths in the file: `search_files(pattern='/home/', path=<writeup dir>)`.
