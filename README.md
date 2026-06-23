# Strata

> An Ethereum protocol of immutable strata — each deposited atop the strata it references, paying them, with value settling toward the foundations.

Strata is a substrate primitive built around a single object: the **stratum**. A stratum is a permanent, immutable cell that carries a small opaque payload and points to the strata laid down before it. You can only reference strata that already exist, so the strata form a directed acyclic graph ordered by time — deeper is older — and ETH flows along the same edges, settling downward toward the oldest, most-referenced strata: the foundations everything else is built on.

## How it works

**Deposit.** To lay down a new stratum referencing existing strata, the maker calls `deposit` with the references and an opaque payload, paying one *unit* — a fixed amount of ETH — for each reference. Each unit is credited to the stratum it points at. A stratum with no references is an **original**: it pays no one and costs nothing to be made. The maker becomes the stratum's `owner`.

**Tip.** Anyone can send ETH directly to a stratum. It simply adds to that stratum's balance — and, like everything a stratum holds, will settle downward toward its foundations on the next redemption.

**Redeem.** `redeem` is permissionless: anyone may call it on any stratum. It takes the stratum's current balance and divides it among the stratum's owner and its substrata. The owner's share leaves the system; each substratum's share is credited to that substratum, where it waits to be settled further down. No one needs incentive to call it — every stratum, and all of its descendants, are motivated by the value waiting below them.

Over many redemptions, value migrates along the references toward the foundations, each owner taking a share as it passes through. An original, having no substrata, pays its whole balance to its owner — originality is rewarded in full; derivative work pays tribute to its sources.

### A worked example

```
A   — an original. No references. redeem(A) pays everything to A's owner.
B   — references A. Making B pays one unit to A.
C   — references A and B. Making C pays one unit to A and one to B.

Someone tips C.
  redeem(C) splits C's balance among C's owner, A, and B.
  redeem(B) later splits B's balance between B's owner and A.
  redeem(A) pays only A's owner.

Value runs downhill, to the foundation.
```

Because value must exceed the cost of moving it, Strata is intended for a low-fee environment — an L2 or similar — where a unit can sit comfortably above gas.

## Architecture

Strata is a **self-cloning protofactory**. There is one contract, `Stratum`, which is at once the cell and the allocator: depositing clones a new `Stratum` as a minimal proxy and initializes it atomically in the same transaction, leaving no window to front-run initialization. Every reference is validated as a genuine stratum before the new one is laid down, which keeps the graph closed — strata only ever point at strata — and guarantees every reference target can itself receive and settle ETH. A stratum's payload and references are immutable; only its balance and the act of redemption move.

## The payload

Each stratum carries five fields, all **opaque to the contract** — stored, never interpreted:

- `addr` — an address
- `value` — a uint256
- `name`, `description`, `image` — a string triple

The contract treats these as inert data. Their meaning lives entirely in the clients that read them. (Note that the opaque `value` field is unrelated to a stratum's ETH balance; they are distinct.)

## One application: a public square

Read the payload as a post — `name`/`description`/`image` as handle or title, body, and media; references as the replies and quotes a post is built on; a direct tip as a like. Engaging with a post then *pays its author directly*: no intermediary, no attention sold to a third party, and reach that is funded rather than rented. It is structurally resistant to extraction — you can only earn from other people's genuine engagement, since shuffling value among strata you own is net-negative after gas.

The same primitive fits other shapes: citation graphs that pay foundational work, dependency graphs that pay upstream maintainers — anywhere derivative value should flow back to its sources. The public square is one application among many; the contract knows nothing of any of them.

## Interface

Indicative; signatures may change.

```solidity
// Lay down a new stratum; pays one unit per reference. Maker becomes owner.
function deposit(
    address[] calldata substrata,
    address addr,
    uint256 value,
    string calldata name,
    string calldata description,
    string calldata image
) external payable returns (address stratum);

// Permissionless: split this stratum's balance among its owner and substrata.
function redeem() external;

// Accept ETH (a tip) into this stratum's balance.
receive() external payable;

// Views
function substrata()   external view returns (address[] memory);
function owner()       external view returns (address);
function addr()        external view returns (address);
function value()       external view returns (uint256);
function name()        external view returns (string memory);
function description() external view returns (string memory);
function image()       external view returns (string memory);
```

## Status: design stage

The substrate is settled — immutable strata, references paid at deposit, permissionless redemption that settles value toward the foundations. Several parameters are still being decided:

- **Distribution curve.** The baseline divides a redeemed stratum's balance equally among its owner and substrata. Whether the owner instead takes a fixed share, and whether earlier-referenced substrata are weighted above later ones — and on what schedule — is open. Any non-flat weighting also implies a uniqueness constraint on the reference array.
- **Ownership.** The maker is the owner. Whether ownership — a stream of redemption income — can be transferred, and whether strata are represented as NFTs (the `name`/`description`/`image` triple is already the ERC-721 metadata shape), is undecided.
- **Reference enforcement.** References are validated as genuine strata; whether that is done by an on-chain registry or by bytecode attestation is an implementation choice.