# ctf-writeups

CTF writeups: solved challenges, methodology, notes, and tooling from Capture The Flag
competitions.

## Repository Structure

```
ctf-writeups/
├── web/            # SQLi, XSS, SSRF, auth bypass, file upload
├── crypto/         # ciphers, RSA/ECC, hash cracking
├── pwn/            # buffer overflow, ROP, format strings, shellcode
├── reversing/      # decompilation, obfuscation, binary analysis
├── forensics/      # steganography, memory/disk imaging, pcap
├── osint/          # username trails, metadata, geolocation
├── misc/           # anything that doesn't fit elsewhere
├── skills/
│   └── ctf-writeup-generator/   # Hermes Agent skill that generates writeups
└── README.md
```

## Writeup Naming

`[platform]_[challenge-name]_writeup.md` — lowercase, hyphens for spaces.
Example: `hackthebox_lame_writeup.md`.

Each writeup carries: challenge metadata, summary, reconnaissance, step-by-step
solution with annotated commands, tools used, flag, key takeaways, and references.

## Writeup Generator

`skills/ctf-writeup-generator/` is a [Hermes Agent](https://hermes-agent.nousresearch.com/)
skill. Given your solving notes it detects the flag format, categorizes the challenge,
and emits the writeup markdown described above.

```bash
# Load it in a Hermes session
skill_view(name='ctf-writeup-generator')

# Then just say: "generate a writeup for <challenge> on <platform>"
```

It does not solve challenges for you — it documents the solution you already have.

## Solved Challenges

| Platform | Challenge | Category | Difficulty | Writeup |
|---|---|---|---|---|
| — | — | — | — | — |

## License

Code and tooling under [MIT](LICENSE). Writeup prose is the author's own.
