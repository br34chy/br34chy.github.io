---
layout: post
title: "PG Lab - Resourced"
date: 2026-01-07
 00:00:00 -0400
categories: [Labs, Proving Grounds]
tags: [Active Directory]
years: ['2026']
comments: true
---

![Resourced](/assets/img/blog/PGLabs/Resourced/Resourced.png)

## Delegation Misconfiguration

![labdescr](/assets/img/blog/PGLabs/Resourced/labdescr.png)

**Resourced (PG Lab)** demonstrates a classic **Active Directory delegation abuse** that escalates from a low-privileged domain user to **Domain Admin** through **Resource-Based Constrained Delegation (RBCD)**. A known RBCD abuse pattern is followed; **[Dpsypher][Dpsypher]'s public write-up** was used as a reference during analysis.

## Initial Enumeration

Service enumeration confirmed an **Active Directory** environment, with standard domain services exposed (**DNS, Kerberos, LDAP, SMB**).

![nmap1](/assets/img/blog/PGLabs/Resourced/nmap1.png)

`Enum4linux` identified a **low-privileged domain user**, `V.Ventz`, along with a note of **onboarding password** note `HotelCalifornia194!`.

![enum4linux](/assets/img/blog/PGLabs/Resourced/enum4linux.png)

Tested **anonymous access** through `smbclient`, followed by authentication using the discovered user with an onboarding password.

![smbclient](/assets/img/blog/PGLabs/Resourced/smbclient.png)
![smbrecursels](/assets/img/blog/PGLabs/Resourced/smbrecursels.png)

The `ntds.dit` file was exposed through a **misconfigured SMB share**, allowing it to be retrieved using SMB command `get` without prior domain administrator privileges. **NTDS.dit (New Technology Directory Services dot Directory Information Tree)** is the underlying database used by Active Directory to store domain objects and credential data.

![smbget](/assets/img/blog/PGLabs/Resourced/smbget.png)


## Credential Access

**Impacket** is a Python library and toolset that allows researchers to build, test, and simulate Windows network behavior without requiring a Windows host. It was ***never designed as an exploit framework***, but as **a protocol interaction toolkit**. Over time, security researchers realized something important: if you can faithfully speak a protocol, you can also abuse the *trust assumptions baked into it*. This made Impacket the foundation of modern Active Directory attack tooling.

`impacket-secretsdump` used `NTDS.dit` together with the `SYSTEM` hive to extract **domain account hashes**; the `SAM` and `SECURITY` hives were not required.

![impacket-secretsdump](/assets/img/blog/PGLabs/Resourced/impacket-secretsdump.png)
![hashes](/assets/img/blog/PGLabs/Resourced/hashes.png)
![crackstation](/assets/img/blog/PGLabs/Resourced/crackstation.png)

One recovered hash for `L.Livingstone` successfully authenticated against domain resources, indicated by the `(Pwn3d!)` result below.

![crackmapexe1](/assets/img/blog/PGLabs/Resourced/crackmapexec_command.png)
![crackmapexe2](/assets/img/blog/PGLabs/Resourced/crackmapexec_result.png)


## Initial Access

`ItachiUchiha` with a cracked password proved unstable due to repeated disconnects, so access was switched to `L.Livingstone` using **pass-the-hash** with `evil-winrm`.

**WinRM** is a Windows service for remote PowerShell management over HTTP(S) (ports 5985/5986). **Evil-WinRM** is an offensive tool that abuses WinRM to obtain an interactive PowerShell shell, including support for pass-the-hash and plaintext credentials.

![evilwinrm](/assets/img/blog/PGLabs/Resourced/evilwinrm.png)
![evilwinrm_priv](/assets/img/blog/PGLabs/Resourced/evilwinrm_priv.png)

`l.livingstone` is not a local or domain administrator, but possessed the `SeMachineAccountPrivilege`, which allows the creation of computer accounts in the domain.

Privilege Escalation looks like this below:

**1)** use **L.Livingstone (non-administrative account)** to create **fake computer account**<br>
**2)** give fake computer account **RBCD (Resource-Based Constrained Delegation) rights**<br>
**3)** forge **Kerberos ticket**<br>
**4)** log in as **SYSTEM** on server<br>
**5)** pivot to **Domain Admin**<br>

This escalation path was selected based on delegated privileges observed during Active Directory enumeration, rather than brute-force or vulnerability-based exploitation.

## Active Directory Enumeration
**[SharpHound][SharpHound]** was executed on the compromised host. SharpHound collects Active Directory relationship data for BloodHound to analyze privilege escalation paths.

**Note for macOS users:** As of ~2 weeks ago, SharpHound releases no longer include a precompiled `SharpHound.exe` and now ship source code only (`.sln`). macOS cannot practically build SharpHound into a usable Windows executable, so **SharpHound must be compiled or executed from a Windows environment (e.g., Parallels Desktop, VMware, or a native Windows host)**. Kali and macOS do not provide SharpHound binaries by default.

![sharphound](/assets/img/blog/PGLabs/Resourced/sharphound.png)

The resulting ZIP was transferred and ingested into **BloodHound** for local analysis. 

![bloodhound](/assets/img/blog/PGLabs/Resourced/bloodhound.png)

The `GenericAll` edge between `L.Livingstone` and `ResourceDC` means full control over the target object, allowing delegation-related attributes to be modified and enabling RBCD abuse.

![GenericAll](/assets/img/blog/PGLabs/Resourced/GenericAll.png)

**PowerView** (specialized PowerShell reconnaissance toolkit) was used to inspect **ACLs** on the target object, confirming inherited **ACEs** that granted **excessive control**. The final ACE was inherited (`IsInherited`, `ContainerInherit`) and mapped to `SID 519` (**Enterprise Admins**). `IsInherited` means the permission comes from a parent object, while `ContainerInherit` means that permission propagates to child objects. 

![powerviewps1](/assets/img/blog/PGLabs/Resourced/powerviewps1.png)
![ACEs](/assets/img/blog/PGLabs/Resourced/ACEs.png)

## Privilege Escalation

The Impacket tools used here are *self-explanatory* by name. A new computer account was added, and a Kerberos service ticket was requested for impersonation.

![impacket-addcomputer](/assets/img/blog/PGLabs/Resourced/impacket-addcomputer.png)
![rbcd](/assets/img/blog/PGLabs/Resourced/rbcdattack.png)
![impacket-getST](/assets/img/blog/PGLabs/Resourced/impacket-getST.png)
![etchost](/assets/img/blog/PGLabs/Resourced/etchost.png)
![impacket-psexec](/assets/img/blog/PGLabs/Resourced/impacket-psexec.png)

This resulted in execution with SYSTEM-level privileges on a domain controller, completing domain compromise.

## Remedies

1. **Never store credentials in SMB shares**
    1. use password manager
    2. secure onboard workflow
    3. just-in-time credential delivery

2. **Apply Princple of Least Privilege to SMB shares**
    1. never `Everyone`
    2. no authenticated user unless required
    3. separate read and write access
    4. read-only access is still exposed

3. **No backup or sensitive artifacts in SMB shares**
    1. audits shouldn't be stored there
    2. backup location should be restricted to Backup Operators only

4. **Remove SeMachineAccountPrivilege from standard users**
    1. remove `SeMachineAccountPrivilege` from standard users
    2. remove it from service accounts
    3. restrict it to Domain Admins only

5. **Audit and restrict RBCD permissions**
    1. regularly audit `msDS-AllowedToActOnBehalfOfOtherIdentity`
    2. prevent non-admin users from modifying delegation setting

6. **Limit computer accounts to prevent new created accounts**
    1. set `ms-DS-MachineAccountQuota` to `0`
    2. delegate computer creation to dedicated provisioning accounts

7. **Disable legacy NTLM authentication**
    1. reduce or disable NTLM usage
    2. enforce Kerberos hardening
    3. monitor pass-the-hash activity

[Dpsypher]:https://medium.com/@Dpsypher/proving-grounds-practice-resourced-b3a50d40664b
[SharpHound]:https://github.com/SpecterOps/SharpHound/releases
