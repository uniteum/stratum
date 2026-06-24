# Strata

> An Ethereum protocol of immutable strata — each deposited atop the strata it references, paying them, with value settling toward the foundations.

Strata is a substrate primitive built around a single object: the **stratum**. A stratum is a permanent, immutable cell that carries a small opaque payload and points to the strata laid down before it. You can only reference strata that already exist, so the strata form a directed acyclic graph ordered by time — deeper is older — and value flows along the same edges, settling downward toward the oldest, most-referenced strata: the foundations everything else is built on.

## How it works

**Deposit.** To lay down a new stratum referencing existing strata, the maker calls `deposit` with the references, an opaque payload, and the **price** others must later pay to reference the new stratum — a currency and an amount. Every stratum sets its own price for inclusion as a substratum: to reference a stratum, the maker pays it its demanded amount in its demanded currency, credited to it. The currency is an address — `0x0` means the chain's native currency, any other address an ERC-20 token — so one deposit may pay its references in several different currencies at once. A stratum with no references is an **original**: it pays no one and costs nothing to be made, though it still sets a price for others to reference it. The maker (`msg.sender`) becomes the stratum's `owner`.

**Tip.** Anyone can send funds directly to a stratum — the native currency or any ERC-20. It simply adds to that stratum's balance in that currency — and, like everything a stratum holds, will settle downward toward its foundations on the next redemption.

**Redeem.** `redeem` is permissionless: anyone may call it on any stratum, for a chosen currency. It takes the stratum's current balance in that currency and divides it among the stratum's owner and its substrata. The owner's share leaves the system; each substratum's share is credited to that substratum, where it waits to be settled further down. A stratum may hold several currencies at once, and each is redeemed independently. No one needs incentive to call it — every stratum, and all the strata beneath it, are motivated to draw down the value waiting above them, each taking its share as it passes.

Over many redemptions, value migrates along the references toward the foundations, each owner taking a share as it passes through. An original, having no substrata, pays its whole balance to its owner — originality is rewarded in full; derivative work pays tribute to its sources.

### A worked example

```
A   — an original, priced at 0.01 native. No references.
B   — references A; making B pays A its price (0.01 native). B prices itself at 5 ACME (an ERC-20).
C   — references A and B; making C pays A 0.01 native and B 5 ACME.

Someone tips C in native.
  redeem(C, native) splits C's native balance among C's owner, A, and B.
  redeem(B, native) later splits B's native balance between B's owner and A.
  redeem(A, native) pays only A's owner.

Value runs downhill, to the foundation — one currency at a time.
```

Because value must exceed the cost of moving it, Strata is intended for a low-fee environment — an L2 or similar — where a stratum's price can sit comfortably above gas.

## Architecture

Strata is a **self-cloning protofactory**. There is one contract, `Stratum`, which is at once the cell and the allocator: depositing clones a new `Stratum` as a minimal proxy and initializes it atomically in the same transaction, leaving no window to front-run initialization. Every reference is validated as a genuine stratum before the new one is laid down, which keeps the graph closed — strata only ever point at strata — and guarantees every reference target can itself receive and settle the funds paid into it. A stratum's payload and references are immutable; only its balance and the act of redemption move.

## The payload

Each stratum carries five fields, all **opaque to the contract** — stored, never interpreted:

- `addr` — an address
- `value` — a uint256
- `name`, `description`, `image` — a string triple

The contract treats these as inert data. Their meaning lives entirely in the clients that read them. (Note that the opaque `value` field is unrelated to both a stratum's balance and its price, below; the three are distinct.)

## The price

Separately from the opaque payload, each stratum carries two fields the contract **does** interpret, fixed at deposit and immutable thereafter:

- `token` — the currency required to reference this stratum: `0x0` for the chain's native currency, any other address for an ERC-20.
- `price` — the amount of that currency charged for inclusion as a substratum.

These are the only fields the contract reads to enforce payment. A `price` of zero makes a stratum free to reference.

## One application: a public square

Read the payload as a post — `name`/`description`/`image` as handle or title, body, and media; references as the replies and quotes a post is built on; a direct tip as a like. Engaging with a post then *pays its author directly*: no intermediary, no attention sold to a third party, and reach that is funded rather than rented. It is structurally resistant to extraction — you can only earn from other people's genuine engagement, since shuffling value among strata you own is net-negative after gas.

The same primitive fits other shapes: citation graphs that pay foundational work, dependency graphs that pay upstream maintainers — anywhere derivative value should flow back to its sources. The public square is one application among many; the contract knows nothing of any of them.

## Interface

Indicative; signatures may change.

```solidity
// Lay down a new stratum, paying each reference its own demanded price.
// Native-priced references are paid from msg.value; ERC-20-priced ones are
// pulled from the maker via transferFrom (approval required). token/price are
// the new stratum's own inclusion price. Maker becomes owner.
function deposit(
    address[] calldata substrata,
    address token,
    uint256 price,
    address addr,
    uint256 value,
    string calldata name,
    string calldata description,
    string calldata image
) external payable returns (address stratum);

// Permissionless: split this stratum's balance in `token` among its owner and
// substrata. token == 0x0 settles the native balance. Call once per currency.
function redeem(address token) external;

// Accept the native currency (a tip) into this stratum's balance. ERC-20 tips
// are made by transferring the token directly to the stratum's address.
receive() external payable;

// Views
function substrata()   external view returns (address[] memory);
function owner()       external view returns (address);
function token()       external view returns (address);
function price()       external view returns (uint256);
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