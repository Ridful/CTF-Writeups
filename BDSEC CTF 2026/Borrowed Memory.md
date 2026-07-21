# Borrowed Memory — writeup (BDSEC CTF 2026)

**category:** rev
**flag:** `BDSEC{p01nt3rs_l13_bUt_0ffs3ts_r3m3mb3r}`

so this one's called "borrowed memory" and honestly the name is doing a lot of foreshadowing lol. let's just walk through it chill style.

## first look

file's named `borrowed_memory.bdsec` but that extension is a total lie, it's just an ELF binary wearing a costume.

```
file chal
-> ELF 64-bit LSB pie executable, x86-64, stripped
```

stripped, no symbols, so `objdump -d` gives us plt names only. cool cool, we get to read raw asm for a bit. strings dump shows a nice ascii art banner:

```
BORROWED MEMORY
0x???? -> 0x???? -> 0x????
Return what was borrowed.
```

plus a suspicious `rejected` string sitting nearby. classic "give me the right number(s) or get yeeted" setup.

## what it actually wants

reading through the disassembly, the program does two loops back to back:

1. **the read loop** — prompts you with `>` and reads a number (via `strtoul`, base 0, so hex or decimal both work), twelve times in a row. each number just has to land in the range `0x4000–0x47FF` or it insta-rejects. that's it for validation at this stage — it's not checking correctness yet, just "is this a plausible address."
    
2. **the VM loop** — after it's slurped up all 12 numbers, it starts walking through an internal 2048-byte scrambled memory table (built at program startup via a little xorshift-ish PRNG) using an _address_ that starts at some fixed value and evolves round to round based on bytes it reads out of that table. each round it:
    
    - checks that the number **you** supplied for that round equals `current_address + 0x4000`
    - decodes an "opcode" out of the table at that address (4 possible cases)
    - computes the next address from table lookups + a running hash mix
    - ticks a checksum register down by a fixed amount

Basically: your 12 numbers are supposed to be the exact **breadcrumb trail of addresses** the VM is going to walk through its own scrambled memory. You're not solving a puzzle so much as reading the answer key back to it.

The countdown checksum starts at `0xbeef` and drops by `0x111` each round — do the math and it lands exactly on the required target (`0xb223`) after **exactly 12 rounds**. Nice, that matches the 12-number requirement, so we know we're not missing steps.

## the big realization

Here's the fun part: none of the address-chain math depends on what you type. The next address is 100% determined by the table + a few fixed constants baked into the binary. Your input is _only_ ever compared against it — it never influences it.

Which means we don't need to reverse-engineer the whole hash-mixing mess by hand. We just need to **watch the binary compute its own answers** and copy them down.

## getting gdb to do the tedious part

Grabbed gdb (wasn't installed, `apt-get install -y gdb` sorted that out), then:

1. Found the load base with `starti` + `info proc mappings` (PIE, so addresses shift at runtime).
2. Broke right at the setup code, fed it 12 throwaway numbers (all `0x4000`, just to get past the "is it in range" check and reach the VM), and dumped:
    - the initial address (`PC0 = 0x1A4`, found by reading `$esi` right after it gets XORed against a constant)
    - the whole 2048-byte scrambled table (`dump binary memory table.bin $rbx $rbx+0x800`)
3. Set a breakpoint right at the comparison instruction (`cmp r8w, ax`) with a little gdb command list attached to it:

```gdb
break *ADDR_OF_CMP
commands
  silent
  printf "REQ %x\n", $eax & 0xffff
  set $r8 = ($eax & 0xffff)
  continue
end
run < dummy_input.txt
```

Every time that breakpoint fires, gdb prints what value it _wanted_, then patches the register so the check passes anyway, and keeps going. One single run walks the entire 12-round chain and prints out the full answer sequence — no manual re-running needed.

Got this list of required values:

```
41a4 42f0 4143 436c 421d 44a8 40f6 455b 4317 468c 425a 473d
```

## anticlimactic finish

Fed those 12 numbers (converted to decimal) straight into the _actual unmodified_ binary, no gdb, no patching, just genuinely typing the right numbers:

```
16804
17136
16707
17260
16925
17576
16630
17755
17175
18060
16986
18237
```

and it happily decrypted its own flag bytes and printed:

```
[+] BDSEC{p01nt3rs_l13_bUt_0ffs3ts_r3m3mb3r}
```

## takeaways

- "stripped binary with a hand-rolled VM" sounds scarier than it is once you notice the control flow doesn't actually depend on your input — sometimes the fastest path isn't a full static reverse-engineer, it's letting the program tell on itself.
- gdb breakpoint `commands` blocks are severely underrated for this kind of "watch it compute, steal the answer" move.
- flag name checks out — the whole challenge is just pointers wandering around memory that "remembers" its own offsets regardless of what lies you feed it.

