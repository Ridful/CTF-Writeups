
# Crack Me Vault — writeup

so we get a file called `crack_me_vault.bdsec`. weird extension but whatever, `file` says it's just a normal stripped ELF binary wearing a costume. cool, let's just run it.

```
$ file crack_me_vault.bdsec
crack_me_vault.bdsec: ELF 64-bit LSB pie executable, x86-64, stripped
```

yep. rename it, chmod +x, run it:

```
Enter the flag:
```

classic. ascii art vault, says "THE BYTECODE VAULT" up top which is a nice lil hint that something VM-ish is going on inside.

## poking around

`strings` doesn't give us the flag (never does, but gotta check). so it's `objdump -d` time. binary's stripped so no function names, just one big blob at `.text` starting `0x10c0`. not too scary though, it's short.

first thing it does: read a line with `fgets`, trim the newline with `strcspn`. normal stuff.

then things get spicy. there's a loop that walks over 4 bytes from a table in `.rodata` at `0x22f2`, XORs each one against a rolling key that starts at `0xa5` and goes up by `0x11` every step, and then branches on the result:

- `0x11` → "check length == 50, AND that into a global success flag"
- `0x37` → "build a lookup table out of the user's input"
- `0x6b` → "compare that table against ANOTHER encrypted table, AND results into the success flag"
- `0xe0` → "if the success flag is still true, print Access granted"

so basically the actual _opcode program_ the vault runs is itself XOR-encrypted, and decoding it tells you the vault only really does 4 things: check length, build a table from your input, compare it, print result. kinda cute — "bytecode VM" is basically just a dressed-up dispatch table, no actual interpreter loop indirection or anything scarier than that.

## the real meat: two loops

**Loop A (opcode `0x37`)** — builds a 50-byte table from your input. For each input position `k` (0 to 49):

```
r8_byte   = (0x41 + 0x1d*k) & 0xff        # rolling xor key
shift     = (k % 7) + 1                    # rotate amount
rotated   = rol8(input[k] ^ r8_byte, shift)
A         = ((11*k) & 0xff) ^ 0x17
table[perm(k)] = (A + rotated) & 0xff      # perm(k) = (17*k) mod 50
```

**Loop B (opcode `0x6b`)** — decrypts a second embedded 50-byte array (sitting right before the opcode table, at `0x22c0`) with its own rolling XOR key, and compares it against `table[perm(j)]` for `j = 0..49`, using the _same_ permutation `perm(j) = (17*j) mod 50`.

Because both loops use the identical permutation to read/write the table, position `k` in your input maps 1:1 to position `k` in the encrypted array. No cross-character dependencies, no real "bytecode interpretation" of your input — it's just a per-character substitution cipher wearing a trench coat. Which means: no need to brute force or symbolically execute anything, just algebra it.

## flipping it around

Every step above is invertible, so for each `k`:

```python
target    = enc[k] ^ key(k)                 # undo loop B's xor
A_low     = ((11*k) & 0xff) ^ 0x17
rotated   = (target - A_low) & 0xff         # undo the add
input[k]  = ror8(rotated, shift) ^ r8_byte   # undo the rotate + xor
```

threw that in a quick python script, ran it over all 50 positions, and out popped:

```
BDSEC{c0nTr0L_fl0w_1s_4_l13_bUt_bYt3c0d3_d03s_n0t}
```

which, love the flag text btw, very on-brand for a challenge that's all about the fact that "bytecode dispatch" _feels_ like real control-flow obfuscation but is actually just linear and solvable if you sit down and read it.

didn't even have to guess — ran the candidate flag straight through the actual binary locally first and got:

```
[+] Access granted.
```

before touching the real submission box. solid since we only get 10 attempts on the real thing.

## tl;dr

1. binary decrypts its own "opcode program" with a rolling XOR, revealing it's just: check len 50 → build table from input → compare table to second encrypted blob → print result.
2. both the table-build and table-compare use the same `(17*k) mod 50` permutation, so it collapses into 50 independent per-character equations.
3. algebra > guessing. reverse each op, get the flag, verify locally, submit once.

**Flag:** `BDSEC{c0nTr0L_fl0w_1s_4_l13_bUt_bYt3c0d3_d03s_n0t}`

