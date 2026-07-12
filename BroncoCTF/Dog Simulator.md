# Dog Simulator

**Category:** Reversing · **Author:** dot.t

> _If you can survive 6 dog-days of walkies, tricks, and zoomies, you might uncover a secret (if you can stick to the right routine in the right order)!_

**Flag:** `bronco{mans_best_friend}`

---

## the vibe

You're a dog. For six days you bark, fetch, sit, eat, do zoomies, and occasionally attempt human speech. Do the right things in the right order and the game coughs up a flag. Do the wrong things and your owner passive-aggressively tells you the "timing was off." Classic.

The intended solve is to actually _figure out the routine_. I did not do that. I let the math do the walkies for me. More on that below.

## first look

```
$ file dog-sim-mac
dog-sim-mac: Mach-O 64-bit arm64 executable
```

An arm64 Mach-O, and I'm on an x86 Linux box, so no just-run-it. Fine. Static it is. It's tiny (~3.7 KB of actual code), stripped of function names but not much else.

`strings` immediately spills the whole mood:

```
Day %d: Owner says: "%s"
1) Bark (+%d)
2) Fetch (+%d)
3) Sit (+%d)
4) Eat (+%d)
5) Zoomies (%d)
6) Speak (type a command)
Owner: the routine felt right, but the timing was off.
Owner: he keeps repeating what he heard... but it's not quite right.
Who's a good reverse engineer?
```

No literal flag in `strings`, though. That "Who's a good reverse engineer?" line is the taunt right before the payoff, so the flag is clearly _computed_, not stored.

## tooling

No radare, no working objdump for Mach-O on this box, so I rolled my own with Python:

- **LIEF** to parse the Mach-O — grab sections, and (the useful bit) walk the chained-fixup bindings to map each `__got` slot to its imported symbol.
- **Capstone** to disassemble `__text`, with a little post-pass that resolves `adrp`+`add` pairs into the string they point at and rewrites `bl` targets into `-> _printf` / `-> _fgets` / etc.

That turned raw arm64 into something readable, e.g.:

```
0x100000678: add x0, x0, #0x674  ; "(End of day %d) Score=%d Bond=%d Energy=%d Mood=%s\n"
0x10000067c: bl  #0x1000013e4     ; -> _printf
```

Much nicer.

## how the game actually works

Main loop runs 6 days. Each day = **one** action, then "end of day," then either loop or (on day 6) hit the finale. Every action does two things:

1. Bumps a pile of state — score, energy (`w26`), a couple of "combo" trackers (`w22`, `w24`), some per-action counters on the stack.
2. Folds itself into a **rolling hash** in `w23` using an xorshift-multiply mixer with action-specific constants.

`Speak` is special: it reads a word, strips it to lowercase letters, and FNV-1a-32 hashes it (the giveaway constants `0x811c9dc5` basis and `0x01000193` prime). There's a check that one spoken word must hash to `0x9f58d866`, another must literally be `"gremlin"` (what the owner keeps calling you — rude but fair), and the two speaks together have to total 19 letters.

## the finale (where the loot is)

At the end of day 6 there's a glorious wall of chained `ccmp` gates. All of these have to be true at once:

- exact per-action counts (one sit, one fetch, one eat, no zoomies, two speaks…)
- `w25 == 24`, `w26 > 20`, `w22 > 2`
- the FNV / "gremlin" / 19-letter speak conditions
- two final hash equalities: `w23 == 0xf5d38524` and `[sp+0x60] == 0x740a8a98`

Miss any and you get a themed "so close" message. Nail all of them and it decrypts the flag.

Here's the important part: **the flag bytes aren't in the binary.** They come out of a NEON keystream (a vectorized murmur/xorshift thing) XORed against ciphertext constants in `__const`. And that whole keystream is seeded from _just three numbers_ — the final `w22`, `w25`, `w26`:

```
seed = ((w25*31) ^ (w22*0x13579bdf) ^ (w26*13)) ^ 0x510211db
seed = murmur3_fmix32(seed)
```

The giant pile of gates only decides **whether** you get to see the flag. The **value** of the flag is purely `f(w22, w25, w26)`.

## the shortcut

So… I don't actually need to solve the routine, guess the magic word, or satisfy the hash checks. I just need the three final numbers that make the keystream decrypt to real text.

Plan:

1. Spin up **Unicorn** as an arm64 CPU.
2. Map the binary's sections at their real addresses so the `__const` constants are where the NEON code expects them.
3. Jump straight into the flag-gen block (`0x100000ee8` → `0x1000011e0`), skipping the entire game.
4. Set `w22/w25/w26` to candidate values (and `w28 = 0x846ca68b`, the murmur multiplier that's live at that point).
5. Read the 24-byte buffer it builds at `x29-0xe0`.

Then sweep small ranges for those three registers and look for output that starts with `bron`.

```python
def run(w22, w25, w26):
    uc.reg_write(UC_ARM64_REG_SP, sp)
    uc.reg_write(UC_ARM64_REG_X29, fp)
    uc.reg_write(UC_ARM64_REG_W22, w22)
    uc.reg_write(UC_ARM64_REG_W25, w25)
    uc.reg_write(UC_ARM64_REG_W26, w26)
    uc.reg_write(UC_ARM64_REG_W28, 0x846ca68b)  # murmur multiplier
    uc.emu_start(0x100000ee8, 0x1000011e0, count=3000)
    return bytes(uc.mem_read(fp - 0xe0, 24))

for w22 in range(40):
    for w25 in range(64):
        for w26 in range(64):
            out = run(w22, w25, w26)
            if b'bron' in out:
                print(w22, w25, w26, out)
```

Because the flag is exactly 24 bytes, it's `bronco{` (7) + 16 chars + `}` — fills the buffer perfectly, no null terminator even needed inside.

The sweep lands almost instantly:

```
w22=0 w25=20 w26=2 : b'bronco{mans_best_friend}'
```

(A second `(w22,w25,w26)` combo pops out too — different inputs, same seed after the pre-mix collapses them, so same flag. Not a bug, just fmix being fmix.)

## flag

```
bronco{mans_best_friend}
```

## takeaways

- When a flag is _computed_, find the seed, not the string. The author bolted a dozen anti-guess gates onto this thing, but they were all guarding the door — none of them changed what was behind it.
- Emulating a single hot block with Unicorn is criminally underused. I skipped 99% of the binary (and all six days of being a dog) and ran only the ~120 instructions that mattered.
- LIEF + Capstone + a tiny annotation pass gets you a very readable disassembler when your usual tools don't speak the target's file format.

Good boy. 🐕
