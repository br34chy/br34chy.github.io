---
layout: post
title: "PG Lab - Access"
date: 2026-01-30 00:00:00 -0400
categories: [Labs, Proving Grounds]
tags: [Active Directory]
years: ['2026']
comments: true
---

![Access](/assets/img/blog/PGLabs/Access/Access.png)

## From Restricted File Upload to SYSTEM

![labdescr](/assets/img/blog/PGLabs/Access/labdescr.png)

**Access (PG Lab)** began with a web-layer attack. A **restricted file upload** proved insufficient, unfolding into a chain of access weaknesses. This aligns with James Reason's Swiss cheese model: *security failures occur when multiple small weaknesses line up.* 

1. restricted file upload flaw
2. service account execution context
3. service to service trust abuse
4. SYSTEM-level escalation

## Initial Enumeration

**Nmap** revealed an **Active Directory** machine. **Port 80** stood out immediately, **PHP on Windows** has historically widened the *attack surface*. When a domain controller also runs a web app, the web layer is usually the *open gap*.

![nmap](/assets/img/blog/PGLabs/Access/nmap.png)

The webpage revealed a **file upload feature**. It appeared to be restricted, as multiple PHP-related extensions e.g.`.php`, `.phtml`, and `.phar` were rejected.

![buytix1](/assets/img/blog/PGLabs/Access/buytix1.png)

![buytix2](/assets/img/blog/PGLabs/Access/buytix2.png)

Uploads using **alternative extensions** such as `.png` were accepted, raising the question of **where those files were actually stored**.

![buytix3](/assets/img/blog/PGLabs/Access/buytix3.png)

![buytix4](/assets/img/blog/PGLabs/Access/buytix4.png)

This is where `ffuf` comes in. I filtered out `200` and `404` responses to reduce noise. The output isn’t pretty, but the `301` populated in few top rows: `uploads`, `assets`, and `forms`.

![ffuf](/assets/img/blog/PGLabs/Access/ffuf.png)

Has to be `uploads`.

![uploads](/assets/img/blog/PGLabs/Access/uploads.png)

The restriction appeared to be **extension-based**. Given that PHP was already in use, this hinted at a possible execution path rather than a hard block.

![google1](/assets/img/blog/PGLabs/Access/google1.png)

*“httpd allows for decentralized management of configuration via special files placed inside the web tree.”* The lab’s name, **Access**, and uploads landing under the **web root** pointed directly toward `.htaccess`.

![google2](/assets/img/blog/PGLabs/Access/google2.png)

![google3](/assets/img/blog/PGLabs/Access/google3.png)

![google4](/assets/img/blog/PGLabs/Access/google4.png)

![google5](/assets/img/blog/PGLabs/Access/google5.png)

**File handling** could be manipulated via `.htaccess`, indicating that *PHP execution remained reachable* despite the upload restrictions.

## Initial Access

A **reverse shell payload** was embedded within `purchase.pdf` and uploaded through the application.

![revshell](/assets/img/blog/PGLabs/Access/revshell.png)

![purchasepdf](/assets/img/blog/PGLabs/Access/purchasepdf.png)

A `.htaccess` file was then used to influence file handling within the upload directory, causing files with a `.pdf` extension to be interpreted as PHP.

![htaccess](/assets/img/blog/PGLabs/Access/htaccess.png)

![hiddenfile](/assets/img/blog/PGLabs/Access/hiddenfile.png)

A **listener** was prepared to receive the incoming connection.

![reversed](/assets/img/blog/PGLabs/Access/reversed.png)

![whoami](/assets/img/blog/PGLabs/Access/whoami.png)

A foothold was established. 

## Credential Access

Kerberos enumeration using native PowerShell scripts proved unreliable in this environment due to platform and execution constraints when operating from macOS with a Kali VM.

**An alternative approach the next day produced the expected results; while the Proving Grounds lab IP retained the same final octet, my local `tun0` IP differed, explaining screenshot discrepancies.**

To avoid further delays, **[Rubeus][Rubeus]** was used. Credit to [routezero.security][routezero.security] for surfacing this tool. Created by Benjamin Delpy and Will Schroeder (GhostPack), Rubeus is purpose-built for interacting with Kerberos in Active Directory environments. The name comes from Rubeus Hagrid, who owned Fluffy (Cerberus), a three headed dog in Harry Potter.

![rubeus](/assets/img/blog/PGLabs/Access/rubeus.png)

Using a [precompiled binary][precompiled binary] due to macOS limitations.

![r3motecontrol](/assets/img/blog/PGLabs/Access/r3motecontrol.png)

One Kerberoastable service account `svc_mssql` was identified and taken forward for offline analysis.

![kerberoast](/assets/img/blog/PGLabs/Access/kerberoast.png)

Captured hashes were then processed to recover service account credentials with `hashcat`.

![hashcat](/assets/img/blog/PGLabs/Access/hashcat.png)

![cracked](/assets/img/blog/PGLabs/Access/cracked.png)

![password](/assets/img/blog/PGLabs/Access/password.png)

## Lateral Movement

**[Invoke-RunasCs.ps1][Invoke-RunasCs.ps1]** allows execution of a process under another user or service account (svc_mssql in this case) without requiring an interactive logon session. A new listener was set up to receive the spawned process, effectively capturing the context switch as a command shell.

![invokerunascs1](/assets/img/blog/PGLabs/Access/invokerunascs1.png)

![invokerunascs2](/assets/img/blog/PGLabs/Access/invokerunascs2.png)

![mssql](/assets/img/blog/PGLabs/Access/mssql.png)

Two privileges are worth eyeing: **SeMachineAccountPrivilege**, and **SeManageVolumePrivilege**. The former enables adding a computer to the domain for delegation-based escalation, covered in [Resourced: Delegation Misconfig][Resourced: Delegation Misconfig], and is not applicable here. The latter enables a **local-to-SYSTEM** escalation via service trust abuse, making it the correct choice in this context. A precompiled [SeMangeVolumePrivilege exploit][SeMangeVolumePrivilege exploit] was used. 

![smve](/assets/img/blog/PGLabs/Access/smve.png)

`icacls` confirmed `NT AUTHORITY\SYSTEM` and `TrustedInstaller` with **Full Control**. `System32` become writable, allowing payload placement in trusted locations and service-level execution to complete the escalation.

![icacls](/assets/img/blog/PGLabs/Access/icacls.png)

A final reverse shell was generated with `msfvenom`, disguised as `tzres.dll` (timezone resources), and placed in `C:\Windows\System32\wbem`. Because WBEM is trusted, executing `systeminfo` caused the DLL to load, triggering code execution.

![msfvenom](/assets/img/blog/PGLabs/Access/msfvenom.png)

![tzresdll](/assets/img/blog/PGLabs/Access/tzresdll.png)

![wbem](/assets/img/blog/PGLabs/Access/wbem.png)

Pwned.

![pwned](/assets/img/blog/PGLabs/Access/pwned.png)

## Remedies

1. **Harden file upload handling**

    1. enforce server-side MIME validation, not extension checks

    2. store uploads outside the web root

    3. disable directory listing on upload paths

2. **Disable or tightly scope `.htaccess`**

    1. set `AllowOverride None` where not explicitly required

    2. prevent executable directives in writable directories

    3. separate writable paths from execution contexts

3. **Restrict dangerous local privileges**

    1. audit and limit privileges such as `SeManageVolumePrivilege`

    2. remove high-impact rights from service accounts

    3. ensure service accounts cannot influence system directories

4. **Protect trusted system paths**

    1. audit write access under `System32` (including subdirectories like `wbem`)

    2. retain write permissions only for `TrustedInstaller` and `SYSTEM`

    3. monitor for unexpected DLL writes or replacements in trusted locations


[Rubeus]:https://github.com/GhostPack/Rubeus
[routezero.security]:https://routezero.security/2024/11/04/proving-grounds-practice-access-walkthrough/
[precompiled binary]:https://github.com/r3motecontrol/Ghostpack-CompiledBinaries
[Invoke-RunasCs.ps1]:https://github.com/antonioCoco/RunasCs
[Resourced: Delegation Misconfig]:https://br34chy.github.io/posts/PGLab-Resourced/
[SeMangeVolumePrivilege exploit]:https://github.com/CsEnox/SeManageVolumeExploit/releases