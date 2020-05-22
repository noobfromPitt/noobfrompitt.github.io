---
title: "Protostar stack0 walkthrough"
date: 2020-05-21
tags: [ctf, protostar, stack0, walkthrough]
header:
  image: "/images/protostar/protostar.png"
---

## Protostar exercises - [stack0](https://exploit-exercises.lains.space/protostar/stack0/)

#### About
This level introduces the concept that memory can be accessed outside of its allocated region, how the stack variables are laid out, and that modifying outside of the allocated memory can modify program execution.

This level is at /opt/protostar/bin/stack0

#### Source code:
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  modified = 0;
  gets(buffer);

  if(modified != 0) {
      printf("you have changed the 'modified' variable\n");
  } else {
      printf("Try again?\n");
  }
}
```

#### Procedure

In the program, input is loaded into buffer using `gets` function. Since `gets` is not a safe function, we can use it to overwrite the memory where 'modified' variable is stored.
We can solve this if we can change the 'modified' variable from 0

##### Vulnerability:

`gets` is vulnerable as it is impossible to tell how many characters gets() will reaed. it will continue to store the characters past the end of buffer.
`fgets` needs to be used instead.

##### Disassembly:

![disassembly]({{site.url}}{{site.baseurl}}/images/protostar/stack0/disassembly.png)

Lets put breakpoints before and after `gets` call to see what registers are changed
```
break *0x0804840c
break *0x08048411
```
Lets also define some hooks to print at breakpoints
```
define hook-stop
>info registers
>x/24wx $esp
>x/2i $eip
>end
```
From disassembly, we can see [esp+0x5c] will store the value of 'modified' variable. So our aim is to overwrite this address

Now lets run and give the input as 'AAAAAAAA' and see what registers are writter with 0x41's

![AAAAA]({{site.url}}{{site.baseurl}}/images/protostar/stack0/AAAAA.png)

Looks like the registers 0xbffff76c and 0xbffff780 are overwrittern
We can check the 'modified' value using `x/x $esp+0x5c` which is 0x00000000, which is loacated at address 0xbffff7ac

To overwrite this address, we need to write from 0xbffff76c to 0xbffff7ac + 0x8 (0xbffff7b0), that is 0x44 (or 68) characters - AAAAAAAABBBBBBBBCCCCCCCCDDDDDDDDEEEEEEEEFFFFFFFFGGGGGGGGHHHHHHHHIIII

![done]({{site.url}}{{site.baseurl}}/images/protostar/stack0/done.png)

Now, we can see the register esp+0x5c is overwritten with 0x49494949 (IIII) and the variable is changed.

We can verify the input by running stack0 without gdb

![verify]({{site.url}}{{site.baseurl}}/images/protostar/stack0/verify.png)
