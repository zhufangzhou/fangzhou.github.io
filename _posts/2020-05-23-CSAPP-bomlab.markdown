---
layout: post
title:  "CSAPP Bomb Lab Notes"
date:   2020-05-23 11:03:36 +0800
categories: Assembly
typora-root-url: ../../zhufangzhou.github.io
---

This is a notes for bomb lab of CSAPP3e.

# Introduction

Your job for this lab is to defuse your bomb. You will be given a binary file `bomb` and you need to execute it and type 6 **CORRECT** strings to defuse the bomb. Each time you type a **WRONG** string will cause the `bomb` to explode, so you need to be very careful.

You can use `objump -d` command to get the assembly code of the bomb and know how it works.

The **bomb lab** materials can be download from [http://csapp.cs.cmu.edu/3e/labs.html](http://csapp.cs.cmu.edu/3e/labs.html). 



# My Solution Notes

Before dive into the assembly code, we should take a glance at the C code (`bomb.c`) to know how the bomb work from a high-level perspective.

The core code is shown below. We can see that code for each phase is almost similar: 

1. Read a line to get input;
2. Run the phase code to check whether the input is correct;
3. Defuse this phase if the input is right.

We can also know from this code that:

1. 6 phases are solved one by one;
2. The only input of `phase_x` function is the input string pointer.

```c
/* Do all sorts of secret stuff that makes the bomb harder to defuse. */
initialize_bomb();

printf("Welcome to my fiendish little bomb. You have 6 phases with\n");
printf("which to blow yourself up. Have a nice day!\n");

/* Hmm...  Six phases must be more secure than one phase! */
input = read_line();             /* Get input                   */
phase_1(input);                  /* Run the phase               */
phase_defused();                 /* Drat!  They figured it out!
              										* Let me know how they did it. */
printf("Phase 1 defused. How about the next one?\n");

/* The second phase is harder.  No one will ever figure out
 * how to defuse this... */
input = read_line();
phase_2(input);
phase_defused();
printf("That's number 2.  Keep going!\n");

/* I guess this is too easy so far.  Some more complex code will
 * confuse people. */
input = read_line();
phase_3(input);
phase_defused();
printf("Halfway there!\n");

/* Oh yeah?  Well, how good is your math?  Try on this saucy problem! */
input = read_line();
phase_4(input);
phase_defused();
printf("So you got that one.  Try this one.\n");

/* Round and 'round in memory we go, where we stop, the bomb blows! */
input = read_line();
phase_5(input);
phase_defused();
printf("Good work!  On to the next...\n");

/* This phase will never be used, since no one will get past the
 * earlier ones.  But just in case, make this one extra hard. */
input = read_line();
phase_6(input);
phase_defused();

/* Wow, they got it!  But isn't something... missing?  Perhaps
 * something they overlooked?  Mua ha ha ha ha! */
```



Now, we can begin to hack the assembly code. Let' generate it by executing

```shell
objdump -d bomb.c > bomb.asm
```

## Phase 1

First we search for keyword `phase_1`, we can find the code where this function is called.

This short snippet of code is self-explained and I add some comments to some key rows.

```nasm
  ...
  400e19: e8 84 05 00 00        callq  4013a2 <initialize_bomb>
  400e1e: bf 38 23 40 00        mov    $0x402338,%edi
  400e23: e8 e8 fc ff ff        callq  400b10 <puts@plt>
  400e28: bf 78 23 40 00        mov    $0x402378,%edi
  400e2d: e8 de fc ff ff        callq  400b10 <puts@plt>
  400e32: e8 67 06 00 00        callq  40149e <read_line>			; call <read_line> function and the return value (input string pointer) is stored in %rax
  400e37: 48 89 c7              mov    %rax,%rdi					; set input argument of <phase_1> function
  400e3a: e8 a1 00 00 00        callq  400ee0 <phase_1>				; call <phase_1> function
  400e3f: e8 80 07 00 00        callq  4015c4 <phase_defused>		; defuse this phase if the string is correct
  ...
```

Then, we need to find out how the  `<phase_1>` function check our input string.

```nasm
0000000000400ee0 <phase_1>:
; Input
; %rdi : address of input string
  400ee0: 48 83 ec 08           sub    $0x8,%rsp
  400ee4: be 00 24 40 00        mov    $0x402400,%esi             	; $0x402400 is the address of the predefined string, use gdb to read memory untill 0x00 and translate the bytes with ascii
  400ee9: e8 4a 04 00 00        callq  401338 <strings_not_equal>
  400eee: 85 c0                 test   %eax,%eax                  	; check whether %eax is 0. 0 means strings are equal.
  400ef0: 74 05                 je     400ef7 <phase_1+0x17>
  400ef2: e8 43 05 00 00        callq  40143a <explode_bomb>
  400ef7: 48 83 c4 08           add    $0x8,%rsp
  400efb: c3                    retq
```

As the assembly code shows that, it calls `<strings_not_equal>` function with two argument:
1. %rdi: address of input string
2. %rsi: $0x402400, which is the address of a constant string

The return value stores in %eax and the bomb explodes if `%eax != 0`. So, our job is to find out the exactly value stores in address $0x402400.



After entering the `gdb` interactive console, we can get a 4-byte word by typing the following command (`x` in `wx` means force the output in hex format)

```shell
x/wx 0x402400
```

![image-20200526131112465](/assets/2020-05-23-CSAPP-bomlab.assets/image-20200526131112465.png)

Some other useful command are list in the following table

|      Command      |                       Description                        |
| :---------------: | :------------------------------------------------------: |
| x/w   0xbffff890  |   Examine (4-byte) word starting at address 0xbffff890   |
| x/2w   0xbffff890 | Examine two (4-byte) word starting at address 0xbffff890 |
| x/g   0xbffff890  |   Examine (8-byte) word starting at address 0xbffff890   |
| x/2g   0xbffff890 | Examine two (8-byte) word starting at address 0xbffff890 |

Then we can parse every byte with the ASCII table to find the corresponding character until we reach `0x00`, which means the end of a string.



The solution of this phase is:

```
Border relations with Canada have never been better.
```

