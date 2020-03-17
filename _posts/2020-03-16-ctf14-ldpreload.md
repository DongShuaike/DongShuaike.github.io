---
layout: post
title:  "The LD_PRELOAD usage in ctf"
date:   2020-03-16 13:56:45 +0800
categories: ctf binary RE
---

Recently I was thinking how to make my reverse engineering skills stronger so that I can perform better in an interview, I decided to start working on CTF challenges during which I can review some basic background and learn some fancy operations. The first day I selected one problem featured "usage of **LD_PRELOAD**"

The problem is named ***Hack In The Box Amsterdam: Bin 100*** (2014), in which we are given a `elf` binary called hitb_bin100.elf.

First let's `file` it to see the basic information.
<<<<<<< HEAD
![](/images/20200311-114439.png)
As the image shown, the binary is a 64-bit dynamically linked file.

Now run it `chmod +x ./hitb-bin100.elf && ./hitb-bin100.elf`.
![](/images/20200311-114647.png)
=======
![](/ctf/binary/re/2020/03/16/imgs/20200311-114439.png)
As the image shown, the binary is a 64-bit dynamically linked file.

Now run it `chmod +x ./hitb-bin100.elf && ./hitb-bin100.elf`.
![](/ctf/binary/re/2020/03/16/imgs/20200311-114647.png)
>>>>>>> 1591b672748d3dba9b3f2c5035135db7912c5e13

The binary keeps printing out random lyrics with an around one-second time gap in between.

I then dragged it into ida-pro to see what's inside.![](/images/20200311-114836.png) 

The control flow graph seems normal with two obvious loop structures (red and blue).

Following the control flow I found a basic block(actually not a real basic block, it's just a unit block in ida super control flow graphs). 
<<<<<<< HEAD
![](/images/20200311-115032.png)
=======
![](/ctf/binary/re/2020/03/16/imgs/20200311-115032.png)
>>>>>>> 1591b672748d3dba9b3f2c5035135db7912c5e13

This piece of code make me review how arguments are passed in x64 assembly:

> Generally when there are fewer than **7** arguments we will store them into registers: **rdi, rsi, rdx, rcx, r8, and r9** from **left** to the **right**. While when the number of arguments is larger thant 7, those extra arguments are **pushed into the stack** from **right** to **left**, which stays the same as x86 assembly.

> Apart from that, the return value of a function is first passed to **rax**. If there is a second return value, it will be passed to **rdx**.

OK, now let's see what does this code block do by translating it into the C code.

1. The prototype of `time()` in C is `time_t time (time_t* timer)`, it will return the current time information in a `time_t` struct object.
2. After that, the address of that  `time_t` object is  passed to `srand` as the argument. The `srand()` function is to initialize the psedu-random number generator with the current time info.
3. `rand()` function executes, producing a psedu-random number and stores it in register `r13d`. (Note that `d` in `r13d` refers to `double-word` which means `r13d` has the size of 32 bits.)
4. The value of `r13d` is stored to `[rsp+rbx+198h+var_188]` and then the lyric with the offset `[rbx*8]` in `funny` variable is fetched and stored into `r13`.

Now let's move to the next code block.
<<<<<<< HEAD
![](/images/20200311-123753.png)
=======
![](/ctf/binary/re/2020/03/16/imgs/20200311-123753.png)
>>>>>>> 1591b672748d3dba9b3f2c5035135db7912c5e13

Inside it we find there is a string searching operation done by `repne scasb`. For `repne scasb`, we just search `eax` in target string `[rsi:rdi]`. In this case, `eax` is zero and therefore the code snippet is actually looking for the end of a C string. `rcx` keeps the number of attempts to do the search, which is the length of the target string.

`rcx` is then compared with `r12` which has been `xor`ed to zero in the previous block. Let's see what if the string being scanned is not zero length.

<<<<<<< HEAD
![](/images/20200311-124748.png)

If the string length is not zero, the string will be output by `printf_chk` and then the machine will **sleep** for one second.

![](/images/20200311-124917.png)

And until `[rsp+198+var_190]` is decreased to zero, will we jump to the final treasure target:

![](/images/20200311-125010.png)
=======
![](/ctf/binary/re/2020/03/16/imgs/20200311-124748.png)

If the string length is not zero, the string will be output by `printf_chk` and then the machine will **sleep** for one second.

![](/ctf/binary/re/2020/03/16/imgs/20200311-124917.png)

And until `[rsp+198+var_190]` is decreased to zero, will we jump to the final treasure target:

![](/ctf/binary/re/2020/03/16/imgs/20200311-125010.png)
>>>>>>> 1591b672748d3dba9b3f2c5035135db7912c5e13

There we output the value of the key.

Where do we set the value of `[rsp+198+var_190]`?
<<<<<<< HEAD
![](/images/20200311-125140.png)
=======
![](/ctf/binary/re/2020/03/16/imgs/20200311-125140.png)
>>>>>>> 1591b672748d3dba9b3f2c5035135db7912c5e13

At this step, we have the intuition of capturing the flag -- Just accelerating the loops, so that 31337 can be more quickly deduced to zero.

The first idea is to bypass all the loops and directly run the block 0x4007e8. However, this cannot work, since the key value is fetched from `[rsp+198h+var_188]`, which is generated by `rand` function byte-by-byte during the big loop.

**Note**: As the below image shows, the value of the key is generated byte by byte during the loop. Since the initial seed of the `rand` is always the same whenever you run the program (the result `lea edi, [rbp+rax+0]` will be the same)

<<<<<<< HEAD
![](/images/20200311-131024.png)
=======
![](/ctf/binary/re/2020/03/16/imgs/20200311-131024.png)
>>>>>>> 1591b672748d3dba9b3f2c5035135db7912c5e13

Now comes to the second idea, we change the `sleep(1)` to `sleep(0)`. Does it work?

No. The result is like below.
<<<<<<< HEAD
![](/images/20200311-143849.png)
=======
![](/ctf/binary/re/2020/03/16/imgs/20200311-143849.png)
>>>>>>> 1591b672748d3dba9b3f2c5035135db7912c5e13

Why `sleep()` matters a lot?






