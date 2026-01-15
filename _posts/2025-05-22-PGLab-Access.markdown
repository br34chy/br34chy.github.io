---
layout: post
title: "PG Lab - Resourced"
date: 2025-12-08 00:00:00 -0400
categories: [Labs, Proving Grounds]
tags: [Active Directory]
years: ['2025']
comments: true
---

## Delegation Misconfig and Remedy

![test](/assets/img/blog/PGLabs/Resourced/test.png)

![resourced_descr](assets/img/blog/PGlabs/Resourced/resourced_descr.png)

**Resourced (PG Lab)** demonstrates a classic **Active Directory delegation abuse** path that leads from a low-privileged domain user to **Domain Admin** via **Resource-Based Constrained Delegation (RBCD)**. This write-up focuses on the attack chain, why it works, and how to remediate after impact is proven. This lab follows a known RBCD abuse path; **[Dpsypher][Dpsypher]'s public write-up** was used as a reference during analysis.

## Initial Enumeration

Service enumeration revealed an **Active Directory** environment, with standard domain services exposed (**DNS, Kerberos, LDAP, SMB**).

![nmap](./assets/img/blog/PGlabs/Resourced/nmap.png)

`Enum4linux` discovered a **low-privileged domain user** with a default password, providing an **initial foothold**.

![enum4linux](./assets/img/blog/PGlabs/Resourced/enum4linux.png)

## SMB Enumeration

Tested **anonymous access**, followed by authentication using the discovered user with a default password.

![smbclient](/assets/img/blog/PGlabs/Resourced/smbclient.png)
![smbrecursels](/assets/img/blog/PGlabs/Resourced/smbrecursels.png)

The `ntds.dit` file was exposed via a **misconfigured SMB share**, allowing it to be retrieved without prior **domain administrator privileges**.

![smbget](/assets/img/blog/PGlabs/Resourced/smbget.png)


## Credential Access

Extracted `SAM` and `SYSTEM` hives to recover **local account hashes**; `SECURITY` was not required.

![impacket-secretsdump](/assets/img/blog/PGlabs/Resourced/impacket-secretsdump.png)
![hashes](/assets/img/blog/PGlabs/Resourced/hashes.png)
![crackstation](/assets/img/blog/PGlabs/Resourced/crackstation.png)

Identified a user account with **valid hash-based authentication**.

![crackmapexe1](/assets/img/blog/PGlabs/Resourced/crackmapexec_command.png)
![crackmapexe2](/assets/img/blog/PGlabs/Resourced/crackmapexec_result.png)

Although both local and domain-related artifacts were collected, only the recovered hashes that authenticated successfully against **domain resources** were used for further access.

## Initial Access

`ItachiUchiha` with cracked password disconnected repeatedly; switched to `L.Livingstone` using **pass-the-hash**.

![evilwinrm](/assets/img/blog/PGlabs/Resourced/evilwinrm.png)
![evilwinrm_priv](/assets/img/blog/PGlabs/Resourced/evilwinrm_priv.png)

`l.livingstone` is not a local or domain administrator but possessed the `SeMachineAccountPrivilege`, which allows the creation of computer accounts in the domain.

Privilege Escalation looks like this below:

**1)** using **L.Livingstone (non-administrative account)** to create **fake computer account**<br>
**2)** give fake computer account **RBCD (Resource-Based Constrained Delegation) rights**<br>
**3)** forge **Kerberos ticket**<br>
**4)** log in as **SYSTEM** on server<br>
**5)** pivot to **Domain Admin**<br>

This escalation path was selected based on delegated privileges observed during Active Directory enumeration, rather than brute-force or vulnerability-based exploitation.

## Active Directory Enumeration

**SharpHound** was executed on the compromised host, and the resulting ZIP was transferred and ingested into **BloodHound** for local analysis.

![sharphound](/assets/img/blog/PGlabs/Resourced/sharphound.png)
![bloodhound](/assets/img/blog/PGlabs/Resourced/bloodhound.png)
![GenericAll](/assets/img/blog/PGlabs/Resourced/GenericAll.png)

**PowerView** was used to inspect **ACLs** on the target object, confirming inherited **ACEs** that granted **excessive control**. The final ACE was inherited (`IsInherited`, `ContainerInherit`) and mapped to `SID 519` (**Enterprise Admins**).

![powerviewps1](/assets/img/blog/PGlabs/Resourced/powerviewps1.png)
![ACEs](/assets/img/blog/PGlabs/Resourced/ACEs.png)

## Privilege Escalation

![impacket-addcomputer](/assets/img/blog/PGlabs/Resourced/impacket-addcomputer.png)
![rbcd-attack](/assets/img/blog/PGlabs/Resourced/rbcd-attack.png)
![impacket-getST](/assets/img/blog/PGlabs/Resourced/impacket-getST.png)
![etchost](/assets/img/blog/PGlabs/Resourced/etchost.png)
![impacket-psexec](/assets/img/blog/PGlabs/Resourced/impacket-psexec.png)

This resulted in execution with SYSTEM-level privileges on a domain controller, completing domain compromise.

## Remedies

1. **Never store credentials in SMB shares**
    1. use password manager
    2. secure onboard workflow
    3. just-in-time credential delivery

2. **Apply Princple of Least Privilege to SMB shares**
    1. Never `Everyone`
    2. no authenticated user unless required
    3. seperate read and write access
    3. read-only access is still exposed

3) **No backup or sensitive artifacts in SMB shares**
    1. audits shouldn't be stored there
    2. backup location should be restricted to Backup Operators only

4) **Remove SeMachineAccountPrivilege from standard users**
    1. remove `SeMachineAccountPrivilege` from standard users
    2. remove it from service accounts
    3. restrict it to Domain Admins only

5) **Audit and restrict RBCD permissions**
    1. regularly audit `msDS-AllowedToActOnBehalfOfOtherIdentity`
    2. prevent non-admin users from modifying delegation setting

6) **Limit computer accounts to prevent new created account**
    1. set `ms-DS-MachineAccountQuota` to `0`
    2. Delegate computer creation to dedicated provisioning accounts

7) **Disable legacy NTLM authentication**
    1. reduce or disable NTLM usage
    2. enforce Kerberos hardening
    3. monitor pass-the-hash activity

[Dpsypher]:https://medium.com/@Dpsypher/proving-grounds-practice-resourced-b3a50d40664b

