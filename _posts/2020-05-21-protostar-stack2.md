---
title: "Protostar stack2 walkthrough"
date: 2020-05-21
tags: [ctf, protostar, stack2, walkthrough]
header:
  image: "/images/protostar/protostar.png"
---
  
## Protostar exercises - [stack2](https://exploit-exercises.lains.space/protostar/stack2/)

#### About
Stack2 looks at environment variables, and how they can be set.

This level is at /opt/protostar/bin/stack2

#### Source code:
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```

#### Procedure:

Similar to the previous one, the value of 'modified' variable needs to be changed to a specific value. Again, there is a `strcpy()` being used, but now its copying an environment variable GREENIE. 

We need to create a GREENIE environment variable and overwrite 'modified' variable with it.

###### Vulnerability:

`strcpy` is used to copy a string. if the destination string of a `strcpy()` is not large enough, then buffer overflow happens. We can use this to overflow the 64 bit buffer and write into the memory address where 'modified' is stored.

###### Disassembly:

![disassembly]({{site.url}}{{site.baseurl}}/images/protostar/stack2/disassemble.png)

We can see that 'modified' is assigned an address $esp+0x58. Lets setup breakpoints before and after `strcpy()` to see which registers are changing

Lets create GREENIE variable. `export GREENIE=AAAAAAAA`  and run the program.

```
break *0x080484df
break *0x080484e4
define hook-stop
>info registers
>x/24wx $esp
>x/2i $eip
>end
```
![firstrun]({{site.url}}{{site.baseurl}}/images/protostar/stack2/firstrun.png)

We can see that registers 0xbffff748 and 0xbffff74c are written with 0x41's.
Also 'modified' is stored at 0xbffff788 ($esp+0x58) and has value of 0x00000000

To overwrite it, we need to fill the addresses from 0xbffff748 to 0xbffff788. That is 40+4 words or 68 bytes.

Lets change the GREENIE value `export GREENIE=AAAAAAAABBBBBBBBCCCCCCCCDDDDDDDDEEEEEEEEFFFFFFFFGGGGGGGGHHHHHHHHIIII`

![secondrun]({{site.url}}{{site.baseurl}}/images/protostar/stack2/second.png)

Now, we can see that $esp+0x58 (0xbffff748) is changed to 0x49494949. Lets change it to the desired value 0x0d0a0d0a, which is '\r\n\r\n'

Since the machine formats little endian, the required string is AAAAAAAABBBBBBBBCCCCCCCCDDDDDDDDEEEEEEEEFFFFFFFFGGGGGGGGHHHHHHH\n\r\n\r

![done]({{site.url}}{{site.baseurl}}/images/protostar/stack2/done.png)

Now, the varialbe is changed to the desired value and program executes without error.
