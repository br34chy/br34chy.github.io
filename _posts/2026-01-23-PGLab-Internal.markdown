---
layout: post
title: "PG Lab - Internal"
date: 2026-01-23 00:00:00 -0400
categories: [Labs, Proving Grounds]
tags: [Active Directory]
years: ['2026']
comments: true
---

## Exploited Without Metasploit, Yet Befuddled

![labdescr](/assets/img/blog/PGLabs/Internal/labdescr.png)

**Internal (PG Lab)** machine was compromised **without Metasploit** on a MacBook Pro. Replicating the same process on a PC failed. Environmental might be the issue rather than exploit-related. This write-up focuses on *why* that discrepancy occurred and concludes with a remedy.

I'm no expert, PG lab walkthroughs on Medium are usually reviewed before diving in. With this machine, almost everyone resorted to `msfconsole`. Except for **[kh@n][kh@n]**. The *result* was there, the *how* wasn't.  

![kh@n](/assets/img/blog/PGLabs/Internal/kh@n.png)

His walkthrough noted that the shellcode length exactly matched the original through NOP padding (`\x90`), which was replicated.

![comment](/assets/img/blog/PGLabs/Internal/nopad.png)

Despite matching the shellcode and reusing his `rpcclient` NULL session command, access was not obtained. A reread of the post revealed no obvious reason the exploit should fail.

A review of **[Dpsypher][Dpsypher]**’s walkthrough revealed a comment by Joaquin Argas, who did not publish a full write-up.

![comment](/assets/img/blog/PGLabs/Internal/comment.png)

The only hint in the comment referenced `python2`. While python2 itself could not be installed, `python2.7` was available and installed system-wide, along with `get-pip.py` and `pysmb`. This approach did not succeed. After reverting to `Python3` and retrying, the result was an **NT_STATUS_NOT_FOUND** error, even though the listener reported a connection. I dug a bit more and came across a relevant comment on the [HTB forum][HTB forum] by Rachel Gomez.

![NTSNF](/assets/img/blog/PGLabs/Internal/NTSNF.png)

This pointed away from network reachability and toward how the **client interacted with the server**. On Debian-based systems, this raised suspicion of missing client-side SMB tooling. The package most likely responsible appeared to be `samba-common-bin`. Additional packages such as `smbclient`, `impacket-scripts`, and `python3-impacket` were installed to eliminate other variables.

![dependencies](/assets/img/blog/PGLabs/Internal/dependencies.png)

I went for one more attempt and was on the brink of giving in to Metasploit. Initially, the attempt appeared unsuccessful.

![failornot](/assets/img/blog/PGLabs/Internal/failornot.png)

The listener connected and access was obtained! 

![connected1](/assets/img/blog/PGLabs/Internal/connected1.png)

The flag was located in `Administrator/Desktop`, as expected.

![flag](/assets/img/blog/PGLabs/Internal/flag.png)

To validate the behavior, the process was repeated on a PC with packages installed incrementally. Installing samba-common-bin alone was insufficient. Installing smbclient also failed to resolve the issue. Even after installing the Impacket packages, the outcome remained inconsistent. **I’m befuddled**.

## Remedies (Legacy environments)

Although the issue has since been patched, environments that must retain legacy SMB or RPC behavior should focus on reducing blast radius rather than eliminating compatibility outright.

1. **Restrict anonymous and NULL SMB access**
    1. disable anonymous SMB sessions
    2. disable NULL RPC bindings
    3. require authentication for SMB named pipes

2. **Reduce SMB attack surface**
    1. limit SMB access to required hosts only
    2. restrict SMB access to internal management networks

3. **Enforce consistent SMB client behavior**
    1. require SMB signing
    2. disable fallback authentication methods

4. **Monitor suspicious SMB behavior**
    1. alert on SMB connections followed by failed RPC calls
    2. monitor repeated NT_STATUS errors


[kh@n]:https://medium.com/@ayxantanirverdiyev6/pg-practice-internal-walkthrough-49bca402f737
[Dpsypher]:https://medium.com/@Dpsypher/proving-grounds-practice-internal-e5098dd29793
[HTB forum]:https://forum.hackthebox.com/t/error-nt-status-not-found-smbclient-pwnbox/266639/4
[`samba-common-bin`]:https://manpages.debian.org/experimental/samba-common-bin/samba.7.en.html