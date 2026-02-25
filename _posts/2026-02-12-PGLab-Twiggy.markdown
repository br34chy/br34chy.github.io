---
layout: post
title: "Proving Grounds Twiggy: Backport"
date: 2026-02-13 00:00:00 -0400
categories: [Labs, Proving Grounds]
tags: [Active Directory]
years: ['2026']
comments: true
---

**Imagined a forest.** <br>
**It was a twig.**

**Lesson: Don't overthink. Think in primitives.**

![Twiggy](/assets/img/blog/PGLabs/Twiggy/Twiggy.png)
![1labdescrb](/assets/img/blog/PGLabs/Twiggy/1labdescrb.png)

**Twiggy (PG Lab)** revolves around a backport regression. A fix from a newer version was applied to an older codebase, but architectural differences introduced unintended exposure. When code is transplanted without full context, assumptions break.

The machine is rated **easy** and **community-rated intermediate**. It should be easy. I made it harder than it needed to be.

## Initial Enumeration

Sometimes `-p-` is necessary depending on scope. The first nmap scan did not reveal ports `4505` and `4506`. A full port scan did.

![2nmap](/assets/img/blog/PGLabs/Twiggy/2nmap.png)

Ports `80` and `8000` were also open. The website had a lot of links to click on and offered nothing immediately useful.

![webpage](/assets/img/blog/PGLabs/Twiggy/webpage.png)

Ports `4505` and `4506` were open. I was not familiar with them. **Googled "port 4505 4506 ZeroMQ ZMTP 2.0"**. Led to this [stackoverflow link][stackoverflow link], hinting at **SaltStack**.

![3goole1](/assets/img/blog/PGLabs/Twiggy/3google1.png)

A quick `searchsploit saltstack` confirmed public exploits.

![4searchsploit](/assets/img/blog/PGLabs/Twiggy/4searchsploit.png)

Inside the exploit script were two commented GitHub references. The first [github link][github link] pointed to SaltStack checker scripts introduced as part of a security backport.

![checker](/assets/img/blog/PGLabs/Twiggy/checker.png)

Exploit usage options were listed.

![5source](/assets/img/blog/PGLabs/Twiggy/5source.png)

Removed comments in the top of exploit script to make it executable. 

![6saltypy](/assets/img/blog/PGLabs/Twiggy/6saltypy.png)

There was a requirement, `pip3 install salt`.

![7saltreq](/assets/img/blog/PGLabs/Twiggy/7saltreq.png)

A missing module error appeared.

![8pyyml](/assets/img/blog/PGLabs/Twiggy/8pyyml.png)

More `ModuleNotFoundErrors` followed, I installed each dependency as it appeared. 

```bash
pip install pyyaml
pip install looseversion
pip install packaging
pip install tornado
```

A different error message surfaced, `NameError name 'msgpack' is not defined`. Same solution, install the missing module. Additional `ModuleNotFoundErrors` messages required packages. 

```bash
pip install distro
pip install jinja2
pip install zmq
```
Finally the exploit script executed successfully! The root key was exposed, but not leverageable yet. 

![9installs](/assets/img/blog/PGLabs/Twiggy/9installs.png)

This is where I overthought the problem. The exploit usage referenced uploading a evilcrontab, which I interpreted as a direct hint to pivot through cron. Instead of pausing to think, I jumped straight into it.

![11evilcrontab](/assets/img/blog/PGLabs/Twiggy/11evilcrontab.png)

I reviewed `/etc/crontab` to understand the format and confirmed I could overwrite it.

![14verifycrontab](/assets/img/blog/PGLabs/Twiggy/14verifycrontab.png)

I attempted to trigger a reverse shell through crontab with an active listener. Nothing happened. I double-checked the syntax to ensure the command was correct.

![13revshelcron](/assets/img/blog/PGLabs/Twiggy/13revshelcron.png)

Every attempts returned `Successfully scheduled job: .....`. 

![10schjob](/assets/img/blog/PGLabs/Twiggy/10schjob.png)

At that point, I stepped back. Using `-r`, I confirmed I could read `/etc/shadow`. That should have been the pivot. Instead of chasing persistence or reverse shells, I already had root-level file access. 

![mkpasswd](/assets/img/blog/PGLabs/Twiggy/mkpasswd.png)

The solution was simple: generate a SHA-512 hash with `mkpasswd`, append a new UID `0` user, and overwrite `/etc/passwd` and upload it. That was it.

![upload](/assets/img/blog/PGLabs/Twiggy/upload.png)
![proof](/assets/img/blog/PGLabs/Twiggy/proof.png)



## Remedies

1. **Harden backport and patch validation**
    1. perform security regression testing before releasing backported fixes
    2. review authentication and authorization flows after modifying privileged components
    3. validate that new security features do not expose internal functions externally
2. **Restrict exposure of management services**
    1. bind administrative services to localhost or private interfaces only
    2. restrict access to management ports (e.g., `4505/4506`) using firewall rules
    3. enforce network segmentation between management services and untrusted networks
3. **Enforce strict authentication on control APIs**
    1. require authentication and authorization checks before executing privileged functions
    2. avoid exposing diagnostic or checker endpoints without access control
    3. implement monitoring and logging for all administrative API interactions

[stackoverflow link]:https://stackoverflow.com/questions/30671469/what-open-ports-are-required-on-firewall-to-allow-for-salt-stack-remote-executio
[github link]:https://github.com/rossengeorgiev/salt-security-backports
[exploit script]: https://github.com/jasperla/CVE-2020-11651-poc