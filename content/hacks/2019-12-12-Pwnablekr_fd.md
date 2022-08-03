---
title: Pwnable.kr fd
author: Marius Kimmina
date: 2019-11-24 14:10:00 +0800
categories: [CTF, Binary Exploitation]
tags: [File descriptor]
published: true
---

## 0x0 Introduction
Pwnable.kr is a so called "wargame site" which offers a bunch of 'pwn' challenges for system exploitation, 
in my Blog I will only cover the *easy* ones as we do not have the permission to post solutions for the more advanced challenges.  
If you found these posts and you are intrested in learning more about exploitation I highly recommend that you try to solve these challenges on your own first.

## 0x1 Challenge description
Mommy! what is a file descriptor in Linux?

## 0x2 Challenge and solution

3 Files were provided for this challenge, a flag file which we can't read, an executeable *fd* and fd.c which is the source code for fd.

```
fd@pwnable:~$ tree
.
├── fd
├── fd.c
└── flag
```

Upon executing fd, we are being told to enter a number

```
fd@pwnable:~$ ./fd
pass argv[1] a number
```

and if we pass it some random number it responds with a good advice 

```
fd@pwnable:~$ ./fd 5
learn about Linux file IO
```

Now, lets take a look at the provided source code for this program.

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
                printf("good job :)\n");
                system("/bin/cat flag");
                exit(0);
        }
        printf("learn about Linux file IO\n");
        return 0;

}
```
The Key here is that the fd variable is the first argument to the "read" function, according to the documentation
the first argument to read defines the file descriptor that is being read from.   
`ssize_t read(int fildes, void *buf, size_t nbyte);`  
And the description reads: 
```
The read() function shall attempt to read nbyte bytes from the file associated with the open file descriptor, fildes, into the buffer pointed to by buf.
```
Now Linux filedescriptors are defined as:
* 0: stdin
* 1: stdout
* 2: stderr

This means that if we get fd to be equal to 0, the program will take input from stdin and we can then give it the "LETMEWIN" keyword.  
All we need to do is to convert 0x1234 to dezimal, you can use python for the conversion:  

```
>>> int("0x1234", 16)
4660
```

With that out of the way we can simply parse 4660 as the first argument to `fd`.

```
fd@pwnable:~$ ./fd 4660
LETMEWIN
good job :)
mommy! I think I know what a file descriptor is!!
```

On a side node, STDOUT would also work in this case as they both refer to the terminal.

## 0x3 conclusion
After this challenge you should have a better understanding of linux filedescriptors. I hope you enjoyed the writeup and that you were able to solve this own your own before reading it.
