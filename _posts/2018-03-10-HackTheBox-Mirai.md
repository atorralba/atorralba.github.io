---
layout: post
title: Hack The Box - Mirai
---

I've been *very* busy with my PWK course for OSCP lately, and that's why I've not been posting much here. But recently I received the notification that **Mirai**, a box from [Hack The Box](https://www.hackthebox.eu/) (a site you should *really* check out if you haven't yet), had been retired. Since I solved it back in the day, and luckily I had some notes about how I did it, I thought of writing a little walkthrough and post it here.

And yeah, that's what you're reading right now. Crazy, huh?

## Enumeration

As always, let's launch `nmap` and see what we get:

```
# nmap -A -p- 10.10.10.48

Starting Nmap 7.40 ( https://nmap.org ) at 2017-10-31 22:52 CET
Stats: 0:00:50 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 16.67% done; ETC: 22:53 (0:00:30 remaining)
Nmap scan report for 10.10.10.48
Host is up (0.045s latency).
Not shown: 65529 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u3 (protocol 2.0)
| ssh-hostkey:
|   1024 aa:ef:5c:e0:8e:86:97:82:47:ff:4a:e5:40:18:90:c5 (DSA)
|   2048 e8:c1:9d:c5:43:ab:fe:61:23:3b:d7:e4:af:9b:74:18 (RSA)
|_  256 b6:a0:78:38:d0:c8:10:94:8b:44:b2:ea:a0:17:42:2b (ECDSA)
53/tcp    open  domain  dnsmasq 2.76
| dns-nsid:
|_  bind.version: dnsmasq-2.76
80/tcp    open  http    lighttpd 1.4.35
|_http-server-header: lighttpd/1.4.35
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
1324/tcp  open  upnp    Platinum UPnP 1.0.5.13 (UPnP/1.0 DLNADOC/1.50)
32400/tcp open  http    Plex Media Server httpd
| http-auth:
| HTTP/1.1 401 Unauthorized\x0D
|_  Server returned status 401 but no WWW-Authenticate header.
|_http-cors: HEAD GET POST PUT DELETE OPTIONS
|_http-title: Unauthorized
32469/tcp open  u	pnp    Platinum UPnP 1.0.5.13 (UPnP/1.0 DLNADOC/1.50)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.40%E=4%D=10/31%OT=22%CT=1%CU=32400%PV=Y%DS=2%DC=T%G=Y%TM=59F8F0
OS:EE%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=10B%TI=Z%CI=I%II=I%TS=8)OP
OS:S(O1=M54DST11NW6%O2=M54DST11NW6%O3=M54DNNT11NW6%O4=M54DST11NW6%O5=M54DST
OS:11NW6%O6=M54DST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=7120%W6=7120)EC
OS:N(R=Y%DF=Y%T=40%W=7210%O=M54DNNSNW6%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=
OS:AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(
OS:R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%
OS:F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N
OS:%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%C
OS:D=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 199/tcp)
HOP RTT      ADDRESS
1   45.12 ms 10.10.14.1
2   45.23 ms 10.10.10.48

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 75.55 seconds
```

Okay, `ssh`, `dns` and a webserver. `UPnP`, uhm, *another* webserver and... huh, more `UPnP`. Okay, I guess.

The banners give us some hints already. For example, the webserver at `32400` says `Plex Media Server httpd`, and it returns a `401 Unauthorized` error. So let's try port `80` and see if we have more luck.

## Webserver

Once we open `http://10.10.10.48/` in a browser, we get a fabulous blank page and nothing more. Great. I tried some basic stuff manually, like `robots.txt`, then more crazy things that came to mind and I prefer not to disclose to not sound like a *maniac*, but at some point I decided to launch `dirb` and hope for the best.

```
# dirb http://10.10.10.48 /usr/share/dirb/wordlists/big.txt

-----------------
DIRB v2.22
By The Dark Raver
-----------------

START_TIME: Wed Nov  1 00:11:18 2017
URL_BASE: http://10.10.10.48/
WORDLIST_FILES: /usr/share/dirb/wordlists/big.txt

-----------------

GENERATED WORDS: 20458

---- Scanning URL: http://10.10.10.48/ ----
==> DIRECTORY: http://10.10.10.48/admin/
```

(**Note:** As you can see, I used the `big.txt` wordlist here right away. The reason is I actually had spent a *pretty shameful* amount of time trying all sort of crazy things in all the exposed ports, so I was a *little* frustrated. I hadn't started PWK yet, so I wasn't used to this kind of frustration :)

Anyway, our guy `dirb` actually found something at `/admin`, and to there I browsed. The login page of a [`Pi-hole`](https://pi-hole.net/) appeared before my eyes as I looked at it blankly and confused.

But then it hit me. You usually install a media server and a `Pi-hole` in one, and only one, kind of device. And there was the name of the box, also.

Was I attacking a Raspberry Pi? (*dramatic music*)

## SSH

I actually own two Pi's (which means I've configured, like, 8) and I know pretty well the default SSH credentials for this little, cute boxes: `pi:raspberry`. Worth a shot, right?

```
# ssh pi@10.10.10.48

pi@raspberry:~ $
```

Aaaaand we are in. `user.txt` flag acquired. Great!

Another useful piece of information that you obtain when you play with these things (or, you know, use Google) is that `pi` is in the `sudoers` group by default, and often even without the need of using a password. So you can imagine how delighted I was when I typed `sudo su` and reached `root` like *real hackers* do. I didn't have sunglasses at hand to put them on and whisper *I'm in*, but you get the point.

## Root

So it was as easy as `cd`ing to `/root` and getting the `root.txt` flag, right? Yeah, well, no.

```
# cd

# cat root.txt
I lost my original root.txt! I think I may have a backup on my USB stick...
```

Damn. I was losing *leetness* by moments here. Let's see what's in the USB stick then...

```
# cd /media/usbstick

# ls
lost+found
damnit.txt

# cat damnit.txt
Damnit! Sorry man I accidentally deleted your files off the USB stick.
Do you know if there is any way to get them back?
-James
```

James, seriously, what's wrong with you.

So this sounded like *forensics*, which I had nearly 0 experience on, but well, it was a day good as any other to learn. After Googling a little, I decided it would be easier to create an image of the USB stick and take it to my Kali box, where some pre-installed forensics tools could be of help.

```
# dd if=/dev/sdb bs=1M > /home/pi/disk.img

# cd /home/pi/

# python -m SimpleHTTPServer
```

And then from my Kali:

```
# wget 10.10.10.48:8000/disk.img
```

Okay, time to put my recently acquired forensics *knowledge* into practice! I launched `testdisk`, a tool that promised to be able to recover lost files. And even though it actually found a deleted file (very conveniently named `root.txt`) inside the image, it wasn't able to recover its contents.

I then tried with `photorec`, another tool recommended by my friend "The Intrernet", but the result was similarly disappointing.

I was about to surrender and leave the city and spend the rest of my life farming and living in harmony with Mother Nature, but then I thought of something simpler I hadn't tried yet:

```
# strings disk.img
>r &
/media/usbstick
lost+found
root.txt
damnit.txt
>r &
>r &
/media/usbstick
lost+found
root.txt
damnit.txt
>r &
/media/usbstick
2]8^
lost+found
root.txt
damnit.txt
>r &
--ROOT FLAG MD5 HERE--
Damnit! Sorry man I accidentally deleted your files off the USB stick.
Do you know if there is any way to get them back?
-James
```

Of course, `--ROOT FLAG MD5 HERE--` was actually the real flag, which meant **Challenge Complete!** It hurt a little to not have thought of this earlier, but the sound of *pure victory* in my ears silenced that small detail.

## Conclusion

This was my first box at Hack The Box and had real fun with it, though it depended a little on the *Eureka!* moment. I'm actually glad I didn't think of the simple `strings` solution to the "forensics" part right away, since that made me learn a little about dedicated tools used for recovering lost files (even though they didn't work for this particular problem).

Thanks to the author for creating Mirai, and to Hack The Box for hosting it. Sure enough it won't be my last machine in the site! (But first, I'll get my OSCP...)
