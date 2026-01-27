---
layout: post
title: "PG Lab - Heist"
date: 2026-01-26 00:00:00 -0400
categories: [Labs, Proving Grounds]
tags: [Active Directory]
years: ['2026']
comments: true
---

## SSRF via NTLMv2 Abuse

**Heist (PG Lab)** demonstrates an **SSRF (Server-Side Request Forgery)** abuse path that enables **NTLMv2 credential capture and relay**, leading to local privilege escalation on a Windows system. [HuWanyu][HuWanyu]'s public walkthrough was used as a reference during analysis. His endgame approach is clever, short, and sweet.

![labdescr](/assets/img/blog/PGLabs/Heist/labdescr.png)

## Initial Enumeration

**Kerberos + LDAP + DNS + kpasswd** on the same host, that combo exists for AD and basically nothing else. SMB shows **message signing enabled and required**.

![nmap](/assets/img/blog/PGLabs/Heist/nmap.png)

JavaScript `alert()` was ready in `test.html` to provide a visible execution marker. Confirmed the **server-side request succeeded**.

![execute](/assets/img/blog/PGLabs/Heist/execute.png)

![executed](/assets/img/blog/PGLabs/Heist/executed.png)

## Credential Access

Activated **Responder** to passively wait for outbound requests and actively capture **NTLM authentication** by impersonating the requested service.

![responder](/assets/img/blog/PGLabs/Heist/responder.png)

![NTLMv2hash](/assets/img/blog/PGLabs/Heist/NTLMv2hash.png)

The captured NTLMv2 hash was cracked offline with **hashcat and RockYou**, yielding valid credentials.

![hashcat](/assets/img/blog/PGLabs/Heist/hashcat.png)

## Initial Access

An **Evil-WinRM** session was established as `heist\enox`, confirming initial access to the domain.

![evil-winrm](/assets/img/blog/PGLabs/Heist/evil-winrm.png)

![whoami](/assets/img/blog/PGLabs/Heist/whoami.png)

## Active Directory Enumeration

Mapped domain relationships, privileges, and potential escalation paths with `bloodhound-python` and **BloodHound**.

![bh-python](/assets/img/blog/PGLabs/Heist/bh-python.png)

![bloodhound](/assets/img/blog/PGLabs/Heist/bloodhound.png)

After marking enox as **ADD TO OWNED**, outbound control revealed `ReadGMSAPassword`, allowing the **gMSA password hash** to be retrieved and reused via **pass-the-hash**.


![gmaspasswordreader](/assets/img/blog/PGLabs/Heist/gmsapasswordreader.png)

## Privilege Escalation

The quickest (and laziest, though risky) way to obtain `GMSAPasswordReader.exe` is to grab a precompiled binary from GitHub. Building it from source requires Visual Studio, which isn’t available in my Kali VM. I pulled the binary from [expl0itabl3][expl0itabl3]'s Github.

![svc_apache_GMSA](/assets/img/blog/PGLabs/Heist/svc_apache_GMSA.png)

Dumped the gMSA password hashes for `svc_apache$`, updated /etc/hosts, and then pass-the-hash in Evil-WinRM.

![etchost](/assets/img/blog/PGLabs/Heist/etchost.png)

![svc_apache_all](/assets/img/blog/PGLabs/Heist/svc_apache_all.png)

Running `whoami /all` showed `svc_apache$` as a **Domain Computer** account with `SeRestorePrivilege`. This privilege allows protected files to be overwritten by bypassing normal permission checks, making it dangerous when abused. Initial enumeration also showed `RDP` exposed on `port 3389`. Since `utilman.exe` can be triggered before login over RDP, the binary was replaced with `cmd.exe`, resulting in a **SYSTEM shell** being spawned at the login screen. 

![cmdexe](/assets/img/blog/PGLabs/Heist/cmdexe.png)

## Remedies (Legacy environments)

This behavior cannot be patched because it is an intended design. NTLMv2 remains widely enabled in Active Directory environments, making SSRF-induced credential leakage a realistic risk where outbound authentication is not restricted.

1. **Constrain server-side requests**
    1. block access to internal, loopback, and link-local addresses
    2. disallow UNC paths and non-HTTP(S) protocols e.g. `\\server\share`
    3. prevent automatic credential forwarding on outbound requests

2. **Reduce NTLM exposure**
    1. disable NTLM where possible
    2. prefer Kerberos for domain authentication
    3. monitor and alert on outbound NTLM authentication attempts

3. **Protect gMSA credentials**
    1. audit and tightly restrict `ReadGMSAPassword`
    2. scope gMSAs only to required services
    3. remove unnecessary service account permissions

4. **Limit high-impact local privileges**
    1. audit accounts with `SeRestorePrivilege`
    2. remove it from non-administrative service accounts

5. **Harden RDP pre-auth execution paths**
    1. restrict write access to `System32`
    2. monitor integrity of accessibility binaries (utilman.exe, sethc.exe)


[HuWanyu]:https://medium.com/@huwanyu94/proving-grounds-practice-heist-walkthrough-fdb639824fe8
[expl0itabl3]:https://github.com/expl0itabl3/Toolies/blob/master/GMSAPasswordReader.exe