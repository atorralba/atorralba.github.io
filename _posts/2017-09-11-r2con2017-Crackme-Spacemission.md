---
layout: post
title: r2con 2017 Crackmes - Spacemission
---

This was my first year attending [r2con](http://radare.org/con/2017/), and I can assure you I'm 100% coming back next year! It was lots of fun, I learned *a lot* in the trainings and the talks were super interesting. But, as a complete **noob** with `radare2` (and reversing in general), one of the things I enjoyed the most were the [Crackmes](http://radare.org/con/2017/crk/#download-the-crackmes). Well, actually *the* crackme, because I only managed to solve one, the easiest of them: **spacemission**. Here I will try to explain how I approached this challenge from beginning to end, of course using `radare2` during the whole process!

## Hello, chief reverse engineer

After downloading the [binary](http://radare.org/con/2017/crk/files/spacemision), let's copy it into a docker container (one never knows) and execute it right away. The following appears on screen:

```
$ chmod +x spacemission && ./spacemission
Hello, ...?
Hello, chief reverse engineer root of the spaceship rbinsegfaulter?
Can you hear, me?
Oh, these speakers seem to be broken.
No matter, if you hear me, or not, this is probably our last chance to survive!
We got attacked from the evil aliens from the binja-system!
Their attack was surprising for us. They killed almost all of us, none of our engineers survived.
Before they left, they planted bombs in the ground, each of them with enough destructive power to blowup the planet
But there is hope, one of our bomb disposal rovers out there is still instact, and waiting in idle for transmition of instruction.
I know, you are just a reverse engineer, but at least you are an engineer, and the only one in reach, who can help us, now.
If you can hear this message, hurry up and create an instruction file for the rover.
Note, that the battery, of the rover is very low.
Restart this communication program, to transmit the instruction file
Good luck!
```

Ok, sounds fun! We have to save this poor planet from destruction by disarming the bombs left behind by these evil aliens. But, mmm, **how** do we do it?

The binary seems to ignore any parameter passed to it on invocation, so it seems that we need to follow the instructions given to us and create an instruction file. But we don't even know the name of the file, not to say the contents of it, so it's time to load the binary into `radare2` and see what happens!

```
$ r2 -d spacemision
Process with PID 517 started...
= attach 517 517
bin.baddr 0x00400000
Using 0x400000
asm.bits 64
 -- You can mark an offset in visual mode with the cursor and the ',' key. Later press '.' to go back
[0x7f1922718c30]>
```

Let's analyze everything since the binary isn't too large, and then navigate to the main function to see what's there:

```
[0x7f1922718c30]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
TODO: esil-vm not initialized
[Cannot determine xref search boundariesr references (aar)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Use -AA or aaaa to perform additional experimental analysis.
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
ptrace (PT_ATTACH): Operation not permitted
= attach 517 517
[0x7f1922718c30]> s main
[0x00400ba0]>  pd 20
/ (fcn) main 3355
|   main ();
|           ; var int local_b0h @ rbp-0xb0
|           ; var int local_80h @ rbp-0x80
|           ; var int local_18h @ rbp-0x18
|           ; var int local_10h @ rbp-0x10
|           ; var int local_9h @ rbp-0x9
|           ; var int local_8h @ rbp-0x8
|           ; var int local_4h @ rbp-0x4
|              ; DATA XREF from 0x0040078d (entry0)
|           0x00400ba0      55             push rbp
|           0x00400ba1      4889e5         mov rbp, rsp
|           0x00400ba4      4881ecb00000.  sub rsp, 0xb0
|           0x00400bab      c645f700       mov byte [local_9h], 0
|           0x00400baf      488d8550ffff.  lea rax, [local_b0h]
|           0x00400bb6      4889c6         mov rsi, rax
|           0x00400bb9      bf571d4000     mov edi, str.robot_instructions ; 0x401d57 ; "robot_instructions"
|           0x00400bbe      e87d0d0000     call fcn.00401940
|           0x00400bc3      85c0           test eax, eax
|       ,=< 0x00400bc5      740a           je 0x400bd1
|       |   0x00400bc7      b800000000     mov eax, 0
|      ,==< 0x00400bcc      e9e80c0000     jmp 0x4018b9
|      |`-> 0x00400bd1      c60574252000.  mov byte [0x0060314c], 1    ; [0x60314c:1]=0
|      |    0x00400bd8      be00000000     mov esi, 0
|      |    0x00400bdd      bf571d4000     mov edi, str.robot_instructions ; 0x401d57 ; "robot_instructions"
|      |    0x00400be2      b800000000     mov eax, 0
|      |    0x00400be7      e864fbffff     call sym.imp.open           ; int open(const char *path, int oflag)
|      |    0x00400bec      8945f0         mov dword [local_10h], eax
|      |    0x00400bef      b800000000     mov eax, 0
|      |    0x00400bf4      e8d4fdffff     call sub.calloc_9cd         ; void *calloc(size_t nmeb, size_t size)
[0x00400ba0]>
```

Alright, it seems pretty clear how our file must be named. The string `robot_instructions` before the `sym.imp.open` call is a pretty decent hint! Let's create it and try again:

```
$ touch robot_instructions
$ ./spacemision
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
X                   \                  X
X                    \########         X
X      o                               X
X    \OO                               X
X       \                              X
X___-##                                X
X #  ##/\/\/\/\/\/\/\/\                X
X_#__##                |               X
X  | ##  ____#----__---|               X
X  |    /##  | #       |               X
X  |    |    |         |/\  /\/\/\/\/\/X
X   \==/     |##       |               X
X            |         |/\/\/\/\/\/\/\ X
X####__########_#######                X
X                      \/\/\/\  /\/\/\/X
X                                      X
X                                      X
X                                      X
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
BOOOOOOM!!!
```

Cool! Even our inactivity caused the bomb to explode, we at least got what seems like a map in our screen. Now we should try to understand how our rover is moved, i.e. what commands we have to write into the `robot_instructions` file.

## Up, Down, Left, Right

Once opened, a file must be read, so a good place to look at should be around `sym.imp.read` calls.

```
[0x00400ba0]> axt sym.imp.read
call 0x4009c2 call sym.imp.read in sub.read_9a6
```

Ok, `sym.imp.read` is called inside the function `sub.read_9a6`, where is that function called from?

```
[0x00400ba0]> axt sub.read_9a6
call 0x401633 call sub.read_9a6 in main
```

Aha! Let's take a look:

```
[0x00400ba0]> s 0x401633
0x00401633]> pd 20
|           0x00401633      e86ef3ffff     call sub.read_9a6           ; ssize_t read(int fildes, void *buf, size_t nbyte)
|           0x00401638      8845f7         mov byte [local_9h], al
|           0x0040163b      0fbe45f7       movsx eax, byte [local_9h]
|           0x0040163f      83f85e         cmp eax, 0x5e               ; '^' ; '^' ; 94
|       ,=< 0x00401642      742f           je 0x401673
|       |   0x00401644      83f85e         cmp eax, 0x5e               ; '^' ; '^' ; 94
|      ,==< 0x00401647      7f13           jg 0x40165c
|      ||   0x00401649      83f83c         cmp eax, 0x3c               ; '<' ; '<' ; 60
|     ,===< 0x0040164c      0f84b3010000   je 0x401805
|     |||   0x00401652      83f83e         cmp eax, 0x3e               ; '>' ; '>' ; 62
|    ,====< 0x00401655      7479           je 0x4016d0
|   ,=====< 0x00401657      e916020000     jmp 0x401872
|   |||`--> 0x0040165c      83f876         cmp eax, 0x76               ; 'v' ; 'v' ; 118
|   |||,==< 0x0040165f      0f84c8000000   je 0x40172d
|   |||||   0x00401665      83f879         cmp eax, 0x79               ; 'y' ; 'y' ; 121
|  ,======< 0x00401668      0f84ed010000   je 0x40185b
| ,=======< 0x0040166e      e9ff010000     jmp 0x401872
```

Perfect, it seems our guessing was right! The file seems to be read, one byte at a time,and then compared to one of the following characters: `^`,`<`,`>`,`v`,`y`. The first four seem quite obvious: the rover probably goes up, left, right and down respectively each time one of these characters is read. A quick test shows we are right, as a little arrow appears on screen performing the moves we introduce into the `robot_instructions` file.

But, there's that `y` command which doesn't directly translate to any movement (the rover stands still). Maybe we should dive a bit more and see what the code does when a `y` is read.

## The `y` command

Following the `jmp` address we can try to see what happens with this command:

```
[0x00401633]> s 0x40185b
[0x0040185b] pd 6
|           0x0040185b      8b15e3182000   mov edx, dword [0x00603144] ; [0x603144:4]=0
|           0x00401861      8b05d9182000   mov eax, dword [0x00603140] ; [0x603140:4]=0
|           0x00401867      89d6           mov esi, edx
|           0x00401869      89c7           mov edi, eax
|           0x0040186b      e87af2ffff     call fcn.00400aea
|       ,=< 0x00401870      eb04           jmp 0x401876
```

Hmmm. Two values are read from memory (`0x00603144` and `0x00603140`), moved to registers and then a function is called. Following the breadcrumbs:

```
[0x0040185b]> s fcn.00400aea
[0x00400aea]> pd 34
/ (fcn) fcn.00400aea 108
|   fcn.00400aea (int arg_6h);
|           ; var int local_18h @ rbp-0x18
|           ; var int local_14h @ rbp-0x14
|           ; var int local_6h @ rbp-0x6
|           ; var int local_4h @ rbp-0x4
|           ; arg int arg_6h @ rbp+0x6
|              ; CALL XREF from 0x0040186b (main)
|           0x00400aea      55             push rbp
|           0x00400aeb      4889e5         mov rbp, rsp
|           0x00400aee      897dec         mov dword [local_14h], edi
|           0x00400af1      8975e8         mov dword [local_18h], esi
|           0x00400af4      8b45ec         mov eax, dword [local_14h]
|           0x00400af7      c1e006         shl eax, 6
|           0x00400afa      6625c007       and ax, 0x7c0
|           0x00400afe      89c2           mov edx, eax
|           0x00400b00      8b45e8         mov eax, dword [local_18h]
|           0x00400b03      01c0           add eax, eax
|           0x00400b05      83e03e         and eax, 0x3e
|           0x00400b08      09d0           or eax, edx
|           0x00400b0a      668945fa       mov word [local_6h], ax
|           0x00400b0e      c745fc000000.  mov dword [local_4h], 0
|       ,=< 0x00400b15      eb36           jmp 0x400b4d
|      .--> 0x00400b17      8b45fc         mov eax, dword [local_4h]
|      ||   0x00400b1a      4898           cdqe
|      ||   0x00400b1c      0fb784009030.  movzx eax, word [rax + rax + 0x603090] ; [0x603090:2]=460
|      ||   0x00400b24      663b45fa       cmp ax, word [local_6h]
|     ,===< 0x00400b28      751f           jne 0x400b49
|     |||   0x00400b2a      8b45fc         mov eax, dword [local_4h]
|     |||   0x00400b2d      4898           cdqe
|     |||   0x00400b2f      0fb784009030.  movzx eax, word [rax + rax + 0x603090] ; [0x603090:2]=460
|     |||   0x00400b37      83c801         or eax, 1
|     |||   0x00400b3a      89c2           mov edx, eax
|     |||   0x00400b3c      8b45fc         mov eax, dword [local_4h]
|     |||   0x00400b3f      4898           cdqe
|     |||   0x00400b41      668994009030.  mov word [rax + rax + 0x603090], dx ; [0x603090:2]=460
|     `---> 0x00400b49      8345fc01       add dword [local_4h], 1
|      !|      ; JMP XREF from 0x00400b15 (fcn.00400aea)
|      |`-> 0x00400b4d      837dfc06       cmp dword [local_4h], 6     ; [0x6:4]=-1 ; 6
|      `==< 0x00400b51      7ec4           jle 0x400b17
|           0x00400b53      90             nop
|           0x00400b54      5d             pop rbp
\           0x00400b55      c3             ret
```

Wow, this one's a little bigger. Let's try to understand it step by step. First, the two values previously read from memory (now saved into `edi` and `esi`) are moved into two local variables, `local_14h` and `local_18h`. Then, a series of transformations are applied to these values. Summarized:

+ `local_14h` is shifted 6 bits to left (or multiplied by 64) and then bitwise-`and`'ed with the constant value `0x7c0`. The resultant value is saved into `edx`.
+ `local_18h` is summed to itself (or multiplied by 2), bitwise-`and`'ed with the constant value `0x3e` and then bitwise-`or`'ed with the previous value stored in `edx`. The resulting value is saved in `local_6h`.

At this point, the transformation is finished and a loop begins, using `local_4h` as the iterator, beginning with 0 and ending with 6, so that's 7 iterations. In every iteration, memory is accessed at the base address of `0x603090` plus 2 * the current iteration (that's 0, 2, 4...). The value obtained from memory is compared to the transformed value that's in `local_6h`.

Let's stop here for a moment and make and educated guess. From what we have seen, when a `y` is read, two values are passed to a function. Since this is a 2D game, the probable candidates for this two values are the coordinates X and Y. This can be confirmed dynamically, setting breakpoints with `db` at the beginning of the function and seeing that, by writing `y<y` into `robot_instructions`, the execution stops twice: `exi` has the same value both times (`0x4`), but `edi` decreases in 1 the second time (so it has the value `0x3`). This seems to indicate that `esi` (`local_18h`) contains the Y coordinate and `edi` (`local_14h`) the X coordinate.

If this is true, what is happening next seems pretty obvious: X and Y are transformed (hashed/obfuscated) into a single value, and that is compared to a value read from memory. If they are the same, that memory address is written with something else. If not, the loop continues and the next memory address is compared to the value obtained from the current coordinates, up to 7 comparisons. Then the function ends.

Whew, did you read all that? Well, if you did, you probably know what's up by now, but if you didn't here's a little `TL;DR`:

+ The `y` command attempts to deactivate a bomb in the current location
+ 7 positions are obtained from memory, so probably there are 7 bombs in the whole map

This is getting interesting! We only have to get the 7 bomb positions from memory and we are done, right? Well, not quite.

## The 7 bombs

By placing a breakpoint at `0x00400aea`, we can see the 7 values compared to our obfuscated coordinates and save them. These are the obtained values:

```
0x1cc 0x5c2 0x04e 0x186 0x394 0x0e0 0x61c
```

Of course, these are obfuscated with the same transformations applied to the current coordinates, so we still don't have the positions for the 7 bombs. At this point, I thought of writing a "deobfuscation" function that could obtain the coordinates for every obfuscated value. But after some minutes of trying, I started to see maybe there was an easier way.

We are playing in a 40x20 map. That's 800 possible XY combinations where our rover can be at a given time, without taking obstacles into consideration. That is a very reasonable value to try a "brute-force" attack, as if the positions of the bombs where hashed and we were trying to crack them. And coding the obfuscation function is way easier, as it's only a matter of translating assembly to Python. The following script accomplishes exactly that:

```python
def obfuscate(x, y):
  temp = x*64
  temp = temp & 1984
  temp2 = y*2
  temp2 = temp2 & 62
  return temp | temp2

values = [ 0x1cc, 0x5c2, 0x04e, 0x186, 0x394, 0x0e0, 0x61c ]
for x in range(40):
  for y in range(20):
    value = obfuscate(x,y)
    if value in values:
      print x,y,hex(value)
      values.remove(value)
```

Drum roll, please!

```bash
$ python bombs.py
1 7 0x4e
3 16 0xe0
6 3 0x186
7 6 0x1cc
14 10 0x394
23 1 0x5c2
24 14 0x61c
```

*Hacker voice*: Got it.

## A matter of optimization

We are done! Our reversing skillz are the best ever! We are l33t h4ck3rs! Now we just have to write a path that walks over all the positions, deactivates all bombs and we won, right? Right?

Well, it seems not. Remember that message at the beginning that mentioned the rover's battery is limited? That means we have a **limited** number of movements before the battery dies. Uh, oh.

Yes, it's only a matter of finding the optimal way. And yes, there is a "trick" that I discovered by chance but that can be found in the code: the `_` characters in the map are special, in the sense that they are not the same kind of obstacle as the rest of them (`X`, `#`, `-` and so on). If the rover walks into a `_` *from up to down*, it jumps above it instead of stopping. Cool! That's actually the only way of reaching the bomb at `(1,7)`, and it was the only reason I discovered this. Later, I checked in the code that the `v` command has a special treatment just for this case.

But even with this trick, I kept writing paths of >190 movements, and the battery of our rover only lasts for 179. Here I started thinking on finding the optimal path programatically, but a part of me refused the approach as over-complicated. "It's only a one star challenge, it should be easier", I thought. And then began a deep navigation into the assembly, trying to find other special obstacles that maybe allowed for "teleportation" from one point to another, like wormholes or something.

Needless to say, that led to nowhere. After an undisclosed amount of hours, I fell back into the good old manually counting positions and discovered a few optimization errors in my path. I reduced it to 185, then 183, and then, finally, 179 movements. It ultimately looked like this:

```
$ cat robot_instructions
^>>y
<<vv<v>vvv>>^>^>^>>>>>>>>>>v<v<<<^y
v>>vv<vv>>>>>>>>>>>>>>>^^<<<<<<y
>>>>>>>>>>>>>>^^<<<<<<<<<<<^^>>>^^^^^^^^^<<<<<<<y
>>>>>>>vv<<<<<<<<<<<<<<<<<<<<<vvv<<y
^<<<<<<vy
vvvvv>>>>vv<<y
```

Was it victory?

```
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
X                   \                  X
X                    \########         X
X      o                               X
X    \OO                               X
X       \                              X
X___-##                                X
X #  ##/\/\/\/\/\/\/\/\                X
X_#__##                |               X
X  | ##  ____#----__---|               X
X  |    /##  | #       |               X
X  |    |    |         |/\  /\/\/\/\/\/X
X   \==/     |##       |               X
X            |         |/\/\/\/\/\/\/\ X
X####__########_#######                X
X                      \/\/\/\  /\/\/\/X
X  <                                   X
X                                      X
X                                      X
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
You saved us all, thank you!
```

Yay!

## Conclusion

To wrap up, I got a lot of fun with this challenge. It was my first time using `radare2`, and almost my first reversing challenge too. I learned quite some `r2` commands in the process, and lost a little fear on reading assembly ;-)

The final part of finding the optimal path was a little frustrating, and I still feel maybe there was a way to automate the process. But well, sometimes banging your head against the wall is what you need to take a deep breath and reconsider your options.

Big thanks to **condret** for providing the challenge, to **pancake** and the rest of the **r2con2017** organization for setting up such a wonderful event, and to [t0n1](http://hacktracking.blogspot.com.es/) and [as0ler](https://twitter.com/as0ler) for offering their help when I got stuck.
