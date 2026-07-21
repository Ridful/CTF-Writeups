
# Night Shift — writeup

so we get handed a file called `night_shift.bdsec` and honestly the extension's a total lie, `file` says it's just an ELF binary wearing a costume. cool, whatever, chmod +x it and we're rolling.

## poking at it

first move is always `strings`, just see what falls out:

```
shift code>
Eight assignments remain.
The shift report was rejected.
The morning report has been approved.
```

so it wants some kind of code, 8 things, and it's either gonna yell at us or give us the good stuff. ran it once with garbage input just to see the shape of things — asks for a line, we type stuff, it judges us instantly. fair.

## digging into the guts

it's stripped so no nice function names, just vibes and addresses. threw it into `objdump` and started reading.

turns out the input format is: 8 numbers separated by spaces, each one has to be `0`–`4`, parsed with `strtok_r` + `strtoul`. anything else and it bails straight to the rejection message. simple enough gate.

past that gate it spins up **5 threads**, and they're all doing this synchronized dance with a mutex + condvar — basically taking turns so the 8 positions in your code get processed in strict order, one at a time, no cutting in line. each thread's ID matches one of your digit values, so really "thread 3" = "whoever's handling the digit-3 spots."

for each position it mixes some state (4 accumulator words that start life as the eternally funny `0xC001D00D` and `0x0BADF00D`) through a hash function that's basically [lowbias32](https://nullprogram.com/blog/2018/07/31/) — that xorshift-multiply integer hash thing, fully invertible, very tidy. digit value decides _which_ mixing branch runs (there's a jump table for it). there's also a running FNV-ish accumulator getting updated alongside it.

after all 8 positions are chewed through, it checks the final state against four hardcoded 64-bit/32-bit constants. found these sitting right in the disassembly:

```
cmp QWORD PTR [rsp+0xe8], 0x75a2cc729c8a97dc
cmp QWORD PTR [rsp+0xf0], 0x4969e73d1d87ef0f
cmp DWORD PTR [rsp+0x118], 0x4455cee8
```

match all three (plus a sanity count check) → you get let into the good branch that prints the flag. miss any → straight back to rejected.

## so how do we actually crack it

honest answer: **don't bother reverse engineering the hash by hand to invert it, just brute force the input space.** it's only digits 0–4, eight slots, so `5^8 = 390,625` combos. nothing.

wrote a python sim of the whole mixing pipeline (all 5 branches, the lowbias32 calls, the FNV mix, all of it) and just churned through every combo checking which one hits all three target constants:

```python
for digits in itertools.product(range(5), repeat=8):
    s, posArr, fnv = process(digits)
    if fnv == TARGET_FNV and s == [TARGET_S0, TARGET_S1, TARGET_S2, TARGET_S3]:
        print("found it:", digits)
```

got exactly **one match**: `(2, 0, 4, 1, 3, 0, 2, 4)`.

## the part where I almost shot myself in the foot

so there's _also_ a decode step — once you pass the check, it prints 36 bytes, each one XORed against a keystream byte generated from another lowbias32-based mix that depends on your digit code and the position. reasonable move: reimplement that too in python and decode the flag offline without even touching the binary again.

except my python reimplementation of _that_ part had a bug in it somewhere (some bit of the position-table indexing or rotation amount got flipped, never fully chased it down) and it spat out a pile of non-printable garbage instead of a flag. classic.

but here's the thing — I didn't actually need to reverse-engineer the decode step at all. the digit code itself was already confirmed correct because it satisfied the _exact_ hardcoded comparisons the binary itself uses to decide pass/fail. so instead of debugging my own broken decode logic, just... let the binary do what it's already good at:

```
$ echo "2 0 4 1 3 0 2 4" | ./night_shift

========================================
              NIGHT SHIFT
========================================

The building is closed.
Eight assignments remain.

shift code> The morning report has been approved.
BDSEC{0rd3r_h1d3s_b3tw33n_th3_l1n3s}
```

approved, flag printed, done. moral of the story: if the _checker_ already told you the input's right, you don't need to also independently reinvent the _decryptor_ — just feed the confirmed-correct input back into the real thing and let it show its own work.

## flag

```
BDSEC{0rd3r_h1d3s_b3tw33n_th3_l1n3s}
```

pretty fitting name for a challenge that's basically "figure out which order the threads eat your digits in."
