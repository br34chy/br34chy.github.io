---
layout: post
title: "Proving Grounds PlanetExpress: Silent S"
date: 2026-02-09 00:00:00 -0400
categories: [Labs, Proving Grounds]
tags: [Active Directory]
years: ['2026']
comments: true
---

**Trusted relay.** <br>
**Privilege carried.**

**Lesson: trusted services can quietly carry privilege where it doesn’t belong.**

![PlanetExpress](/assets/img/blog/PGLabs/PlanetExpress/PlanetExpress.png)

## SUID Misconfiguration

![labdescr](/assets/img/blog/PGLabs/PlanetExpress/labdescr.png)

**PlanetExpress (PG Lab)** is an example of **SUID (Set User ID/setuid) misconfiguration**. SUID abuse occurs when a service runs with **root level privilege**. In file permissions, SUID appears as an `s`, for example: `-rwsr--r--`. Attacker can use that service to **cross a privilege boundary**.  

This lab is offically rated **Easy**, yet the community rates it **Very Hard**. *Which is it?* 

The difficulty does not come from exploitation, it comes from **discovery**. The hardest part is **directory enumerating**. Relying on tools like `john` or `ffuf` cost time, searching through wordlists before an entry point is found. Using the right tool, such as `dirsearch` makes this easy. 

## Initial Enumeration

Only three ports were open: `22` **ssh**, `80` **http**, and `9000` **cslistener**, an unfamiliar service listening on the host.

![nmap](/assets/img/blog/PGLabs/PlanetExpress/nmap.png)

SSH was running **OpenSSH 7.9p1**, which has a few reported CVEs, but requires credentials to exploit. HTTP hosted **Pico CMS**, which warranted further inspection later.  Googling "port 9000 cslistener" identified it as a **PHP-FPM (FastCGI Process Manager)**. When PHP is in play, it could be exploitable. The host was running **Debian Linux**.

![port9000](/assets/img/blog/PGLabs/PlanetExpress/port9000.png)

The webpage itself exposed nothing and appeared solid.

![webpage](/assets/img/blog/PGLabs/PlanetExpress/webpage.png)

`ffuf` yielded almost nothing, so I switched to `dirsearch` to speed up enumeration. Populated several subdirectories: `/assets`, `/config`, `/content`, `/plugins`, `/server-status`, `/themes`, and `/vendor`.

![dirsearch1](/assets/img/blog/PGLabs/PlanetExpress/dirsearch1.png)
![dirsearch2](/assets/img/blog/PGLabs/PlanetExpress/dirsearch2.png)
![dirsearch3](/assets/img/blog/PGLabs/PlanetExpress/dirsearch3.png)

Most `.gitignore` files indicated intentionally empty directories. `/plugins/.gitignore` file stated that plugins are stored in this directory and listed **/PicoDeprecated**. Accessing `../plugins/PicoDeprecated` returned a Forbidden page, so I decided to investigate further. This revealed that **Pico CMS is entirely PHP based**.  

![PicoDeprecated](/assets/img/blog/PGLabs/PlanetExpress/PicoDeprecated.png)
![PicoCMS](/assets/img/blog/PGLabs/PlanetExpress/PicoCMS.png)

One subdirectory contained a file that stood out from the `.gitignores`. Appendeding `/config/config.yml` rendered the file directly in the browser or opened in a text editor.

![configyml](/assets/img/blog/PGLabs/PlanetExpress/configyml.png)

At bottom of the `config.yml` file, a plugin named `PicoTest` was shown "enabled". Accessing `/plugins/PicoTest/` failed, so I tried `/plugins/PicoTest.php`. 

![pluginphp](/assets/img/blog/PGLabs/PlanetExpress/pluginphp.png)

## Initial Access

In `PicoTest.php`, what caught my eyes was **Server API: FPM/FastCGI**. This brought back **port 9000 cslistener**. FPM/FastCGI might be already exposed, so I searched for it.

![google1](/assets/img/blog/PGLabs/PlanetExpress/google1.png)
![google3](/assets/img/blog/PGLabs/PlanetExpress/google3.png)

[HackTricks: FPM.py](https://book.hacktricks.wiki/en/network-services-pentesting/9000-pentesting-fastcgi.html)

I downloaded the ZIP from GitHub and extracted `FPM.py` on my local Kali.

![google4](/assets/img/blog/PGLabs/PlanetExpress/google4.png)

The -h flag was used to review FPM.py usage.

![fpmpyh](/assets/img/blog/PGLabs/PlanetExpress/fpmpyh.png)

I am not familiar with PHP shell functions, so I googled once again.

![google5](/assets/img/blog/PGLabs/PlanetExpress/google5.png)

PHP documentation lists several functions. `shell_exec` looked promising, and other options are shown on the right side column. 

![google6](/assets/img/blog/PGLabs/PlanetExpress/google6.png)
![shell_exec](/assets/img/blog/PGLabs/PlanetExpress/shell_exec.png)

PHP message output warned that `shell_exec` has been disabled. `PicoTest.php` has a "disable_functions" list. Checked whether `exec` was also disabled. 

![disabled](/assets/img/blog/PGLabs/PlanetExpress/disabled.png)
![exec](/assets/img/blog/PGLabs/PlanetExpress/exec.png)

`exec` was indeed in the disable_functions group. Went through other remaining options, and `passthru` was available. The example syntax initially looked complicated, but the comments clarified that it could be used similarly to shell_exec.

![passthru](/assets/img/blog/PGLabs/PlanetExpress/passthru.png)
![passthru2](/assets/img/blog/PGLabs/PlanetExpress/passthru2.png)
![passthru3](/assets/img/blog/PGLabs/PlanetExpress/passthru3.png)

The `passthru` successfully displayed raw output. `id` confirmed `www-data`. Not `root`, but more than enough to work with.

![uid](/assets/img/blog/PGLabs/PlanetExpress/uid.png)

A **Netcat reverse shell payload** was inserted, and the listener spawned a **raw Bash shell without a TTY**.

![ncbash](/assets/img/blog/PGLabs/PlanetExpress/ncbash.png)

## Privilege Escalation

`/etc/shadow` was unreadable, **SUID binaries** became the next focus. `relayd` was the only binary I didn’t recognize.

![findperm](/assets/img/blog/PGLabs/PlanetExpress/findperm.png)
![relayd](/assets/img/blog/PGLabs/PlanetExpress/relayd.png)

Relayd is a traffic relay, proxy, and a load-balancer daemon that manages connections, not systems. It touches networking, interacts with firewalls, manages low ports, and parses user configuration. The `ls -la` output confirmed the SUID bit, visible as an `s` in the owner’s execute position.

![suid](/assets/img/blog/PGLabs/PlanetExpress/suid.png)

The `-C` option reads configuration from a file, as noted in the `relayd` help output.

![relaydc](/assets/img/blog/PGLabs/PlanetExpress/relaydc.png)

Extracted the hash and cracked it. 

![hashwiki](/assets/img/blog/PGLabs/PlanetExpress/hashwiki.png)
![hashcat](/assets/img/blog/PGLabs/PlanetExpress/hashcat.png)
![cracked](/assets/img/blog/PGLabs/PlanetExpress/cracked.png)

Password accepted. Root. 

![suroot](/assets/img/blog/PGLabs/PlanetExpress/suroot.png)
![prooftxt](/assets/img/blog/PGLabs/PlanetExpress/prooftxt.png)

## Remedies

1. **Restrict exposure of PHP-FPM**
    1. bind PHP-FPM to a Unix socket instead of a public TCP port
    2. block external access to `port 9000` via firewall rules
    3. only web server can communicate with the FPM process internally

2. **Disable or strictly limit PHP command execution**
    1. remove unnecessary functions such as `passthru`, `exec`, and `shell_exec`
    2. enforce hardened disable_functions policies in `php.ini`
    3. enable only when strictly required and then disable immediately

3. **Audit and minimize SUID privileges**
    1. regularly review SUID-enabled binaries using `find / -perm -4000`
    2. remove SUID from binaries that do not strictly require it
    3. prevent privileged services from loading arbitrary user-supplied configuration files