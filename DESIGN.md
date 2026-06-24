# Strata — design notes

The [README](./README.md) is orientation: what a stratum is and how
value settles toward the foundations. This document records the
contract-level decisions the README leaves implicit, and the questions
still open at the design stage.

## Resolved semantics

These are settled for the first implementation. They are not yet
reflected in the prose of the README because they live below its
altitude — they are caller-correctness detail, not orientation.

### Pricing

Each stratum sets **its own price** to be referenced — a `token`
(currency) and a `price` (amount), fixed at deposit and immutable.
`token == 0x0` denotes the chain's native currency; any other address
is an ERC-20. These two fields are contract-interpreted, distinct from
the opaque payload `addr`/`value`. A `price` of zero is allowed (free
to reference).

### Balance

A stratum's balance is its **on-chain balance, per currency** — 
`address(this).balance` for the native currency and
`token.balanceOf(address(this))` for an ERC-20. Each clone literally
holds its own funds; there is no separate internal accounting
variable. A stratum may accumulate several currencies at once (its own
demanded one, native tips, ERC-20 tips, and whatever a parent pushes
down to it on redemption). (For now: if accounting later needs to
diverge from on-chain balance, this is the line that moves.)

### Deposit

- The **maker is `msg.sender`** — not `tx.origin` — and becomes the
  new stratum's `owner`.
- A reference is paid **its own demanded price in its own currency**,
  credited to it. Native-priced references are paid out of `msg.value`
  (forwarded to the substratum, landing in its native balance);
  ERC-20-priced references are paid by `transferFrom(maker →
  substratum)`, which requires the maker to have approved this contract
  first. One deposit can therefore move several currencies at once.
- **Native overpayment is not refunded:** any `msg.value` beyond the
  sum of the native-priced references' prices is retained as the new
  stratum's native opening balance, where it later settles to that
  stratum's own owner and substrata. ERC-20 payments are pulled in
  exact amounts, so there is no ERC-20 overpayment to handle.
- **Duplicate references are allowed.** `[A, A]` pays A its price twice
  at deposit and, on the new stratum's redemption, gives A two shares.
  Each entry in the reference array is treated independently. (Note:
  this holds only under the flat baseline; a non-flat distribution
  curve would require uniqueness — see Open questions.)

### Redeem

- `redeem(token)` settles **one currency per call** — the stratum's
  balance in `token` (native when `token == 0x0`). A stratum holding
  several currencies is settled one at a time; this avoids having to
  enumerate arbitrary ERC-20 balances on-chain (see Open questions for
  the alternative).
- The chosen currency's balance is divided among the owner and the
  substrata. Under the flat baseline that is `N + 1` equal shares for
  `N` substrata (the owner counts as one), matching the worked example
  in the README.
- **Dust goes to the owner.** Integer division of that currency's
  balance leaves a remainder; the owner's share absorbs it.
- The owner's share is **pushed** to the owner address; substrata
  shares are likewise sent (native transfer, or ERC-20 `transfer`). A
  **revert by the owner is caught** — for both native and ERC-20 — so
  payouts to substrata still succeed and a hostile or contract owner
  cannot brick a permissionless `redeem`. The uncaught case for the
  owner's own share (retained for a later redeem vs. forfeited) is
  noted under Open questions.

## Open questions

Three are already flagged in the README's Status section; the rest are
the silent gaps surfaced while turning the README into a spec.

### From the README Status section

- **Distribution curve.** Flat split is the baseline. An owner-fixed
  share, and weighting earlier-referenced substrata above later ones,
  are open. Any non-flat weighting implies a **uniqueness constraint**
  on the reference array — which would retract the "duplicates allowed"
  decision above.
- **Ownership.** Whether ownership (a stream of redemption income) is
  transferable, and whether strata are ERC-721 NFTs (the
  `name`/`description`/`image` triple already fits the metadata shape).
- **Reference enforcement.** Validating references as genuine strata
  via an on-chain registry vs. bytecode attestation.

### Still to decide

- **Caught owner-payout disposition.** When the owner's push is caught,
  what becomes of that share? Retained in the stratum balance (so a
  later `redeem` retries it, re-dividing it among owner and substrata)
  is the natural default, but "retry" vs. "forfeit downward" should be
  pinned down.
- **`redeem` on a zero balance.** No-op or revert. It is permissionless
  and repeatedly callable, so this needs a defined answer.
- **Reference-count bound.** Deposit pays each reference in a loop and
  redeem divides across them; a large reference array is a gas /
  griefing surface. Cap the array length, or leave it gas-bounded?
- **ERC-20 mechanics.** Non-standard tokens need care: use a
  SafeERC20-style wrapper for tokens that don't return a bool; decide
  the policy for **fee-on-transfer / rebasing** tokens, where the
  amount received differs from the amount sent (credit what actually
  arrived, or disallow). Deposit's `transferFrom` also requires the
  maker to approve this contract for each ERC-20-priced reference.
- **Redeem enumeration.** `redeem(token)` settles one currency per
  call, which keeps the contract from having to know every token it
  holds. The alternative — track the set of currencies a stratum has
  ever received so a single call can settle all of them — costs storage
  on every inbound transfer and is rejected for now; revisit if
  per-currency calls prove too unergonomic for clients.
- **ERC-20 tips.** A native tip arrives via `receive`. An ERC-20 tip is
  a plain `transfer` to the stratum address, which runs no code — fine
  for `redeem(token)` since it reads the live balance, but it emits no
  event from this contract, so tip attribution lives off-chain. Decide
  whether an explicit `tip(token, amount)` entrypoint is worth adding
  for indexability.
- **Reentrancy ordering.** With pushed payouts, ERC-20 transfer hooks
  (e.g. ERC-777), and balances read live from chain, the order of
  effects vs. external calls in `redeem` must be fixed (zero/snapshot
  the relevant accounting before pushing, or guard) so a reentrant
  `redeem` cannot double-spend.
