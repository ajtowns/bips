```
  BIP: ?
  Layer: Peer Services
  Title: Stale Tip Relay
  Authors: Anthony Towns <aj@erisian.com.au>
  Status: Draft
  Type: Specification
  Assigned: ?
  License: BSD-3-Clause
  Requires: 434
```

## Abstract

Bitcoin miners sporadically produce stale blocks (that is, new blocks that
don't improve on the cumulative proof of work of the current tip). This
BIP defines a peer-to-peer (P2P) message that can be used to announce
recent stale block headers (and the availability of the transaction
content of those stale blocks) to peers.

## Motivation

Being aware of stale blocks can be useful in ensuring the health of the
Bitcoin network.

The most immediate and practical benefit, is that when there is a stale
block with the same cumulative proof of work as the current active tip,
there is the potential for the current active tip to be reorged
out in favour of a child of the stale block. In this case, having the
stale block already downloaded (and possibly already validated) will
make dealing with the reorg faster.

A more long term benefit of tracking stale blocks is that it allows for
an indirect measurement of the efficiency with which miners are able to
update to a new tip (in that slower updates to new tips will result in
more stale blocks), and a measure of the differences in block creation
policy between mining pools (in that the contents of stale blocks will
tend to show how similar or different the mining pools' mempools are at
the point in time that the blocks were found).

These benefits are not essential to the operation of the Bitcoin network,
so this is proposed as an optional feature, with low performance demands.

## Specification

### Feature Negotiation

- Feature ID: URL to development branch (currently "https://github.com/ajtowns/bitcoin/tree/202601-staleblocks")
- Feature data: 1 byte boolean (prefer_blocks)
  - true: only send tips where we have full block data
  - false: send tips when headers are known

### STALETIP Message Format

- hash_fork_point: uint256 - hash of last common block on active chain
- headers: vector of CompressedBlockHeader
- have_block: boolean - whether sender has full block data for tip

### CompressedBlockHeader Format

- Compact encoding of block headers in a chain
- Uses varint deltas for timestamps
- First header: full 80 bytes
- Subsequent headers: omit prev_block (implied from chain)

### Eligibility Criteria

- Fork point must be on active chain
- Fork point must have sufficient work (not pre-minimum-chain-work)
- Tip height >= active_tip_height - MAX_HEIGHT_DELTA (1000)
- Fork length <= MAX_FORK_LENGTH (20 blocks)
- Signet: must have full block data (not headers-only)

### Node Behavior

#### Receiving STALETIP

- Validate fork point exists and is on active chain
- Validate headers chain correctly from fork point
- If tip already known: ignore (or request blocks if missing)
- If valid new tip: add to cache, optionally request block data

#### Sending STALETIP

- Track stale tips in cache
- Announce new tips to peers who sent feature message
- Respect peer's prefer_blocks setting
- Don't re-announce tips peer already knows

### Operational Modes (-staletips option)

- none: disabled, don't send or relay
- headers: announce when headers known
- blocks: announce only after full block data received

## Rationale

### Why compressed headers?

- Reduces message size for multi-block forks
- Previous block hash is redundant (implied by chain)

### Why MAX_HEIGHT_DELTA of 1000?

- Balances usefulness vs resource usage
- Tips older than ~1 week unlikely to be interesting
- [need to verify reasoning from code]

### Why MAX_FORK_LENGTH of 20?

- Longer forks are extremely rare in practice
- Limits resource usage for validation/storage
- [need to verify reasoning from code]

### Why require block data on signet?

- Signet uses signed blocks with variant headers
- Can't fully validate header without block data
- Prevents invalid tips from being relayed

### Privacy considerations

- Feature enabled by default is not identifying (widespread deployment)
- Timing of announcements: natural network latency provides variation
- Stale blocks rare (~1.2/week): insufficient for timing correlation
- Generating stale blocks for fingerprinting prohibitively expensive (~$250k)
- Recommendation: nodes with intermittent connectivity should use -staletips=none
  - Reconnecting node might announce "old" tips, revealing downtime pattern

## Backward Compatibility

This BIP introduces a new P2P message (`staletip`) and relies on [BIP 434][BIP434]
for feature negotiation. Nodes that do not implement this BIP will simply not
send the corresponding `feature` message, and will therefore not receive
`staletip` messages from peers that do implement it.

Nodes implementing this BIP will not send `staletip` messages to peers that
have not indicated support via the feature negotiation mechanism.

There is no impact on consensus rules or existing P2P message handling.

## Reference Implementation

- Bitcoin Core branch: [link to branch]
- Key files: staletips.h, staletips.cpp, net_processing.cpp changes

## Test Vectors

- [To be added: example STALETIP messages]
- [To be added: example CompressedBlockHeader encoding]

## Copyright

This BIP is licensed under the 3-clause BSD license.

[BIP434]: https://github.com/bitcoin/bips/blob/master/bip-0434.md
