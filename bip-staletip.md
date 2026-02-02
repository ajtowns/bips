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

### STALETIP Message Format

- `hash_fork_point`: uint256 - block where stale chain forks from active chain
- `headers`: CompressedBlockHeader[] - headers from fork point (exclusive) to tip (inclusive)
- `have_block`: bool - whether sender has transaction data for tip

#### CompressedBlockHeader Format

- First header: full 80 bytes
- Subsequent headers: omit `prev_block` (implied), varint delta for timestamp

### STALETIP Feature Negotiation

- `staletip` message SHOULD NOT be sent unless the peer has indicated support for
  the STALETIP feature.
- Feature ID: `https://github.com/ajtowns/bitcoin/tree/202601-staleblocks`
- Feature data: 1 byte (0x00 = headers mode, 0x01 = blocks mode)

### Sending STALETIP Messages

- Node becomes aware of new stale tip not previously sent to this peer
- Tip meets eligibility criteria
- Peer's mode preference (headers vs blocks) is satisfied
- Fork point:
   - is an ancestor of the earliest, highest work block the peer has advertised
   - has at least minimum-chain-work
- Stale tip should be recent (eg with 1000 blocks of the main tip)
- Fork length should be small

### Receiving STALETIP Messages

- Validate `hash_fork_point` is known block
- Optional: validate headers chain is short and recent
- Validate headers chain from fork point
- Add valid headers to db
- Cache stale tip, and advertise it to other peers
- Optional: request block data, if `have_block` was indicated

## Considerations

### DoS Considerations

- [To be written]

### Privacy Considerations

- [To be written]

### Signet Considerations

- [To be written: require block data, variant header validation]

### Using Stale Tip Information

- [To be written]

### Backward Compatibility

This BIP introduces a new P2P message (`staletip`) and relies on [BIP 434][BIP434]
for feature negotiation. Nodes that do not implement this BIP will simply not
send the corresponding `feature` message, and will therefore not receive
`staletip` messages from peers that do implement it.

Nodes implementing this BIP will not send `staletip` messages to peers that
have not indicated support via the feature negotiation mechanism.

There is no impact on consensus rules or existing P2P message handling.

## Rationale

### Feature ID

### Compressed headers

### `MAX_HEIGHT_DELTA`

### `MAX_FORK_LENGTH`

## Reference Implementation

- Bitcoin Core branch: [link to branch]
- Key files: staletips.h, staletips.cpp, net_processing.cpp changes

## Test Vectors

- [To be added: example STALETIP messages]
- [To be added: example CompressedBlockHeader encoding]

## Copyright

This BIP is licensed under the 3-clause BSD license.

[BIP434]: https://github.com/bitcoin/bips/blob/master/bip-0434.md
