# ctf-writeups-tryhackme

TryHackMe room writeups. Each writeup covers the full approach — recon, exploitation, and post-exploitation — with screenshots at every step.

---

## Index

| Room | Difficulty | Topics | Writeup |
|---|---|---|---|
| Pickle Rick | Easy | Web enumeration, command injection, privilege escalation | [Read](Pickle-Rick.md) |
| Corridor | Easy | IDOR, URL pattern analysis, web enumeration | [Read](Corridor.md) |
| Neighbour | Easy | IDOR, source code review | [Read](Neighbour.md) |
| TakeOver | Easy | Subdomain enumeration, subdomain takeover, DNS | [Read](TakeOver.md) |
| HeartBleed | Easy | CVE exploitation, OpenSSL, network security | [Read](HeartBleed.md) |
| The Game | Easy | Static analysis, reverse engineering, Ghidra | [Read](The-Game.md) |
| Compiled | Easy | Binary analysis, decompilation, Ghidra | [Read](Compiled.md) |
| Smol | Medium | WordPress exploitation, plugin backdoor, password cracking | [Read](Smol.md) |
| Bricks Heist | Medium | WordPress forensics, malware analysis, crypto wallet tracing | [Read](Bricks-Heist.md) |
| Mr. Robot | Medium | WordPress brute force, RCE via theme editor, SUID privesc | [Read](MrRobot.md) |

---

## Methodology

Every writeup follows the same flow:

### 1. Recon
Start with connectivity check (`ping`) and a service scan (`nmap -sV`). Identify open ports and running services before touching anything else.

### 2. Enumeration
Go deeper based on what recon found. For web: check source, inspect JS, fuzz directories. For binaries: run `file`, check strings, import into Ghidra. For domains: enumerate subdomains with `ffuf` or `gobuster`.

### 3. Exploitation
Use the information gathered to get initial access or extract the flag. Follow the path of least resistance — credentials in source, IDOR via parameter manipulation, known CVE against a fingerprinted service version.

### 4. Post-Exploitation
Where the room requires deeper access: check SUID binaries (`find / -perm -4000`), inspect running services, look for world-readable sensitive files. Escalate privileges only as far as needed to complete the objective.

### 5. Cleanup & Notes
Document every command and screenshot. Write the explanation as if explaining to someone learning — not just a flag dump.

---

## Setup

All writeups are done on my own machine (Kali Linux) connected to the TryHackMe VPN. No AttackBox.

```bash
sudo openvpn your-tryhackme.ovpn
```

---

## Profile

[tryhackme.com/p/smrayyans](https://tryhackme.com/p/smrayyans)
