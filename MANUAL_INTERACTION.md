# Tosh Protocol — Manual Interaction Guide

**Version:** v5.1 — BNB Smart Chain / PancakeSwap Infinity
**Audience:** Users and power users operating **without** the Tosh frontend
**Status page:** <https://jayoo101.github.io/tosh-status/> — §6b of the
incident playbook links here from there during a frontend outage

> This guide lives in the status-page repository rather than the protocol
> repository, so that it stays readable in the situation it was written for. It
> used to sit in a private repository, which meant the outage playbook pointed
> users at a URL that returned 404 to them. The operator-side companions
> (`INCIDENT_RESPONSE.md`, `SECURITY_AUDIT.md`) are referenced by name below
> but are not public; nothing in this file depends on reading them.

> **Why this exists.** The Tosh contracts do not depend on the Tosh website.
> If the frontend is down, geoblocked, or you simply do not trust it, every
> user action below is reachable directly from the chain with `cast`.
>
> **The one that matters: `refund()` needs nothing from us.** No signature, no
> API, no website. If a genesis round is abandoned, your BNB is retrievable
> with a single transaction for as long as the chain exists. That property is
> the whole point of this document. Read §4 for when the path opens — it is
> later than v5.0 of this guide claimed, and the difference is a week.

> ⚠️ **Owner and incident procedures are NOT here.** `pause`, `setBlacklist`,
> `setPogSigner`, `haltLadderMinting` and the rest live in the operators'
> incident playbook, which is not public. This file is user-side only, and
> nothing below needs anything you cannot reach.

---

## Read this before the rest — what changed in v5.1

The protocol moved from Robinhood Chain to **BNB Smart Chain**, and from
Uniswap V4 to **PancakeSwap Infinity**. v5.0 of this guide described the old
deployment, and four of its corrections are the kind that waste your time
rather than merely read oddly:

- **The soft cap does not decide whether you can refund, and v5.0 said it
  did.** This is the correction most likely to have cost somebody real
  distress, so it is first. v5.0's phase table said a round that *missed* its
  soft cap at the deadline was a "failed genesis" you could refund
  immediately. It is not. `canRefund()` reads nothing but the clock: the only
  trigger is the creator letting the **7-day launch window lapse**. A round
  under its cap stays launchable for that entire week, and a depositor who
  followed v5.0 would have called `refund()` at the deadline, been refused,
  and had every reason to think they had been rugged. Corrected in §1, §3, §4
  and §6. This error predates the port — it was wrong about Robinhood too.
- **`totalEthDeposited()` and `ethDeposited(address)` do not exist**, and never
  did on this build. The getters are `totalSatoDeposited()` and
  `satoDeposited(address)`. Every §1 command in v5.0 that used the `Eth` names
  would have failed with no matching function. Corrected throughout.
- **`hookSalt` is no longer mined.** v5.0 sent you off to grind a CREATE2
  address matching the mask `0x20CC`. Infinity declares hook permissions as a
  bitmap instead of encoding them in the address, the factory's bits check is
  gone, and `InvalidHookSalt` no longer exists. Any salt works. See §7.
- **The factory's source is not published on BscScan.** v5.0 told you the
  factory was verified and you could read what you were about to send money
  to. On this chain the *treasury* is verified and the *factory* is not. §0
  says what to do instead.

Denominations are BNB everywhere. Function and error names are unchanged where
the contract still spells them the old way — those are ABI, not prose, and
renaming them here would just make the commands fail.

**The only deployment is testnet 97.** There is no mainnet Tosh contract. What
follows works, and it works on a chain where nothing is at stake.

---

## 0. Setup

Install Foundry (`foundryup`), then set your environment. Every command below
assumes these are exported.

```bash
export RPC=https://data-seed-prebsc-1-s1.bnbchain.org:8545
export FACTORY=0xB224f26a323320376c0b4C6a3228533FA63E5bBd
export TREASURY=0x79de222644E8BBeea6FC55815CCBE9FF136D7674
export HOOK=0x<project-hook-address>     # the project you care about
export TOKEN=0x<project-token-address>
export ME=0x<your-wallet-address>
```

`$TREASURY` is exported here rather than in §5.5 where it is first used,
because v5.0 used it there without ever defining it.

**Where to get the factory address without trusting the site:** two sources
that are not the app, so you can cross-check rather than take either on faith.

1. The **status page** above prints it and links it to the explorer. That page
   is hosted separately from the application and shares none of its
   infrastructure.
2. The **explorer** itself: the factory is the contract that *emits* every
   `LaunchCreated` log — the log's own `address` field, not the transaction's
   `to`, which is only the same thing when someone calls `createLaunch`
   directly and differs the moment the call is routed through anything else.

If those two disagree, do not deposit, and treat it as a report worth sending
us.

> **You cannot read the factory's Solidity on BscScan, and v5.0 said you
> could.** Its source is unpublished on chain 97; the treasury's is published
> (compiler `v0.8.26+commit.8a97fa7a`). So the cross-check above is on the
> *address*, not on the source. What is available instead needs no explorer
> and no trust in us: the deploy artefact at
> `broadcast/Deploy.s.sol/97/run-latest.json` in the protocol repository
> records the address and the commit, and the factory's own
> `HOOK_CREATION_CODEHASH` can be recomputed from that source tree. Publishing
> the factory is a deliberate non-priority on a rehearsal chain whose keys are
> public; on mainnet it would not be optional.

### Reads vs writes

Everything in §1 is a `cast call` — free, read-only, no key needed. Writes in
§2 onward need a signer. Prefer a hardware wallet:

```bash
# Ledger
--ledger --mnemonic-derivation-path "m/44'/60'/0'/0/0"

# Keystore file (never paste a raw private key into a shell)
--keystore ~/.foundry/keystores/<name>
```

Every write example below ends with `$SIGNER`; substitute one of the above.

```bash
export SIGNER='--ledger'
```

> **Simulate before you send.** Append `--rpc-url $RPC` to a `cast call` with
> the same arguments as your intended `cast send` and you get the revert reason
> for free, without paying gas for a failed transaction. Do this every time;
> §6 lists what the errors mean.

---

## 1. Find out where you stand

Run these first. They tell you which section of this document applies.

```bash
# Has the project launched? false = still in genesis or awaiting launch
cast call $HOOK "launched()(bool)" --rpc-url $RPC

# Genesis deadline (unix seconds) and how much has been raised
cast call $HOOK "genesisDeadline()(uint256)"     --rpc-url $RPC
cast call $HOOK "totalSatoDeposited()(uint256)"  --rpc-url $RPC

# The advertised target. Informational ONLY -- see the note below
cast call $HOOK "softCap()(uint256)"             --rpc-url $RPC

# Your own position
cast call $HOOK "satoDeposited(address)(uint256)" $ME --rpc-url $RPC
cast call $HOOK "hasClaimed(address)(bool)"       $ME --rpc-url $RPC
cast call $HOOK "claimableReferral(address)(uint256)" $ME --rpc-url $RPC

# THE decisive read: can you refund right now?
cast call $HOOK "canRefund()(bool)" --rpc-url $RPC
```

`cast call` prints wei. `cast from-wei <value>` converts to BNB.

> The deposit getters are `totalSatoDeposited` / `satoDeposited`, not
> `totalEthDeposited` / `ethDeposited`. If you copied the v5.0 commands and got
> "function not found", that is why, and nothing was wrong with your setup.

> **`softCap()` is a target, not a gate**, and treating it as a gate is the
> single biggest error in v5.0 of this guide. Nothing in the protocol consults
> it: `launch()` does not check it, `refund()` does not check it, and the only
> place the value is read at all is a one-time non-zero assertion when the hook
> is cloned. A creator can launch a round that raised a fraction of its cap —
> that is not a loophole, it is what the chain did on 2026-09-17, launching
> 0.035 BNB against a 0.175 BNB cap. So read `softCap()` to know what the
> project *advertised*, and read the clock to know what you can *do*.

### Which phase am I in?

Only two things decide your options: whether `launched()` is true, and where
the clock sits relative to `genesisDeadline()`. The soft cap appears nowhere in
this table, which is the whole correction.

| `launched()` | `canRefund()` | `block.timestamp` vs `genesisDeadline()` | Phase | Go to |
|:---:|:---:|---|---|---|
| `false` | `false` | before the deadline | **Genesis open** — deposits accepted | §2 |
| `false` | `false` | after it, within `LAUNCH_WINDOW` (7d) | **Awaiting launch** — creator may still launch, cap met or not | §3 |
| `false` | `true` | more than 7d after it | **Abandoned** — refund | §4 |
| `true` | `false` | — | **Live** — claim, trade, shelf-mint | §5 |

There is no "failed genesis" row, because the contract has no such state. A
round that under-raises and a round that over-raises follow the same path, and
both wait on the same person: the creator.

The middle row is the one to be careful about. It can last a full week, during
which your deposit is neither working nor retrievable, and nothing on chain
will tell you which way it is going to resolve. Compute when it ends:

```bash
cast call $HOOK "genesisDeadline()(uint256)" --rpc-url $RPC   # add 604800
```

---

## 2. Genesis — depositing

Deposits go through the **factory**, never the hook directly (`hook.deposit` is
factory-only and will revert with `OnlyFactory`).

### 2.1 Check you are eligible first

```bash
cast call $FACTORY "eligibility(address,address)(bool,uint256,uint256)" \
  $ME $HOOK --rpc-url $RPC
# → (eligible, remainingQuota, cooldownRemaining)
```

A `false` here means one of: you have no PoG quota, you have used it,
your per-hook cooldown has not elapsed, or you are blacklisted. §6 maps the
revert names.

### 2.2 PoG registration requires the off-chain signer

```bash
cast send $FACTORY \
  "registerPoG(uint256,uint256,uint256,bytes)" \
  <maxAlloc> <deadline> <nonce> <signature> \
  --rpc-url $RPC $SIGNER
```

> **This is the one action you cannot self-serve.** The four arguments come
> from an attestation signed by the protocol's PoG oracle key; the contract
> verifies the signature and there is no way to forge one. If the signing API
> is down, **new** PoG registration is genuinely unavailable — but every
> already-registered quota, every deposit made with it, and every refund
> continue to work. Signatures expire (`MAX_SIG_VALIDITY = 24 hours`).

### 2.3 Deposit

```bash
cast send $FACTORY "deposit(address,address)" \
  $HOOK 0x0000000000000000000000000000000000000000 \
  --value 0.05ether \
  --rpc-url $RPC $SIGNER
```

The second argument is your referrer. Use the zero address for none.

**Referral binding is per project, and v5.0 said it was platform-wide.** That
changed with the 2026-09-12 redeploy and it changes what the argument means to
you: the first non-zero referrer you name **for a given hook** is the only one
that hook will ever credit, and naming a different one on a different project
is allowed rather than ignored.

There are two bindings, and they answer different questions:

```bash
# The referrer credited for THIS project
cast call $FACTORY "projectReferrerOf(address,address)(address)" \
  $ME $HOOK --rpc-url $RPC

# Your permanent, platform-wide referrer — set once, on your first ever deposit
cast call $FACTORY "referrerOf(address)(address)" $ME --rpc-url $RPC
```

`REFERRAL_BPS = 1000` (10%) of a deposit goes to referral, split
`PROJECT_REFERRAL_SHARE_BPS = 8000` — 80% of it to the project referrer, the
remainder to the permanent one.

Your deposit is capped per wallet by `perWalletCap()`; exceeding it reverts
with `PerWalletCapExceeded`.

---

## 3. Triggering the launch (project creator only)

Callable only by the address in `creator()`, only after the genesis deadline
has passed, and only within `LAUNCH_WINDOW = 7 days` of it.

```bash
cast call $HOOK "creator()(address)" --rpc-url $RPC   # confirm it is you

cast send $HOOK "launch()" --rpc-url $RPC $SIGNER --gas-limit 3000000
```

> **The soft cap is not a precondition, and v5.0 said it was.** `launch()`
> refuses a caller who is not the creator, a deadline that has not passed, a
> round already launched, a window already lapsed, and a raise of *exactly
> zero* (`ZeroAmount`). It does not look at `softCap()`. If you are a creator
> who read v5.0, concluded your under-subscribed round was dead, and let the
> week run out, the cap was never what stopped you.
>
> Whether launching a thin round is *wise* is a different question and this
> guide does not have an opinion: the raise minus commission becomes the
> pool's entire initial liquidity, so a small one opens a thin, easily-moved
> market. The contract lets you. It does not advise you.

`launch()` seeds the **PancakeSwap Infinity** CL pool in one transaction and is
expensive — measured at ~660,000 gas as a budget ceiling in the test suite —
so let `cast` estimate first, and only pin `--gas-limit` if estimation fails.

> **Miss the 7-day window and the launch is gone.** `LaunchWindowExpired` is
> terminal: the round converts to a refund and every depositor takes their BNB
> back. There is no owner override and no extension. If you are the creator and
> your window is closing, this is the most time-sensitive command in this
> document.

> **Listing on the buyback ladder is a separate step and it can refuse you.**
> `addLadderToken` reads the pool's TWAP and reverts `TwapNotMature` until a
> full `TWAP_WINDOW` (30 minutes) of price history exists. Attempting it in the
> same transaction as `launch()` does not work — that is not a theory, it is
> what the 2026-09-17 rehearsal on this chain did, saw refused, and retried
> successfully half an hour later.

---

## 4. Refunds — the path that always works

Available when `canRefund()` returns `true`, which has exactly one trigger:
**the creator let the 7-day launch window lapse without launching.** In full,
that getter is

```solidity
if (launched) return false;
return block.timestamp > genesisDeadline + LAUNCH_WINDOW;
```

and there is nothing else to it. Not the soft cap, not the deposit count, not
the owner.

```bash
cast call $HOOK "canRefund()(bool)" --rpc-url $RPC   # must be true

cast send $HOOK "refund()" --rpc-url $RPC $SIGNER
```

> **v5.0 said a missed soft cap opened this path at the deadline. It does
> not.** If your round under-raised and the deadline has passed, you are in the
> middle row of §1's table and you must wait out the remainder of the week. The
> refusal you get before then is `Refund not available` — which reads like the
> function is broken or you are blocked, and means neither. It means *not
> yet*, and §1 shows how to compute *when*.

You receive **your exact deposit** in BNB. Facts worth knowing at 3am:

- `refund()` is **not** gated by `pause()`. A paused platform still refunds.
  Pinned by `test_pause_doesNotBlockRefund`.
- It is **not** gated by the ladder halt either.
- There is **no deadline** on refunding. The hook holds your BNB indefinitely;
  claim it in a year if you like.
- No owner, admin, or creator can block, delay, or redirect it.
- It pays **100% of your deposit**, not the deposit minus commission. The 10%
  referral carve is only realised inside `launch()`, so an abandoned round owes
  its referrers nothing and there is no haircut to argue about.

Its three refusals are plain `require` strings rather than the custom errors
everything else in this document uses, so `cast` prints them verbatim:

| What you see | What it means |
|---|---|
| `Genesis not ended yet` | Before the deadline. Deposits are still open (§2). |
| `Refund not available` | After the deadline, inside the 7-day window. **Wait** — you are early, not blocked. |
| `Already launched` | The round launched. Your position is tokens now, not BNB (§5.1). |
| `NoDeposit` | This address never deposited here, or already refunded. |

### 4.1 Freeing an abandoned project's name

Not in v5.0 of this guide at all, and it belongs to users rather than to
operators, so it is here: once a round becomes refundable its name and symbol
are still locked, and **anyone** can release them.

```bash
# Which name key a hook holds -- read it rather than deriving it
cast call $FACTORY "hookNameKey(address)(bytes32)" $HOOK --rpc-url $RPC

# Is a given name/symbol pair taken? The key is keccak256(abi.encode(name,
# symbol)) -- a PAIR, ABI-encoded, and case-sensitive. "Tosh"/"TOSH" and
# "tosh"/"TOSH" are different names.
cast call $FACTORY "nameTaken(bytes32)(bool)" \
  $(cast keccak $(cast abi-encode "f(string,string)" "My Project" "MYP")) \
  --rpc-url $RPC

cast send $FACTORY "releaseAbandonedName(address)" $HOOK \
  --rpc-url $RPC $SIGNER
```

It is permissionless for a reason worth understanding, because it is the same
reasoning as `pokeBuyback()` in §5.5: the precondition is objective and read
off the chain (`canRefund()` on the hook must be `true`), and there is nothing
to steal, since a live or launched project can never satisfy it. Leaving the
job to the creator would strand precisely the names most worth reclaiming — an
abandoned launch is by definition one whose creator stopped showing up.

Like `refund()`, it is deliberately **not** blocked by `pause()`: a pause is
one of the things that can kill a genesis, so it must not also block the
cleanup.

| Error | Meaning |
|-------|---------|
| `NameStillHeld` | The hook is not refundable yet, or the name was already released. |
| `HookNotRegistered` | Not a hook this factory deployed. Cross-check with §8. |

---

## 5. After launch

### 5.1 Claim your genesis tokens

```bash
cast call $HOOK "hasClaimed(address)(bool)" $ME --rpc-url $RPC   # want false
cast send $HOOK "claimGenesis()" --rpc-url $RPC $SIGNER
```

Pro-rata from `GENESIS_CLAIM_SUPPLY` against your share of
`totalSatoDeposited()`. One claim per address, no deadline.

The genesis allocation is `GENESIS_SUPPLY = 8,400,000` tokens and it splits in
two: 4,620,000 (55%) is claimable by depositors, and `GENESIS_LP_SUPPLY =
3,780,000` (45%) is seeded into the pool by `launch()` and locked there. So
your claim is a share of the 55%, not of the whole 8.4M, and the arithmetic is
`4_620_000e18 * yourDeposit / totalSatoDeposited()`.

### 5.2 Claim referral rewards

```bash
cast call $HOOK "claimableReferral(address)(uint256)" $ME --rpc-url $RPC
cast send $HOOK "claimReferralReward()" --rpc-url $RPC $SIGNER
```

Paid in BNB, accrues at `REFERRAL_BPS = 1000` (10%) of your referees'
deposits, claimable only after launch. See §2.3 for how that 10% splits
between the project and permanent bindings.

### 5.3 Buy from the shelf ladder

Quote first — the price gate can reject a mint that looked fine a block ago:

```bash
# Ceiling on what you can mint right now (0 = gate closed or halted)
cast call $HOOK "maxMintable()(uint256)" --rpc-url $RPC

# Exact BNB cost for a given token amount
cast call $HOOK "quoteMint(uint256)(uint256)" <tokenAmount> --rpc-url $RPC

# Full gate snapshot:
# (tierIndex, tierPrice, remaining, spotPrice, twapPrice, ceiling, unlocked)
cast call $HOOK \
  "tierStatus()(uint256,uint256,uint256,uint256,uint256,uint256,bool)" \
  --rpc-url $RPC
```

Then mint, sending **at least** the quoted amount:

```bash
cast send $HOOK "mintBondingCurve(uint256)" <tokenAmount> \
  --value <quotedCostInBnb> \
  --rpc-url $RPC $SIGNER
```

Overpayment is refunded within the same transaction; underpayment reverts with
`InsufficientPayment`. A single call spans at most `MAX_TIERS_PER_TX = 32`
shelves — ask for more and it reverts with `SpanTooManyShelves`; split the
order. A single-shelf mint costs about 182,000 gas.

### 5.4 Trade on the pool

Once launched, the token is an ordinary **PancakeSwap Infinity** CL asset.
Fetch the pool key and route through Infinity periphery or any aggregator:

```bash
cast call $HOOK "getPoolKey()" --rpc-url $RPC
```

> **The pool key has six fields, not five, and that matters if you are
> encoding calls by hand.** An Infinity `PoolKey` is
> `(currency0, currency1, hooks, poolManager, fee, parameters)` — it carries
> the address of the pool manager, and a `parameters` word that packs the tick
> spacing together with the hook's permission bitmap. A Uniswap V4 `PoolKey`
> has five fields and puts tick spacing in its own slot. They are not
> interchangeable, and the action opcodes for liquidity payloads happen to be
> numerically identical between the two, so the key's width is the only thing
> that will tell you which chain's periphery you are talking to.
>
> You can settle it from the raw return without decoding anything: the call
> above answers with `2 + 64 × slots` hex characters, so **386 characters means
> six slots and Infinity**, where 322 would mean five and V4. On the chain-97
> rehearsal hook it is 386.

Pool parameters are `POOL_FEE = 3000` (0.30%) and `TICK_SPACING = 200`. A
1.00% tax applies on hook-routed swaps (`TAX_BPS = 100`), so total friction is
1.30%. Buys split theirs — 0.70% of the BNB input to the ladder treasury,
0.30% to the platform's fee recipient (`PLATFORM_SWAP_FEE_BPS = 30`). Sells are
not split: the full 1.00% of the token input is burned.

> **Beware lookalike pools.** Anyone can open an Infinity pool for the same
> token with a different hook — one that has no tax and no ladder. The
> canonical pool is the one `getPoolKey()` returns. Verify before you trade
> size.

Your swap may or may not carry a buyback, and you cannot choose. `afterSwap`
pokes the treasury only when the trade has gas to spare
(`gasleft() >= PIGGYBACK_MIN_GAS`, currently **270,000**). Measured under
`--isolate` on the Infinity build:

| | gas |
|-|-|
| swap, warm pool, no buyback | 195,949 |
| swap, cold pool, no buyback | 252,726 |
| swap carrying a buyback, 1 token listed | 425,932 |
| swap carrying a buyback, 3 tokens listed | 445,869 |

All four are up on the Uniswap V4 figures v5.0 quoted (which were ~192,000 and
~420,000), for one reason: a swap now settles through Infinity's **Vault**
rather than directly against the pool manager. v5.0's "roughly 363,000 instead
of ~217,000" described neither this AMM nor this chain.

The flatness of the last two rows is deliberate and is asserted in the test
suite: a poke runs **one** leg, so listing more tokens on the ladder must never
make somebody else's trade more expensive. It used to run up to `BATCH_SIZE`
legs, and that row read 578,809.

This is worth knowing when you set a gas limit by hand. A limit sized against a
simulation that happened to skip the buyback will still succeed — the gate
withholds the swap's remaining gas physically, so a poke can never strand the
trade. What it cannot do is make the buyback happen on a tight limit. If you
*want* your swap to carry one, leave headroom; wallets that simulate and add
~15% generally do.

### 5.5 Poke the buyback yourself

Anyone can deploy the reservoir without trading:

```bash
cast send $TREASURY "pokeBuyback()" --rpc-url $RPC $SIGNER
```

No role, no Safe transaction, no allowlist. This exists because the gas gate in
§5.4 means trading alone no longer guarantees the treasury empties: in a market
where wallets attach tight limits, the reservoir can hold BNB indefinitely with
nothing in the event stream to say so. `STATE-06` in `monitoring/alerts.json`
polls for exactly that, and this is its remedy.

It is safe to leave to strangers because it decides nothing. The venue comes from
the listed token's own hook, the size from the treasury balance, the order from
the round-robin cursor, and the price is bounded by the same TWAP floor as every
other leg. Proceeds go to `0xdead`. The only choice a caller has is *when*, and
the cursor makes that uninteresting.

```bash
# Is there anything listed for it to buy?
cast call $TREASURY "ladderTokenCount()(uint256)" --rpc-url $RPC
```

Reverts you should expect rather than debug:

| Error | Meaning |
|-------|---------|
| `NotArmed` | Balance is below `TRIGGER_STEP` (**3.5 BNB** — v5.0 said 1 ETH, which was this constant on the old chain), or no tokens are listed. Nothing to deploy. |
| `PiggybackInProgress` | A cycle is already running in this call stack. Retry after it settles. |

---

## 6. When a transaction reverts

`cast` prints the custom error name. What each one means for you:

### You are not allowed to do this yet

| Error | Meaning | Do this |
|-------|---------|---------|
| `GenesisActive` | `launch()` before the genesis deadline. | Wait for the deadline. |
| `GenesisExpired` | Deposit after the deadline. | Deposits are closed for good; see §1 for what happens next. |
| `NotLaunched` | Action requires a launched project. | Wait for the creator, or refund once the window lapses. |
| `AlreadyLaunched` | Genesis is over. | Use §5. |
| `ZeroAmount` | `launch()` on a round that raised nothing at all. | Nothing to launch. It will become refundable, though there is nothing to refund. |
| `LaunchWindowExpired` | Creator missed the 7-day window. | Refund (§4). Terminal. |
| `OnlyCreator` / `OnlyFactory` / `Unauthorized` | Wrong caller. | `deposit` goes through the factory; `launch` needs `creator()`. |

> v5.0 listed `SoftCapNotMet` here. **No such error exists in the contracts** —
> it is not declared anywhere in `src/`, because nothing ever needed to raise
> it. It was the visible end of the soft-cap misconception described at the top
> of this guide, and it is deleted rather than relabelled.

### Your eligibility

| Error | Meaning | Do this |
|-------|---------|---------|
| `IsBlacklisted` | Address is banned. | Check `blacklistedUntil(address)`; bans can be time-limited. |
| `NoPogQuota` / `QuotaExceeded` | No or insufficient PoG quota. | Register (§2.2) or wait for the quota window to refill. |
| `CooldownActive` | Re-depositing to the same hook too soon. | `eligibility(...)` returns the seconds remaining. |
| `InvalidSignature` / `SignatureExpired` / `NonceConflict` | Bad, stale (>24h), or reused attestation. | Request a fresh one. |
| `PerWalletCapExceeded` | Deposit exceeds `perWalletCap()`. | Send less. |

### Claims and rewards

| Error | Meaning |
|-------|---------|
| `AlreadyClaimed` | Genesis tokens already claimed. |
| `NoDeposit` | This address never deposited into this hook. |
| `NoReferralReward` | Nothing accrued, or already withdrawn. |

### The shelf ladder

| Error | Meaning | Do this |
|-------|---------|---------|
| `InsufficientPayment` | `--value` below the quote. | Re-quote and resend; the price moves. |
| `TierPriceAboveCeiling` | Shelf price is above the TWAP-derived ceiling. | Gate is protecting you. Wait for the pool to catch up. |
| `LadderMintingHalted` | Owner halted shelf minting. | Auto-expires within 7 days. Trading, claims and refunds are unaffected. |
| `SameBlockMintForbidden` | Minting in the pool's own swap block. | Retry next block; `lastSwapBlock()` says which one that was. |
| `ExceedsTierRemaining` / `LadderExhausted` | Order exceeds the shelf, or the ladder is sold out. | Check `tierRemaining()` / `bondingRemaining()`. |
| `SpanTooManyShelves` | Order crosses more than 32 shelves. | Split it. |

### Platform state

| Error | Meaning |
|-------|---------|
| `EnforcedPause` | Platform paused. Affects **only** `createLaunch` and `registerPoG` — deposits into open rounds, claims, trading and refunds all continue. |
| `InsufficientLaunchFee` / `FeeChanged` | Launch fee wrong or changed mid-flight. Re-read `launchFee()`. |
| `CapsChanged` | `expectedSoftCap` or `expectedWalletCap` no longer matches the factory's dials. Re-read and resend — see §7. |
| `NameTaken` | Name/symbol already used. |
| `InvalidAdmin` | `projectAdmin` was the zero address. |

### The buyback treasury

| Error | Meaning |
|-------|---------|
| `NotArmed` | `pokeBuyback()` with the reservoir below `TRIGGER_STEP` or with no tokens listed. Reverts rather than returning quietly, because unlike the swap-borne poke this is nobody's hot path and a caller deserves to know the call did nothing. |
| `PiggybackInProgress` | A buyback cycle is already open in this call stack. |
| `OnlyVault` | Something other than Infinity's **Vault** called `lockAcquired`. You should never see this; if you do, it is not a configuration problem. |
| `OnlyHook` | `autoPiggybackBuyback()` called directly. That entry point is for registered hooks only — use `pokeBuyback()` (§5.5), which is the permissionless door. |
| `PoolNotLaunched` | `addLadderToken` on a project whose `launch()` has not run. Distinct from `InvalidPoolKey`: the hook exists and its key is well-formed, there is just no pool behind it yet. |
| `TwapNotMature` | `addLadderToken` before the pool has a full 30-minute TWAP. Wait it out; see the note in §3. |

> **On `OnlyVault` vs `OnlyPoolManager`, because v5.0 named the wrong one and
> the right answer is "both, for different callers".** v5.0 documented the
> treasury's callback as `unlockCallback` guarded by `OnlyPoolManager`, which
> was the Uniswap V4 shape. Infinity moved the lock frame to the Vault, so the
> callback is now `lockAcquired` and its guard is `OnlyVault` — checking the
> pool manager there would reject the only caller that can legitimately
> arrive.
>
> But `OnlyPoolManager` was **not** deleted, and it is not a leftover. It still
> guards the hook's actual hook callbacks — `beforeSwap`, `afterSwap` and
> friends — because those really are invoked by the CL pool manager. So the
> two errors coexist and the distinction is which frame you are in: settlement
> comes from the Vault, hook callbacks come from the pool manager. Seeing
> either in the wild means something is calling a privileged entry point
> directly, and neither is anything you can cause with the commands in this
> guide.

---

## 7. Creating a launch manually

`createLaunch` takes nine arguments:

```solidity
createLaunch(
    string  name,
    string  symbol,
    address projectTreasury,
    address projectAdmin,
    bytes32 hookSalt,          // ← any value; see below
    uint256 expectedFee,
    uint256 expectedSoftCap,
    uint256 expectedWalletCap,
    uint256 genesisDuration    // 3h | 24h | 72h
) payable returns (address token, address hook)
```

> **`hookSalt` is not mined any more, and v5.0's instructions for mining it are
> the most actively harmful thing in that version.** It sent readers off to
> grind a CREATE2 address whose low bits match `0x20CC`, against a hash read
> from the live factory. None of that applies:
>
> - Infinity hooks declare their permissions as a **bitmap**
>   (`getHooksRegistrationBitmap()`), carried in the pool key's `parameters`
>   word. Uniswap V4 encoded the same permission set in the hook's address,
>   which is why grinding was ever necessary.
> - The factory's bits check is deleted, and `InvalidHookSalt` no longer
>   exists as an error. **Any salt is accepted.**
> - The salt you pass is not the salt used. The factory deploys at
>   `keccak256(abi.encode(msg.sender, hookSalt))`, so two callers passing the
>   same value do not collide, and picking `bytes32(0)` is fine.
>
> There is no mining script to run because there is nothing left to mine.

**`expectedFee`, `expectedSoftCap` and `expectedWalletCap` are guards against
the dials moving under you**, not values you choose. Read each from the factory
in the same breath as you send, and expect `FeeChanged` or `CapsChanged` if the
owner changed one in between:

```bash
cast call $FACTORY "launchFee()(uint256)"          --rpc-url $RPC
cast call $FACTORY "defaultSoftCap()(uint256)"     --rpc-url $RPC
cast call $FACTORY "maxPogAllocationLimit()(uint256)" --rpc-url $RPC
```

`genesisDuration` must be exactly `DURATION_FAST` (3h = `10800`),
`DURATION_STANDARD` (24h = `86400`), or `DURATION_SLOW` (72h = `259200`);
anything else reverts with `InvalidDuration`.

The whole path costs about 614,000 gas, most of it deploying the token proxy.

---

## 8. Reading the registry

Enumerate every launch this factory has produced, without any API:

```bash
cast call $FACTORY "launchCount()(uint256)" --rpc-url $RPC

# LaunchInfo: (token, hook, creator, createdAt)
cast call $FACTORY "launches(uint256)(address,address,address,uint256)" \
  <index> --rpc-url $RPC

# Reverse lookup
cast call $FACTORY "tokenToHook(address)(address)" $TOKEN --rpc-url $RPC

# Confirm a hook really came from this factory — do this before depositing
# into a hook address someone sent you
cast call $FACTORY "registeredHooks(address)(bool)" $HOOK --rpc-url $RPC
```

Or from event logs:

```bash
cast logs --from-block 131563800 --address $FACTORY \
  "LaunchCreated(uint256,address,address,address,string,string)" \
  --rpc-url $RPC
```

131563800 is the block the current factory was deployed in, so there is nothing
to find before it.

> The public BNB dataseed endpoints rate-limit `eth_getLogs` and will refuse a
> wide range rather than answer it slowly. If a query comes back empty when you
> expect rows, narrow the block range before concluding there are none.

---

## 9. Safety checklist

Before signing anything:

1. **Verify the factory address** against the status page *and* the explorer,
   and if you have the protocol repository, against
   `broadcast/Deploy.s.sol/97/run-latest.json`. Everything else derives from
   it. (v5.0 pointed at `factoryDeployments.ts`, which does not exist — the
   addresses are read out of Foundry's own broadcast logs.)
2. **Verify the hook is registered** — `registeredHooks(hook)` must be `true`.
   An unregistered "hook" is someone else's contract.
3. **Check the pool key came from the hook you trust** — `getPoolKey()` on the
   hook, not a key someone handed you. §5.4 explains the lookalike-pool risk.
4. **Simulate with `cast call`** before `cast send`. Free revert reasons.
5. **Never paste a private key into a shell.** `--ledger` or `--keystore`.
6. **Check `canRefund()` before you panic** — and read its answer correctly,
   because v5.0 would have had you misread it. `true` means your BNB is
   already retrievable and nothing else is urgent. `false` on an unlaunched
   round means *not yet*, and says nothing about whether the round hit its
   soft cap; work out the 7-day date from §1 and check back then.

> **And one that only applies to chain 97:** this deployment's `owner`,
> `pogSigner` and `platformTreasury` are all the same address, and its private
> key is public — it was committed to the protocol repository and pushed.
> Anyone can pause it, sign PoG attestations for it, or collect its platform
> fees. That is acceptable because it is a rehearsal holding nothing, and it is
> the reason you should not put real money anywhere in this guide's reach.

---

*Last updated: 2026-09-18 (v5.1 — BNB Smart Chain, PancakeSwap Infinity. v5.0
described Robinhood Chain and Uniswap V4; every changed fact is flagged inline
rather than silently replaced, because a reader who followed the old version
deserves to know which of their assumptions broke.)*
