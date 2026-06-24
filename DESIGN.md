# Strata — design notes

The [README](./README.md) is orientation: what a stratum is and how
value settles toward the foundations. This document records the
contract-level decisions the README leaves implicit, and the questions
still open at the design stage.

## Resolved semantics

These are settled for the first implementation. They are not yet
reflected in the prose of the README because they live below its
altitude — they are caller-correctness detail, not orientation.

### Balance

A stratum's balance is `address(this).balance` — each clone literally
holds its own ETH. A tip via `receive` and a substratum's share on
redemption both arrive as real ETH transfers into the contract; there
is no separate internal accounting variable. (For now: if accounting
later needs to diverge from on-chain balance, this is the line that
moves.)

### Deposit

- The **maker is `msg.sender`** — not `tx.origin` — and becomes the
  new stratum's `owner`.
- Deposit costs one unit per reference, each unit credited to the
  stratum it points at. **Overpayment is not refunded:** any
  `msg.value` beyond `unit × references` is retained as the new
  stratum's opening balance, where it later settles to that stratum's
  own owner and substrata on redemption.
- **Duplicate references are allowed.** `[A, A]` pays A two units at
  deposit and, on the new stratum's redemption, gives A two shares.
  Each entry in the reference array is treated independently. (Note:
  this holds only under the flat baseline; a non-flat distribution
  curve would require uniqueness — see Open questions.)

### Redeem

- The balance is divided among the owner and the substrata. Under the
  flat baseline that is `N + 1` equal shares for `N` substrata (the
  owner counts as one), matching the worked example in the README.
- **Dust goes to the owner.** Integer division leaves a remainder; the
  owner's share absorbs it.
- The owner's share is **pushed** to the owner address; substrata
  shares are likewise sent as ETH. A **revert by the owner is caught**
  — payouts to substrata still succeed, so a hostile or contract owner
  cannot brick a permissionless `redeem`. The uncaught case for the
  owner's own share (retained in the stratum for a later redeem vs.
  forfeited) is noted under Open questions.

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
- **The unit.** Fixed at deploy and immutable, or configurable? The
  README only says it is "a fixed amount of ETH" sized to sit above gas
  on an L2.
- **Reentrancy ordering.** With pushed payouts and balance read as
  `address(this).balance`, the order of effects vs. external calls in
  `redeem` must be fixed (zero the relevant accounting before pushing,
  or guard) so a reentrant `redeem` cannot double-spend.
