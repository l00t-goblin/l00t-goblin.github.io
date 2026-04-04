---
title: simple_encryptor
description: Writeup for the simple_encryptor reverse-engineering challenge on HackTheBox
created: 2025-09-20
tags: rev, ctf, practice, hackthebox
draft: false
---

# Introduction

---

**Summary**

`simple_encryptor` is a very easy reverse-engineering challenge on HackTheBox. You’re given an ELF binary and an encrypted file, flag.enc. The task is to understand the encryption routine and write a decryptor to recover the flag.

At a high level, the program seeds a pseudo-random number generator (PRNG) with the current Unix epoch. For each byte of the flag, it generates two random numbers and applies bitwise operations. After encryption, it writes the Unix epoch (used as the seed) followed by the ciphertext into flag.enc. Because the seed is prepended to the file, you can reproduce the PRNG stream and decrypt the data.

**Challenge Description**

On our regular checkups of our secret flag storage server we found out that we were hit by ransomware! The original flag data is nowhere to be found, but luckily we not only have the encrypted file but also the encryption program itself.

**Difficulty:** Very Easy

# Challenge

---

## Recon

We start with basic recon on the provided ELF binary.

```bash
$ file ./encrypt
./encrypt: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=0bddc0a794eca6f6e2e9dac0b6190b62f07c4c75, for GNU/Linux 3.2.0, not stripped
```

This tells us we’re dealing with a 64-bit, dynamically linked x86-64 PIE. Dumping symbols in .text shows a single interesting function: main.

```bash
$ objdump -t ./encrypt | grep ".text"
00000000000011a0 l    d  .text  0000000000000000              .text
00000000000011d0 l     F .text  0000000000000000              deregister_tm_clones
0000000000001200 l     F .text  0000000000000000              register_tm_clones
0000000000001240 l     F .text  0000000000000000              __do_global_dtors_aux
0000000000001280 l     F .text  0000000000000000              frame_dummy
00000000000014b0 g     F .text  0000000000000005              __libc_csu_fini
0000000000001440 g     F .text  0000000000000065              __libc_csu_init
00000000000011a0 g     F .text  000000000000002f              _start
0000000000001289 g     F .text  00000000000001b5              main
```

Although this isn’t a pwn challenge, it’s still worth checking binary hardening features. Everything is enabled.

```bash
$ pwn checksec ./encrypt
[*] 'encrypt'
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No

```

A quick strings pass doesn’t reveal anything useful.

```bash
$ strings ./encrypt

...
```

## Reverse-Engineering

### Code Review

Open the binary in Ghidra and navigate to main. A cleaned-up decompilation looks like this:

```c
undefined8 main(void) {
  int rand_one;
  time_t tVar1;
  long in_FS_OFFSET;
  uint curr_time;
  uint rand_two;
  long i;
  FILE *flag_file_handle;
  size_t flag_file_size;
  void *flag_memory;
  FILE *flag_enc_file_handle;
  long canary;

  canary = *(long *)(in_FS_OFFSET + 0x28);
  flag_file_handle = fopen("flag","rb");
  fseek(flag_file_handle,0,2);
  flag_file_size = ftell(flag_file_handle);
  fseek(flag_file_handle,0,0);
  flag_memory = malloc(flag_file_size);
  fread(flag_memory,flag_file_size,1,flag_file_handle);
  fclose(flag_file_handle);
  tVar1 = time((time_t *)0x0);
  curr_time = (uint)tVar1;
  srand(curr_time);
  for (i = 0; i < (long)flag_file_size; i = i + 1) {
    rand_one = rand();
    *(byte *)((long)flag_memory + i) = *(byte *)((long)flag_memory + i) ^ (byte)rand_one;
    rand_two = rand();
    rand_two = rand_two & 7;
    *(byte *)((long)flag_memory + i) =
         *(byte *)((long)flag_memory + i) << (sbyte)rand_two |
         *(byte *)((long)flag_memory + i) >> 8 - (sbyte)rand_two;
  }
  flag_enc_file_handle = fopen("flag.enc","wb");
  fwrite(&curr_time,1,4,flag_enc_file_handle);
  fwrite(flag_memory,1,flag_file_size,flag_enc_file_handle);
  fclose(flag_enc_file_handle);
  if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

First, the program reads flag into memory. It then seeds the PRNG with the current Unix epoch via `srand(time(0))`:

```c
// ...

flag_file_handle = fopen("flag","rb");
fseek(flag_file_handle,0,2);
flag_file_size = ftell(flag_file_handle);
fseek(flag_file_handle,0,0);
flag_memory = malloc(flag_file_size);
fread(flag_memory,flag_file_size,1,flag_file_handle);
fclose(flag_file_handle);
tVar1 = time((time_t *)0x0);
curr_time = (uint)tVar1;
srand(curr_time);

// ...
```

The encryption loop works byte-by-byte. For each byte, it:

1. Generates rand_one and XORs it with the byte.
2. Generates rand_two, masks it with 0x07, and rotates the byte left by that amount (ROL8).

```c
// ...

for (i = 0; i < (long)flag_file_size; i = i + 1) {
rand_one = rand();
*(byte *)((long)flag_memory + i) = *(byte *)((long)flag_memory + i) ^ (byte)rand_one;
rand_two = rand();
rand_two = rand_two & 7;
*(byte *)((long)flag_memory + i) =
        *(byte *)((long)flag_memory + i) << (sbyte)rand_two |
        *(byte *)((long)flag_memory + i) >> 8 - (sbyte)rand_two;
}

// ...
```

Finally, it writes flag.enc as: the 4-byte epoch seed followed by the ciphertext.

```c
// ...

flag_enc_file_handle = fopen("flag.enc","wb");
fwrite(&curr_time,1,4,flag_enc_file_handle);
fwrite(flag_memory,1,flag_file_size,flag_enc_file_handle);
fclose(flag_enc_file_handle);
if (canary != *(long *)(in_FS_OFFSET + 0x28)) {
                /* WARNING: Subroutine does not return */
__stack_chk_fail();
}
return 0;

// ...
```

The resulting file layout is:

<pre class="ascii-art">


        0                      3                      N
        +----------------------+------------------------+
        | Unix Epoch Seed      | Encrypted Flag         |
        +----------------------+------------------------+

</pre>

## Solution

To decrypt, read the first four bytes to recover the epoch, seed srand with it, and then invert the per-byte operations in order (reverse the rotate, then reverse the XOR).

Extracting the seed and setting up the PRNG:

```c
// ...

FILE* f = fopen(ENC_FLAG_FPATH, "rb");
if (f == NULL) {
    fprintf(stderr, "Could not open %s\n", ENC_FLAG_FPATH);
    return 1;
}

fseek(f, 0, SEEK_END);
size_t size = ftell(f);
fseek(f, 0, 0);
printf("%s size: %zu\n", ENC_FLAG_FPATH, size);

uint8_t* flag = (uint8_t*)malloc(size);
if (flag == NULL) {
    fprintf(stderr, "Could not allocate %zu bytes for flag\n", size);
    return 1;
}

fread(flag, 1, size, f);

uint32_t seed;
memcpy(&seed, flag, sizeof(uint32_t));
printf("Epoch seed: %d\n", seed);
srand(seed);

for (int i = 0; i < 4; i++) {
    *flag++;
}

// ...
```

Now, let’s reason about the rotation. The encryption uses an 8-bit rotate-left (ROL):

> ROL8(x, n) = ((x << n) | (x >> (8 − n))) & 0xFF

(For example, with x = 0xFC and n = 0x04: ROL8(0xFC, 0x04) = (0xC0 | 0x0F) = 0xCF.)

To invert that, we rotate right by the same amount:

> ROR8(x, n) = ((x >> n) | (x << (8 − n))) & 0xFF

We must undo the steps in reverse order: first the rotation, then the XOR.

```c
// ...

for (int i = 0; i < FLAG_SIZE; i++) {
    int r1 = rand();
    int r2 = rand() & 7;

    flag[i] = (uint8_t)((flag[i] >> r2) | (flag[i] << (8 - r2)));
    flag[i] = flag[i] ^ r1;
}

// ...
```

<details>
    <summary>Full solve.c</summary>

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

#define ENC_FLAG_FPATH  "flag.enc"
#define FLAG_SIZE       28

int main(int argc, char** argv) {
    FILE* f = fopen(ENC_FLAG_FPATH, "rb");
    if (f == NULL) {
        fprintf(stderr, "Could not open %s\n", ENC_FLAG_FPATH);
        return 1;
    }

    fseek(f, 0, SEEK_END);
    size_t size = ftell(f);
    fseek(f, 0, 0);
    printf("%s size: %zu\n", ENC_FLAG_FPATH, size);

    uint8_t* flag = (uint8_t*)malloc(size);
    if (flag == NULL) {
        fprintf(stderr, "Could not allocate %zu bytes for flag\n", size);
        fclose(f);
        return 1;
    }

    fread(flag, 1, size, f);

    uint32_t seed;
    memcpy(&seed, flag, sizeof(uint32_t));
    printf("Epoch seed: %d\n", seed);
    srand(seed);

    // Skip the 4-byte seed at the start of the buffer
    for (int i = 0; i < 4; i++) {
        *flag++;
    }

    for (int i = 0; i < FLAG_SIZE; i++) {
        int r1 = rand();
        int r2 = rand() & 7;

        flag[i] = (uint8_t)((flag[i] >> r2) | (flag[i] << (8 - r2)));
        flag[i] = flag[i] ^ r1;
    }

    printf("flag: %s\n", flag);

    fclose(f);
    free(flag);
    return 0;
}

```

</details>

Compile and run to recover the flag.

# References

---

- [felixcloutier.com - rcl/rcr/rol/ror](https://www.felixcloutier.com/x86/rcl:rcr:rol:ror)
- [stackoverflow.com - Why srand(time(NULL)) is a bad seed](https://stackoverflow.com/questions/30145715/why-srandtime-is-a-bad-seed)
