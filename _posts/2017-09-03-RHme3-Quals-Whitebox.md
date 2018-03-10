---
layout: post
title: RHme3 Quals - Whitebox
---

OK, let's begin with a shameful confession: I had **absolutely no idea** about White-Box Cryptography before starting this challenge. That means I had to read quite a lot about it, understand its purpose, strengths and flaws, before even considering solving it. In this write-up, I will try to explain my approach to the challenge, the problems I encountered and how I finally got the solution. Let's begin!

## First things first

So we are given a binary, and we are told to extract a cryptographic key from it. Seems easy, right? Well, it wasn't! At least for me.

As I said above, the first thing I had to do was reading an awful lot. After learning in general terms what White-Box Cryptography is and what it is useful for, it was time for the specifics and I started looking for practical attacks on White-Box implementations of common crypto algorithms.

My colleague [t0n1][1] kindly pointed me out [Unboxing the White-Box][2], an excellent paper about just that, conveniently written by Riscure people themselves. Judging by the titles of both the paper and the challenge, it promised to be a central piece of the solution (*spoiler alert*: it absolutely was!), so I read it voraciously.

The reading helped me with two things: firstly, learning a bit more about White-Box crypto in general, and secondly, understanding the attacks of Fault Injection and Side-Channel Analysis. Now that I had at least some ideas about how to approach my enemy (the binary), it was time to get my hands dirty.

## Poking

I started poking the binary to try to understand what it was doing. It seemed quite simple: given a 16-byte input, an encrypted 16-byte output was returned.

```bash
$ ./whitebox AAAABBBBCCCCDDDD
18 20 4c 40 8d 7b c0 f9 2c c8 07 7d a6 94 27 42
```

It also seemed interesting that even when less than 16 bytes were given the output had 16 bytes anyway. It was absolutely too soon to tell, but if I had to do so at the moment, I would've said that we were facing an **AES 128** White-Box implementation. I decided to keep that consideration until reality proved me wrong, and so I spent another good amount of time reading about that specific algorithm, trying to understand how it works.

After that, the thing was: OK, I somewhat understand White-Box crypto, I somewhat understand how AES works, and I have a binary that **apparently** is encrypting the input it receives with a White-Box AES. Where to go from here?

After going against some walls, I stumbled upon a presentation that linked to a [GitHub repo][3] that seemed promising. This good guys did not only write functional code of Fault Injection (and other) attacks against AES White-Box implementations, but also named their tools after Marvel characters! How cool is that?!

At first, [Deadpool][4] looked like our guy: he injects faults *statically* into the binary and tries to obtain faulty outputs. The sad part is, no matter how much I tried to adjust the parameters of the tool, I wasn't able to obtain a single faulty output :-(

Then I looked into [JeanGrey][5], the tool that accomplishes phase 2: this is, analyzing the faulty outputs and obtaining the key (i. e. the real hard work). A lot of magic happens inside the code of this tool, and I tried to understand some of it reading the [paper][6] in which the attack JeanGrey uses is explained in detail. I leave to the curious reader the work of further investigating this path, but for the sake of simplicity, we will continue with the following assumptions:

+ JeanGrey performs [Differential fault analysis][7] to obtain cryptographic keys from White-Box implementations (specifically AES)
+ It needs a file the first line of which must be the original, unmodified output for a given input, and the following lines must contain faulty outputs for the given input
+ It works "running the algorithm backwards", i.e. obtaining the key for the last AES round, then using it plus the faulty outputs of the previous round to find the corresponding key, and so on until the first key is found, and with it the full AES key.

So it seemed we had a clear target: to generate enough faulty outputs for each round so JeanGrey could process them and help us find the keys. Now here is where the hard work came: we had to look at the internals of the binary in search of good fault injection locations.

## RE

Well, I have to admit I had some (big) help at this point by the hand of the almighty [t0n1][1], so this section will be a little summarized.

He quickly found an interesting function that got the 16 input bytes and copied them into a buffer. Then, it started applying some transformations to this buffer, transformations that, upon further inspection, very conveniently looked like the different steps of the [AES algorithm][9]. Of course it was not as easy as inspecting the `KeyExpansions` step and finding the key there, because we were facing a White-Box implementation of the algorithm, that's the whole point! But the possibilities of this being AES increased.

Then we saw a counter that repeated the same steps (what we thought were the `SubBytes`, `ShiftRows`, `MixColumns` and `AddRoundKey`, thought we struggled to find each one in the correct order in the assembly code) up to 9 times. It made sense, because the last round skips the `MixColumns` step and so the code must jump to another region.

If we were correct, we had found strong hints that pointed to AES and, what's more important, we may had identified the memory address where each round of the algorithm started. We also knew that the input buffer was copied into another structure with the same size, which couldn't be anything else but the *state* (a 4 x 4 matrix of bytes used by the AES algorithm to perform the cipher). With this information, we had a place (the *state* matrix) and a time (the round start) to try and inject some faults.

## Ready, Aim, Fire

Now that we were pretty sure that we were facing an AES algorithm and had an apparently accurate memory address to inject faults into, it was time to decide how to do so.

After a little tinkering, I wrote this dirty but functional "script" that injects a byte-long fault in the specified memory address for the desired AES round:

```bash
b *0x4636EA
ignore 1 $round
run AAAABBBBCCCCDDDD
d 1
print /x 0x7fffffffeb20+$offset
set *(char*)$1=$fault
c
quit
```

In short, what it does is:

+ Sets a breakpoint at the start of the code that handles an AES round
+ Ignores the first `n` times the breakpoint is reached. This is to be able to inject faults in the round we want to (e.g. if we ignore the first 7 times the breakpoint is reached, we are injecting faults in AES round 8)
+ Executes the binary with the input `AAAABBBBCCCCDDDD` until the breakpoint is reached
+ Deletes the breakpoint as it isn't needed anymore
+ Calculates the specific byte of the AES data matrix from its start address plus a desired offset (0 for the first byte, 1 for the second, etc)
+ Injects the desired fault in the address calculated before
+ Continues the execution
+ Quits GDB

So, with this, we have the ability to inject an arbitrary fault in an arbitrary matrix position for any of the AES rounds. Now we want to do this for *every* round, *every* position and a handful of faults. Again, the following ugly script achieves this:

```bash
rounds=('10 9 8 7 6 5 4 3 2')
faults=('0x41 0x42 0x43 0x44 0x45 0x46 0x47 0x48 0xff 0x00')
offsets=('0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15')
for round in ${rounds[@]}; do
	echo "Round $round"
	echo "$(./whitebox AAAABBBBCCCCDDDD | sed -e "s/ //g")" > round$round.log
	for fault in ${faults[@]}; do
		for offset in ${offsets[@]}; do
			echo "$(cat fault.gdb | sed -e "s/\$round/$((round-2))/" | sed -e "s/\$fault/$fault/" | sed -e "s/\$offset/$offset/" | gdb -q whitebox | grep -v gdb | grep -v Inferior | grep -v Breakpoint | grep -v Reading | grep . | sed 's/ //g' 2>/dev/null)" >> round$round.log
		done
	done
done
```

I know this is horrible (so please don't hate too much), but it does the trick: after the execution, 9 `round<i>.log` files are generated, the first line of each is the unmodified, encrypted output for the input string `AAAABBBBCCCCDDDD`, and the rest are faulty outputs for the specified round. With this files, I finally could summon the phoenix and use JeanGrey to try and get the keys of each AES round. With that in mind I wrote the following python script:

```python
import phoenixAES

foundkey = True
lastroundkeys = []
round = 10
while foundkey and round > 1:
    print("Round %i" % round)
    foundkey = False
    k = phoenixAES.crack('faults/round%d.lst' % round,
        lastroundkeys=lastroundkeys,
        outputbeforelastrounds=False,
        verbose=1)
    if k:
        foundkey = k not in lastroundkeys
        lastroundkeys.append(k)
        round -= 1

with open('lastroundkeys.lst','w+') as f:
    for key in lastroundkeys:
        f.write("%s\n" % key)
```

Which basically tries to find the key of round 10 with that round's faulty outputs, and if successful, tries the same with round 9 plus round 10 key, and so on until all keys are recovered. I was very excited when I saw the output:

````
Round 10
Last round key #N found:
4E44EACD3F54F5B54A4FB15E0710B974
Round 9
Round key #N-1 found:
B7740F2E71101F78751B44EB4D5F082A
Round 8
Round key #N-2 found:
B75D7729C6641056040B5B9338444CC1
Round 7
Round key #N-3 found:
B3AD77C27139677FC26F4BC53C4F1752
Round 6
Round key #N-4 found:
44E7FF79C29410BDB3562CBAFE205C97
Round 5
Round key #N-5 found:
5CB6279A8673EFC471C23C074D76702D
Round 4
Round key #N-6 found:
C19FC271DAC5C85EF7B1D3C33CB44C2A
Round 3
Round key #N-7 found:
A244DC6E1B5A0A2F2D741B9DCB059FE9
Round 2
Round key #N-8 found:
051B4EE0B91ED641362E11B2E6718474
````
And yes, I know you noticed: round 1 isn't there, both the faults and the keys only work until round 2. But isn't that the point? To obtain the round 1 key and therefore the full AES key, which is the challenge's goal?

Well, the thing is that I *tried* to generate faults for round 1 (this is, before any transformation is performed on the data matrix), and even though I had to adjust the script specially for that (another address to inject the fault into, modify more than one byte at a time, and some other things), the faults that I obtained, even though apparently useful, weren't enough for JeanGrey to obtain the last (first round) key. God dammit!

## Extra round

At this point, I was desperate. So close, only one round key left, and yet so far, as I was unable to use all the previous keys with the faulty outputs of round 1 to obtain the damn full AES key! After banging my head against walls for around two days, I tried to expand my point of view. I thought that maybe I wasn't using the tools correctly, so I looked for examples of correct usage of `phoenixAES.py` (aka JeanGrey). And then I analyzed (more carefully this time) the code of `deadpool_dfa_experimental.py`, a kind of companion script for the Deadpool tool that pretty much does what I was trying to do here, with the only exception that my faults were dynamically generated.

The important point is that I discovered that Deadpool has a mode in which it's able to generate faults **directly on the input** of the binary. And isn't that the same than injecting faults into round 1? Of course it is! Now, I have to admit it: I have *no idea* why the faults generated this way worked but the ones generated by me didn't, but the thing is that, using Deadpool's function `runoninput` with all the previously obtained keys (rounds 10 to 2), JeanGrey finally was able to obtain the last round key.

Now it was only a matter of reversing the first stages of the AES algorithm to obtain the full key. I stole some more code from the good guys at [SideChannelMarvels][3] to perform just that:

```python
cint,_,_ = engine.doit(engine.goldendata, processinput(p, 16), lastroundkeys=[])
c = [(cint>>(i<<3) & 0xff) for i in range(16)][::-1]
kr0 = phoenixAES.rewind(cint, lastroundkeys, encrypt=encrypt, mimiclastround=False)

found_key = '%032X' % kr0
```

And then:

````
Round key #N-9 found:
C831FA90BC0598A18F30C7F3D05F95C6
First round key found?:
61316C5F7434623133355F525F6F5235
````

My hands trembled at this point, writing which I hoped was the last line of code to obtain the flag of the challenge:

```python
>>>'61316C5F7434623133355F525F6F5235'.decode('hex')
'a1l_t4b135_R_oR5'
```

Is it the flag? It looks like the flag! We got the flag! \o/

## Conclusion

This challenge was quite unique in its own right. I was forced to learn new things (mainly how White-Box Cryptography and AES work), read some papers, watch some talks... which was great. I also had the opportunity to assemble some code that, even though dirty, did the trick. And I always enjoy coding!

On the other hand, I felt at times I was blindly following a route opened by others. [t0n1][1] was of **huge** help, specially with the RE part, and when he got the flag by performing another attack in a much easier way (check his blog for details), I got quite a bit frustrated. But I decided to keep trying with my "own" method and my dirty scripts and my hundreds of faulty outputs until I got this right. And I'm happy I did!

The [SideChannelMarvels][3] repos were another example of "trusting in magic". I of course read the papers and tried to understand the tools and the attacks, even took some code and rewrote it a little. But in the end, the challenge was solved by throwing inputs at the tool and adjusting the whole thing until the right outputs were thrown back. I don't know, maybe the 100% right thing to do was to implement my own fault analysis tool from scratch just by reading the paper that describes the attack, but I felt that was beyond my current skill and time frame for the RHme3 quals.

To wrap up: fun challenge, I learned a lot, it was very rewarding when the flag appeared on screen. *But* I can't help but feeling a little "script-kiddie" using these on the other hand great tools without fully understanding them (to [SideChannelMarvels][3], you guys are great! Big thanks for the hard work!). But anyway, a flag is a flag and that means challenge complete.

## Code

All the scripts I used to solve this challenge can be found in this [GitHub repo][8].

[1]: http://hacktracking.blogspot.com
[2]: https://www.blackhat.com/docs/eu-15/materials/eu-15-Sanfelix-Unboxing-The-White-Box-Practical-Attacks-Against-Obfuscated-Ciphers-wp.pdf
[3]: https://github.com/SideChannelMarvels
[4]: https://github.com/SideChannelMarvels/Deadpool
[5]: https://github.com/SideChannelMarvels/JeanGrey
[6]: https://eprint.iacr.org/2003/010
[7]: https://en.wikipedia.org/wiki/Differential_fault_analysis
[8]: https://github.com/atorralba/rhme3
[9]: https://en.wikipedia.org/wiki/Advanced_Encryption_Standard#High-level_description_of_the_algorithm
