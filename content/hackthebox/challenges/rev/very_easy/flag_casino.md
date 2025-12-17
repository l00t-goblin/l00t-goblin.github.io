---
title: flag_casino
description: Writeup for the flag_casino challenge on HackTheBox
created: 2025-12-15
tags: rev, ctf, practice, hackthebox  
draft: false
---

# Challenge Info

**Challenge Description**

> The team stumbles into a long-abandoned casino. As you enter, the lights and music whir to life, and a staff of robots begin moving around and offering games, while skeletons of prewar patrons are slumped at slot machines. A robotic dealer waves you over and promises great wealth if you can win - can you beat the house and gather funds for the mission?

**Challenge Category**

> rev

**Challenge Difficulty**

> Very Easy

**Challenge Summary**

> Flag Casino was a very easy reverse-engineering challenge on HackTheBox. The premise revolves around abusing the predictability of a pseudo-random number generator (PRNG). The program allows the user to input a single character; that character is then used as the seed for the PRNG. After seeding, the program calls `rand()` exactly once and compares the returned value to an entry in a list of 32-bit integers. If the generated number matches the expected value, the user is allowed to continue. If not, the program exits.

> Initially, I solved the challenge with a brute-force approach. Since the input is exactly one character, the search space is only 256 values, meaning ~1/100 printable characters will be correct at each position. I wrote a small Python script that attempts each printable character, checks for success, and appends the correct character to the flag before restarting the process.

> However, after solving the challenge, I realized there is a much more elegant method. Because the expected outputs are known and the seed space is small, we can precompute the first `rand()` value for all possible seeds. Then we can simply map each desired output to the correct seed. The intended solution uses Python’s ctypes module to call the real `srand()` and `rand()` from libc, which made for a nice opportunity to learn something new.

# Enumeration

As usual, we begin by examining the binary:

```bash
$ file casino 
casino: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=7618b017ef4299337610a90a0a6ccb7f9efc44a4, for GNU/Linux 3.2.0, not stripped
```

Next, we check for interesting symbols:

```bash
$ objdump -t ./casino | grep -E ".text|.data|.bss"
00000000000010a0 l    d  .text  0000000000000000              .text
0000000000002000 l    d  .rodata        0000000000000000              .rodata
0000000000004060 l    d  .data  0000000000000000              .data
00000000000040f8 l    d  .bss   0000000000000000              .bss
00000000000010d0 l     F .text  0000000000000000              deregister_tm_clones
0000000000001100 l     F .text  0000000000000000              register_tm_clones
0000000000001140 l     F .text  0000000000000000              __do_global_dtors_aux
00000000000040f8 l     O .bss   0000000000000001              completed.0
0000000000001180 l     F .text  0000000000000000              frame_dummy
00000000000012e0 g     F .text  0000000000000001              __libc_csu_fini
0000000000004060  w      .data  0000000000000000              data_start
00000000000040f8 g       .data  0000000000000000              _edata
0000000000002020 g     O .rodata        0000000000000091              banner
0000000000004060 g       .data  0000000000000000              __data_start
0000000000004068 g     O .data  0000000000000000              .hidden __dso_handle
0000000000002000 g     O .rodata        0000000000000004              _IO_stdin_used
0000000000001280 g     F .text  000000000000005d              __libc_csu_init
0000000000004100 g       .bss   0000000000000000              _end
00000000000010a0 g     F .text  000000000000002b              _start
00000000000040f8 g       .bss   0000000000000000              __bss_start
0000000000001185 g     F .text  00000000000000f2              main
0000000000004080 g     O .data  0000000000000078              check
00000000000040f8 g     O .data  0000000000000000              .hidden __TMC_END__

```

Among the usual sections, we see an interesting symbol in `.data`: `check`

Now that we have a feel for what's in the binary, we can dive into some disassembly and code review.

# Code Review

Using IDA, we navigate to the `main()` function. The decompiler produces:

```c
int __fastcall main(int argc, const char **argv, const char **envp)
{
  char userChar; // [rsp+Bh] [rbp-5h] BYREF
  unsigned int i; // [rsp+Ch] [rbp-4h]

  puts("[ ** WELCOME TO ROBO CASINO **]");
  puts(
    "     ,     ,\n"
    "    (\\____/)\n"
    "     (_oo_)\n"
    "       (O)\n"
    "     __||__    \\)\n"
    "  []/______\\[] /\n"
    "  / \\______/ \\/\n"
    " /    /__\\\n"
    "(\\   /____\\\n"
    "---------------------");
  puts("[*** PLEASE PLACE YOUR BETS ***]");
  for ( i = 0; i <= 29; ++i )
  {
    printf("> ");
    if ( (unsigned int)__isoc99_scanf(" %c", &userChar) != 1 )
      exit(-1);
    srand(userChar);
    if ( rand() != check[i] )
    {
      puts("[ * INCORRECT * ]");
      puts("[ *** ACTIVATING SECURITY SYSTEM - PLEASE VACATE *** ]");
      exit(-2);
    }
    puts("[ * CORRECT *]");
  }
  puts("[ ** HOUSE BALANCE $0 - PLEASE COME BACK LATER ** ]");
  return 0;
}
```

The logic is simple:

1. Read a single character from the user
2. Use that character as the seed for `srand()`
3. Generate one random number with `rand()`
4. Compare it to `check[i]`
5. Continue if equal, exit if not

Let’s inspect the check array:
```
.data:0000000000004080 check           dd 244B28BEh, 0AF77805h, 110DFC17h, 7AFC3A1h, 6AFEC533h
.data:0000000000004094                 dd 4ED659A2h, 33C5D4B0h, 286582B8h, 43383720h, 55A14FCh
.data:00000000000040A8                 dd 19195F9Fh, 43383720h, 19195F9Fh, 747C9C5Eh, 0F3DA237h
.data:00000000000040BC                 dd 615AB299h, 6AFEC533h, 43383720h, 0F3DA237h, 6AFEC533h
.data:00000000000040D0                 dd 615AB299h, 286582B8h, 55A14FCh, 3AE44994h, 6D7DFE9h
.data:00000000000040E4                 dd 4ED659A2h, 0CCD4ACDh, 57D8ED64h, 615AB299h, 22E9BC2Ah
.data:00000000000040E4 _data           ends
```

It contains exactly 30 integers, matching the loop count in `main()`.

# First Approach: Inefficent Bruteforce Using pwntools

Since `srand()` takes only a single byte in this challenge (the user’s input character), there are only 256 possible seeds. Additionally, the characters making up the flag are almost certainly printable, further reducing the practical search space.

This makes brute forcing the flag easy:

Try every printable character, keep the one that succeeds, and repeat.

Below is the script I used:

<details>
    <summary>solve.py</summary>

```python
#!/usr/bin/env python3

import string

from pwn import *
from time import sleep

def main()-> None:
    flag: str = ""

    context.log_level = "error"

    while len(flag) != 30:
        print(f"Current flag: {flag}")

        # Guess every character
        for c in string.printable:
            p = process("./casino")
            try:
                p.recvuntil(b"[*** PLEASE PLACE YOUR BETS ***]\n")

                # Replay known prefix
                for ch in flag:
                    p.sendline(ch.encode())
                    resp = p.recvline()
                    if b"INCORRECT" in resp:
                        raise Exception("Incorrect character in flag ???")

                # Attempt next guess
                p.sendline(c.encode())
                resp = p.recvline()
                
                if b"INCORRECT" in resp:
                    continue

                flag += c
                p.close()
                sleep(0.1)
                break

            finally:
                # Ensure fds are released every iteration
                try:
                    p.close()
                except Exception:
                    pass
        
    print(f"Flag: {flag}")

if __name__ == "__main__":
    main()
```

</details>

This works reliably and finds the flag, but it’s definitely not the most elegant method.

# Second Approach: Preprocessing Using `ctypes` Module

After finishing the challenge, I read the intended solution and found it much cleaner. The key insight is:

- The search space for seeds is tiny (0–255).
- The list of expected PRNG outputs is known.
- Therefore, we can precompute the mapping from rand() output -> seed.

Using Python’s `ctypes` module, we can call the real glibc implementation of `srand()` and `rand()` directly.

Here is the intended solution:

<details>
    <summary>solve.py</summary>

```python
#!/usr/bin/env python3

import ctypes
from pwn import *

def main()-> None:
    libc = ctypes.CDLL("libc.so.6")
    mapping: dict = {}

    # Create a mapping. The mapping contains all the potential number
    # we might receive from rand()
    for i in range(256):
        libc.srand(i)
        mapping[libc.rand()] = chr(i)

    # Get the list of values in 'check'
    flag: str = ""
    e = ELF("./casino", checksec=False)
    for i in range(30):
        # Extract unsigned 32-bit integer
        val = e.u32(e.sym["check"] + i * 4)
        flag += mapping[val]

    print(flag)
    
if __name__ == "__main__":
    main()
```

</details>

This solution is far more efficient, and elegant.