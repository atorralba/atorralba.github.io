---
layout: post
title: OSCP Review - Or How I Tried to Try Harder
---

Last week, I very gladly received an e-mail from Offensive Security: I had passed the Pentesting With Kali Linux (PWK) course and exam, and therefore I had obtained my OSCP certification. Given that I had almost fully committed my free time of the last few months to this course, you can imagine how happy I was to receive such message. Now that I've recovered a little (mentally and physically), it seems like a good idea to write some kind of wrap-up of the whole process, beginning to end. I am aware this has been done like a *trillion* times by other people; but, you know, it may still be helpful or at least entertaining, so I'm doing it anyway.

## Background

I've seen some people very interested in this part when asking others about OSCP. What's the necessary background to be able to enjoy and pass the course? I can only talk about my experience, but when I started PWK, I had already done the following:

+ The *uni* thing (Bachelor's in Computer Science, Master's in Information Security)
+ Solved many CTF-like boxes, mainly from [VulnHub][1] and later [Hack The Box][2]. *Full-Disclosure*: At first, I couldn't solve a single one and had to read/watch lots of write-ups (I would specially recommend [g0blin][3]'s and [ippsec][4]'s). Then, I started grasping something similar to a methodology, and began to actually solve some of the easy boxes. Writing about it in this blog also helped with taking notes and giving structure to the process.
+ Solved the first challenges of [exploit-exercises][5] (Nebula and Protostar).
+ Started learning reversing with [radare2][6] and solved a beginner-level crackme.
+ Used Linux as main OS for some years and had some bash trickery up my sleeve.
+ Had 3+ years of experience as software engineer and near 6 months as secure code reviewer, with the occasional pentest as a treat. That gave me some understanding of networking and the most common protocols, certificates and PKI, some basic tooling and the typical security vulnerabilities (SQLi, XSS, Command Injection, Path Traversal...)

Of course, I'm not saying that everyone needs anything of that list before starting the course; I'm just saying where I was when I started it. In the end, it's an entirely personal matter, but I would say you at least need some familiarity with Linux command-line, basic security and networking knowledge, good googling skills and plenty of free time and/or willing to spend it in the course. The rest is up to your stubbornness and ability to not give up to frustration. Personally speaking, I felt I was ready when I stopped cheating (i.e. reading the write-up) while head-butting against CTF challenges and actually solved them on my own.

## The course

Once you start PWK, you are given the course materials and the lab VPN access. Your lab time starts running as soon as you receive the "Starting Pack", so you may feel the rush to jump into the labs right away (it happened to me). Actually, depending on how much lab time you got, you probably will need some "vacation" days and won't be touching the lab for some time anyway. So don't feel bad if you need a couple weeks (or more!) to read the materials, watch the videos and solve the exercises, specially if you don't have solid bases. You are provided excellent learning material, so you should take advantage of that! On the other hand, you are completely free to go back and forth between the lab and the exercises if you feel like it; don't think that once you dive into the labs you are in a one-way journey you will only come back from when the lab time ends. Personally speaking, I ended up using the PDFs and exercises more as reference material while solving the labs that as a traditional course; but that's up to you.

Once said that, I will be honest: the real fun starts with the labs. When you connect to the VPN, you are given access to the THINC internal network, where lots of boxes await for you to pop a shell on them. Some are easier, some are harder, and you can define your own goals. You may want to compromise as much machines as possible. Others may want to take down the almost-mythical "Top 3" machines, which are arguably . You can also aim to go deeper into the network, reaching the IT, Development and finally Administration subnets. It all depends on your interests, available time and personal tastes.

In my case, I decided to go sequentially, starting with the first IP in range and ending with the last. As personal goals, I decided to try to pop at least one of the "Big Three", and then go for the Administration network (which is described as the "ultimate goal" in the course description). This probably isn't the best method to optimize your time in the lab, but it was one I enjoyed, so there's that. One thing I am aware I did wrong, however, was spending too much time with machines I struggled to solve. Instead of moving forward, trying new boxes and come back later, I could spend days or even weeks trying a single privilege escalation, which really increased my frustration and made me needing more "vacation days" than necessary.

Related to that, it was incredibly useful to take detailed notes of every major step of every box I compromised in the lab. It doesn't only keep you motivated by writing "ticks" on the machines you completed, but also serves as a record of the exploit and techniques used. More than once, I needed to go back to a machine I already solved for whatever reason, and keeping the "how-to" written down was absolutely time-saving. Oh, and as an extra hint, it was useful to keep track of some occasional loot I found on certain machines. Surprisingly, it came in handy deeper in the lab!

Another thing I noticed is that the materials don't give you everything you'll need to solve *all* the boxes. They give you the basics, and you have to elaborate from there (elaborate is a fancy work for *furiously googling in search of answers*). A lot of machines depend on finding the specific, public exploit, so making yourself comfortable with [exploit-db][7] and specially its shell utility `searchsploit` is very useful. But others, specially the hard ones, have somewhat obscure/less known vulnerabilities that will require patience and Google-Fu.

Now, the big question. When you are ready for the exam? One never knows. Some say if you get the Big Three, you are set. Others say you should solve a good chunk (or all!) of the lab machines to consider yourself ready. Just like the "when to start the course question", it's more of a personal matter. But as general advice, you're probably fine if you got around 20~30 lab machines, you've practiced the buffer overflow explained in the materials, and you got your methodology right (enumeration being probably the most important skill). Also, It's probably a good idea to get the basic privesc tricks at hand, both Linux and Windows. [g0tmi1k][8] and [fuzzysecurity][9] have excellent guides that cover these subjects.

## The exam

Eventually, the day arrives. You reserved the date in your student panel, your lab time probably expired some days (or weeks) ago, you stocked up coffee and chocolate (for the sugar rush) and told your assistant to "cancel all your appointments". The 24-hour exam plus 24-hour report is a dreadful perspective, but as my physics professor back in high school used to say: "it's only a matter of assuming the amount of work". Which probably means you shouldn't be thinking in the work that's left to do, but the work you are doing right now and will do just next.

Obviously, I can't recommend a perfect way of approaching the exam. But here are a few things that worked out for me:

+ Started at 8 AM on a Saturday. That meant I had to reserve the date some weeks ahead, but it paid off. Starting fresh after sleeping well, knowing you can sleep the whole day once you finish, is great. Don't rush it to a Wednesday at 9 PM just because it's the first date available, unless you really want to because you work best at nights or whatever. It's better to wait a little and get the date you really feel comfortable with.

+ Know when to move on. Even though I actually got quite lucky and solved the machines as I approached them (until the last two - then my personal hell began, but that's another story), that doesn't mean it always goes that well. My advice would be: enumerate fast, look around while a deeper enumeration is in background, and if when it finishes you don't see a clear path right away, try to move on to the next machine. Save the deep, exhausting digging for later, or you will burn out pretty fast.

+ As with regular exams, I like to start with the easiest parts, gain confidence, and progress to the harder ones. Maybe some people don't like this approach, but it worked like a charm with me. Mentally, it's better to have 60% done in 40% of the time, than the other way around, even if that 40% is the hardest part. But of course, do as you feel suits you best.

+ Eat, rest, sleep. This has been said a million times, but its worth mentioning. I started my exam with a good breakfast, then stopped for lunch around 5h later, continued for around 4 hours more, and got stuck. I got really frustrated at this point and felt I wasn't gonna pass. Then I decided to stop, take a rest, watch some TV series, take dinner, and only after feeling refreshed, I continued. Around 3 hours later, I popped one of the two machines left, and went to bed knowing I had passed. Again, these 6 hours of sleep gave me the energy to get the remaining box, just for the challenge, and score a 100 out of 100. I hope this little story stresses enough what I'm trying to say: refreshing is *super* important. The brain works in background when you are resting/sleeping - you can wake up just to find out your brain actually found the solution you couldn't see after 6 hours straight of miserable attempts.

+ Have fun! This seems stupid advice, but it's not. I actually had a lot of fun during the exam (not so much during my "panic moment", but yes overall), and I'm sure that improved my performance by a noticeable amount. Try to get in the mood like you're in the lab, or solving a CTF challenge, or whatever stimulates you. In my case, listening to [certain music][10] did a lot to keep me centered, but you may prefer white noise, or silence. In the end, as in almost everything pointed out in this review, it's up to you.

## Conclusion

PWK is a wonderful learning experience, as well as the OSCP exam. I now understand why it has this near-mythical aura around it, with the "Try Harder" motto and everything. I've already said it, but even though the materials are great, what really makes you work, think and learn are the labs and the exam. I'm not saying the exercises and the videos are not important enough, it's just I prefer to actually *do* things, because it's the only way to demonstrate others and yourself that you really learned something. And it's virtually impossible to pass this course and get the certification if you don't *do* a lot of things beforehand, so you can tell for sure that, whoever got OSCP, they did their homework.

In short, practice, practice, practice. The hours invested, the frustration, everything pays off when you get that beautiful prompt on screen and the high kicks in. After experiencing it a few times, you get addicted. So much, actually, that it's been barely a week after I passed OSCP, and I'm already thinking in starting Crack the Perimeter (CTP) to get my OSCE. I heard it's a very different thing, but I'm completely sure the guys at Offensive Security won't disappoint.

[1]: https://www.vulnhub.com/
[2]: https://www.hackthebox.eu/
[3]: https://g0blin.co.uk/
[4]: https://www.youtube.com/channel/UCa6eh7gCkpPo5XXUDfygQQA
[5]: https://exploit-exercises.com/
[6]: https://github.com/radare/radare2
[7]: https://www.exploit-db.com/
[8]: https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/
[9]: http://www.fuzzysecurity.com/tutorials/16.html
[10]: https://www.youtube.com/watch?v=4lXtqEv4p9A
