# Liquidity Layer — Design

This document describes the intended behaviour of the **Liquidity** program
(`programs/liquidity`, mainnet `jupeiUmn818Jg1ekPURTpr4mFo29p46vygyykFJ3wZC`), the core of
Jupiter Lend. Jupiter Lend is a Solana port of Fluid's Liquidity Layer design, so most of
the material below comes from Fluid's public EVM documentation and Jupiter's developer and
user docs. Where the Solana program differs from the EVM design, the Solana behaviour is
called out explicitly. Sources are listed at the end.

## 1. Role in the system

Jupiter Lend has two layers:

- **Protocol layer** — user-facing programs: **Lending** (deposit/withdraw for yield,
  fTokens) and **Vaults** (collateralised debt positions with liquidation). Flash loans also
  sit on top of the liquidity layer.
- **Liquidity layer** — a single pool per token that holds all supplied funds and lends
  them out to the protocols above it.

> "The Liquidity program serves as the foundational single-pool architecture that manages
> all liquidity within the system. It tracks the deposit and borrow positions of integrated
> programs and enforces program-specific borrowing and withdrawal controls, including
> rate-limited mechanisms such as debt ceilings and withdrawal limits. Access to this
> program is permission-based, and end users never interact with it directly. Only
> whitelisted programs, such as Vaults and Lending, can interact with it through CPIs."

The "users" of the Liquidity program are therefore **protocols** (programs, signing with a
PDA), not end users. Each protocol has, per token, at most one **supply position** and one
**borrow position**. Each protocol is risk-isolated by its own limits while sharing the
token's pool.

## 2. Accounts

| Account | Per | Holds |
|---|---|---|
| `Liquidity` | program | `authority`, `revenue_collector`, global `status` (locked / unlocked), PDA bump |
| `AuthorizationList` | program | `auth_users`, `guardians`, `user_classes` |
| `TokenReserve` | mint | exchange prices, borrow rate, fee, utilisation, max utilisation, last update time, the four totals, total claim amount, "interacting" deposit bookkeeping |
| `RateModel` | mint | rate curve version (1 or 2) and its kinks / rates |
| `UserSupplyPosition` | (protocol, mint) | mode, amount, withdrawal-limit state, status |
| `UserBorrowPosition` | (protocol, mint) | mode, amount, debt-ceiling state, status |
| `UserClaim` | (user, mint) | claimable balance |
| vault | mint | the SPL token account (ATA of the `Liquidity` PDA) holding the pool's tokens |

## 3. Units and precision

- Percentages, rates, fee and utilisation are in **1e2 basis**: 100% = 10 000, 1% = 100
  (`FOUR_DECIMALS = 1e4`). Rates and utilisation are stored as `u16` (max 655.35%).
- Exchange prices use `EXCHANGE_PRICES_PRECISION = 1e12` ("1.0 == 1e12").
- `SECONDS_PER_YEAR = 365 days` (leap years ignored).
- Token decimals allowed on Solana: **2 to 9** (EVM: 6 to 18).
- `MAX_TOKEN_AMOUNT_CAP = 2^60 - 1`: total amounts must stay below this cap
  (EVM uses `int128.max`). Beyond it, only value-reducing operations remain possible.

## 4. Raw vs normal amounts; with-interest vs interest-free

Every position, on each side (supply / borrow), is configured in one of two **modes**:

- **with interest** (`mode = 1`): the position earns (supply) or pays (borrow) interest.
  Its amount is stored as a **raw** amount, which must be multiplied by the exchange price
  to get the token amount: `normal = raw * exchangePrice / 1e12`.
- **interest free** (`mode = 0`): the amount is stored as a **normal** token amount and
  never changes with time.

The token keeps four totals: `total_supply_with_interest` (raw),
`total_supply_interest_free` (normal), `total_borrow_with_interest` (raw),
`total_borrow_interest_free` (normal).

```
totalSupply = supplyInterestFree + supplyRawInterest * supplyExchangePrice / 1e12
totalBorrow = borrowInterestFree + borrowRawInterest * borrowExchangePrice / 1e12
```

A position's limits are kept in the same unit as its amount ("with interest -> raw, without
interest -> normal"). When an admin switches a position's mode, the amount and limit are
converted at the current exchange price and the totals are moved from one bucket to the
other.

## 5. Exchange prices and interest accrual

Each token has a **supply exchange price** and a **borrow exchange price**.

- Both are initialised to `1e12` when the token reserve is created.
- **They only increase over time and never drop below 1e12.**
- They are brought up to date (accrued for the elapsed time) at the start of every
  `operate`, and by `update_exchange_price` (permissionless) and by admin config changes,
  which accrue with the *old* parameters before applying the new ones.

**Borrow price.** With `borrowRate` (1e2 basis, per year) and `Δt` seconds since the last
update:

```
borrowExchangePrice += ceil(borrowExchangePrice * borrowRate * Δt / (SECONDS_PER_YEAR * 1e4))
```

It rounds **up**, so the borrow price never grows slower than the supply price.

**Supply price.** All yield paid by with-interest borrowers goes to with-interest
suppliers, minus the protocol fee. Interest-free legs neither pay nor receive interest.

```
supplyRate = borrowRate * (1 - fee) * ratioSupplyYield
ratioSupplyYield = utilisation
                 * (share of borrowers that pay interest)
                 * (total supply / supply that earns interest)
supplyExchangePrice += supplyExchangePrice * supplyRate * Δt / SECONDS_PER_YEAR   (rounded down)
```

The supply and borrow "ratios" (interest-free vs with-interest, 1e4 basis, with the larger
side as denominator) are how the program computes those shares. Worked example from the
code's own documentation: supply with interest 80, interest free 20; borrow with interest
50, interest free 10; borrow rate 40%, fee 10%; after half a year the with-interest supply
of 80 must be worth 89 and the with-interest borrow of 50 must be worth 60 — supply price
1 → 1.1125, borrow price 1 → 1.2. The supply rate can exceed the borrow rate when few
suppliers earn interest.

No accrual happens when: no time has passed, the borrow rate is 0, or there is no
with-interest borrow. If there is no with-interest supply, the supply price does not move
(the interest becomes revenue).

## 6. Utilisation and the rate model

```
utilisation = totalBorrow / totalSupply        (1e2 basis; 0 when totalSupply = 0)
```

Utilisation is computed on 1e12-scaled amounts so the stored value does not lose precision
to intermediate truncation. It can exceed 100% because of rounding and because the
borrow rate exceeds the supply rate (the fee cut). It is recomputed and stored on every
`operate`, together with the borrow rate derived from it.

**Max utilisation** (per token, `0 < max_utilization <= 100%`): **a borrow that pushes
utilisation above `max_utilization` reverts.** Deposits, withdrawals and paybacks never hit
this check.

**Rate curves.** The borrow rate is a piecewise-linear function of utilisation:

- **v1 (one kink):** `rate_at_utilization_zero`, `kink`, `rate_at_utilization_kink`,
  `rate_at_utilization_max` (at 100%). Requires `0 < kink < 100%` and
  `rate_at_kink <= rate_at_max`.
- **v2 (two kinks):** adds a second kink: `0 < kink1 < kink2 < 100%`,
  `rate_at_kink2 <= rate_at_max`.

Before the (last) kink the rate may be flat or even decline; after it, it must not decline.
On each segment `rate = y1 + (y2 - y1) * (u - x1) / (x2 - x1)`. The result is capped at
`u16::MAX` (655.35%); an unsupported version reverts.

**Fee** (`fee_on_interest`, ≤ 100%): the share of borrowers' interest kept by the protocol
as revenue. Jupiter's description: "Borrowers pay the full interest rate, lenders receive
90%, and 10% goes to the protocol treasury."

## 7. `operate`

`operate(supply_amount: i128, borrow_amount: i128, withdraw_to, borrow_to, transfer_type)`
handles all four actions for one protocol and one token in a single call:

| sign | `supply_amount` | `borrow_amount` |
|---|---|---|
| > 0 | deposit | borrow |
| < 0 | withdraw | payback |
| = 0 | nothing | nothing |

It returns the updated `(supply_exchange_price, borrow_exchange_price)`.

**Inflow (Solana-specific).** EVM Fluid pulls tokens through a callback; Solana uses a
three-step flow inside one transaction:

1. `pre_operate(mint)` — records the calling protocol, the current timestamp and the vault's
   current token balance on the `TokenReserve`.
2. The protocol transfers tokens into the vault (SPL transfer).
3. `operate(...)` — for deposits and paybacks, checks that the same protocol called
   `pre_operate` at the same timestamp, and that the vault balance grew by **at least** the
   deposit + payback amount and by **at most 1% more** than that. The recorded
   "interacting" state is then cleared.

**Steps of `operate`:**

1. Reject if both amounts are below `MIN_OPERATE_AMOUNT` (10), either exceeds `i64::MAX`, or
   a withdraw / borrow has no recipient.
2. Verify the inflow (above), if any.
3. Accrue exchange prices to now.
4. Update the supply position and the borrow position (see §8 for limit checks) and the
   token totals. Conversions between normal and raw amounts **round in the protocol's
   favour**: deposit rounds the credited raw amount down, withdraw rounds the debited raw
   amount up, borrow rounds the raw debt up, payback rounds the raw debt reduction down.
   If rounding makes a with-interest operation change nothing, it reverts
   (`OperateAmountsInsufficient`).
5. Recompute utilisation; for a borrow, enforce max utilisation. Recompute the borrow rate
   from the rate model, store rate, utilisation, timestamp and the new exchange prices.
6. Pay out withdrawals and borrows to `withdraw_to` / `borrow_to` (combined into one transfer
   when both go to the same address). Nothing is transferred for supply and payback.

A user cannot withdraw more than its supply or pay back more than its debt (the subtraction
reverts). Totals are clamped at 0 on decrease, so the sum of positions can differ from the
totals by rounding dust.

**Position status.** A supply or borrow position can only be used when its status is
active, i.e. it has been configured by an auth and is not paused.

**Global lock.** When the `Liquidity` status is locked, `pre_operate`, `operate` and
`claim` all revert.

## 8. Automated limits

Limits stop sudden mass withdrawals and borrows. In Jupiter's words, a withdrawal limit
"expands" by a configured percentage over a configured duration ("usually 25% every 6
hours"); "it would take approximately four days for all liquidity to exit the protocol"
(Fluid). Limits are per protocol per token, in the position's unit (raw or normal).

### Withdrawal limit (supply side)

Config: `expand_percent` (≤ 100%), `expand_duration` (seconds, > 0, < 2^24),
`base_withdrawal_limit` (> 0).

The stored **withdrawal limit is a floor**: the user's supply cannot drop below it in one
`operate`.

- **Expansion over time.** Before each operation the limit decays linearly toward the fully
  expanded floor:
  ```
  maxWithdrawable = supply * expandPercent / 1e4
  current = max(lastLimit - maxWithdrawable * elapsed / expandDuration,
                supply - maxWithdrawable)
  ```
  (A stored limit of 0 means "no limit".)
- **Check.** A withdrawal reverts with `WithdrawalLimitReached` if the new supply would be
  below `current`.
- **After the operation.** If the new supply is below `base_withdrawal_limit`, the limit
  becomes 0 (below the base, 100% is withdrawable). Otherwise the stored limit is
  `max(current, newSupply - newSupply * expandPercent / 1e4)`.
- An auth can override the limit with `update_user_withdrawal_limit` (0 = full expansion
  now; `u128::MAX` = nothing withdrawable now).

Worked example (Fluid docs): expand 20%, duration 200 s, base 5. Supply 15, limit 12,
withdraw 2 → supply 13; the limit expands toward `13 * 0.8 = 10.4`; after 100 s it is
`12 - 13 * 0.1 = 10.7`; after 150 s it would be 10.05 but is floored at 10.4.

*Solana difference:* the EVM "decay bucket" (smoothing withdrawable excess from large
deposits over checkpoints) is not part of the Solana program.

### Borrow limit / debt ceiling (borrow side)

Config: `expand_percent` (< 2^14, ~163.83%), `expand_duration` (> 0, < 2^24),
`base_debt_ceiling` (> 0), `max_debt_ceiling` (≥ base, ≤ 10× the token's total supply).

The stored **debt ceiling is a ceiling** on the position's borrow amount.

- **Expansion over time:**
  ```
  maxExpansion = borrow * expandPercent / 1e4
  current = min(lastLimit + maxExpansion * elapsed / expandDuration, borrow + maxExpansion)
  if borrow + maxExpansion < base: current = base
  current = min(current, max_debt_ceiling)
  ```
- **Check.** A borrow reverts with `BorrowLimitReached` if the new borrow would exceed
  `current`. `max_debt_ceiling` is a hard cap that never expands.
- **After the operation.** The stored ceiling is `current`, but not above the fully expanded
  `newBorrow * (1 + expandPercent)` (clamped to `[base, max]`): **paybacks shrink the
  ceiling immediately.**

Worked example (Fluid docs): 20%, 200 s, base 5, max 7. Borrow 4.5 → ceiling 5.4 after full
expansion; a borrow up to 7.01 reverts at the hard cap of 7.

## 9. Claims (Solana-specific)

`transfer_type` selects how withdrawals and borrows are paid out:

- `DIRECT` — SPL transfer from the vault to the recipient's token account.
- `CLAIM` — instead of transferring, credit the recipient's `UserClaim` account (which must
  belong to that recipient and mint); the recipient later calls `claim(recipient)` to
  receive the whole balance. The token tracks `total_claim_amount`, the tokens sitting in
  the vault that are owed to claimants.
- `SKIP` is not accepted for payouts.

Before any payout the vault must hold the amount **on top of** the outstanding claims;
otherwise the operation reverts (`InsufficientBalance`). Claimed tokens are excluded from
revenue.

## 10. Revenue

The protocol's earnings are the tokens in the vault beyond what is owed to suppliers and
claimants, counting what borrowers owe:

```
revenue = max(0, vaultBalance + totalBorrow - totalClaimAmount - totalSupply)
```

with `totalSupply` rounded **up** and exchange prices accrued to now. If total supply is 0,
the whole vault balance minus claims is revenue (so nothing gets stuck).
`collect_revenue` (auth only) transfers `revenue` to the `revenue_collector`.

## 11. Roles and administration

- **Authority** (`Liquidity.authority`, governance multisig): changes the authority, the
  auth list, the guardian list and the revenue collector. The authority cannot be removed
  from the auth list, and can only be handed over to the fixed governance multisig.
- **Auths** (`auth_users`, ≤ 10): set rate curves, token config (fee, max utilisation),
  user supply / borrow configs, withdrawal-limit overrides, user classes, the global status,
  and collect revenue. A fixed init authority may also initialise token reserves and
  register protocols.
- **Guardians** (`guardians`): pause and unpause individual protocol positions. Protocols in
  **user class 1** ("established") cannot be paused by guardians; class 0 (default, new
  protocols) can.
- **Listing order:** `init_token_reserve` (checks decimals, rejects unsupported token
  extensions and absurd total supplies) → rate curve → token config → user configs.
  Out-of-order calls revert (`InvalidConfigOrder`).
- **Registering a protocol:** `init_new_protocol(supply_mint, borrow_mint, protocol)` creates
  that protocol's positions in a "config not set" state; the first user config activates
  them.

## 12. Design properties stated in the public documentation

Collected from the sources below; these are the behaviours the design is meant to
guarantee.

- Exchange prices start at 1e12 and are monotonically non-decreasing.
- Borrow interest accrues at least as fast as supply interest; with-interest suppliers
  receive borrowers' interest net of the fee; the remainder is protocol revenue.
- Rounding always favours the protocol: user supply amounts round down, user borrow amounts
  round up; total supply rounds down and total borrow rounds up (and total supply up when
  computing revenue).
- A borrow can never push utilisation above the token's max utilisation.
- A protocol's supply can never drop below its current withdrawal limit, and its borrow can
  never exceed its current debt ceiling, nor ever `max_debt_ceiling`.
- An `operate` that rounding makes ineffective reverts rather than moving nothing.
- Deposits and paybacks are only credited for tokens that actually arrived in the vault in
  the same transaction.
- Revenue collection never takes funds owed to suppliers or claimants.
- Only registered, configured, unpaused protocols can operate; everything is blocked when
  the program is locked.

## Sources

Fetched 2026-11-05.

- Fluid Solana programs README — github.com/Instadapp/fluid-solana-programs
- Jupiter developer docs, Lend — developers.jup.ag/docs/lend (architecture, liquidity,
  liquidity analytics, earn, borrow, CPI pages, utilisation after deposit)
- Jupiter user docs, Lend — docs.jup.ag/user-docs/earn/lend (liquidity layer and risk
  management, protocol details, earn, statistics)
- Fluid EVM contracts and specs — github.com/Instadapp/fluid-contracts-public
  (`contracts/liquidity/SPEC.md`, `contracts/libraries/SPEC-liquidityCalcs.md`,
  `docs/docs.md`, `docs/fluid-permissions-and-mutability.md`)
- Fluid documentation — docs.fluid.instadapp.io (key concepts and practices, liquidity
  module reference)
- Fluid guides — fluid.guides.instadapp.io (unifying liquidity, liquidity layer risks)
