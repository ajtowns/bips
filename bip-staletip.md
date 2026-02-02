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

The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in RFC 2119.

### Staletip Feature Definition

This BIP defines a new [BIP 434][BIP434] feature id ("the `staletip`
feature"):

 * `https://github.com/ajtowns/bitcoin/tree/202601-staleblocks`

The `staletip` feature data is a single boolean, `prefers_blocks`,
indicating whether the node advertising the feature prefers to collect
the full block data associated with a stale tip (when `true`), or just
the header information (when `false`). For future compatibility, nodes
SHOULD ignore any additional feature data that may be provided.

### The `staletip` message

The `staletip` message is defined as a message with `pchCommand == "staletip"`
and a payload of:

| Type      | Name         | Description |
| --------- | ------------ | ----------- |
| `uint256` | `fork_point` | The hash of the block where the stale chain forks from the active chain |
| vector of `CompressedHeader` | `headers` | The headers after the fork point up to the tip |
| `bool`    | `have_block` | Whether the sender has the transaction data for the tip |

Vector serialization is as normal -- encode the length of the vector
as a 1, 3, 5 or 9 byte `CompactSize`, then encode each member of the
vector. Only minimally-encoded `CompactSize` values are supported. The
`bool` serialization is a single byte `\x00` for false, and a single byte
`\x01` for true.

#### `CompressedHeader` Format

The `CompressedHeader` is a 48 byte structure equivalent to
regular block header serialization, that omits the previous block's
hash[^rat-compressedheader]. That is:

| Size | Name          | Type       | Description |
| ---- | ------------- | ---------- | ----------- |
|    4 | `version`     | `int32_t`  | Block version information
|   32 | `merkle_root` | `uint256`  | The Merkle root of the block's transactions
|    4 | `timestamp`   | `uint32_t` | The block's timestamp
|    4 | `bits`        | `uint32_t` | The calculated difficulty target being used for this block
|    4 | `nonce`       | `uint32_t` | The nonce used to generate this block

### Sending `staletip` messages

Nodes implementing this BIP MAY send `staletip` messages to advertise
recent stale tips they are aware of. If so,

- `staletip` messages SHOULD NOT be sent unless the peer has indicated
  support for the `staletip` feature.
- Nodes MUST NOT advertise stale tips that violate header consensus rules
  (invalid version, invalid timestamps, invalid difficulty changes,
  insufficient proof of work).
- Nodes SHOULD send `staletip` messages to their peers about recent
  stale tips that they are aware of.
- Nodes SHOULD NOT send `staletip` messages if they are not sure that the
  `fork_point` is a block that their peer is aware of. This can be
  estimated by tracking the highest work blocks announced by the peer,
  and assuming the peer is treating that block as its active tip. Provided
  the `fork_point` is an ancestor of that highest work block, the peer
  will be aware of the `fork_point`.
- The `fork_point` SHOULD be chosen to minimise the number of entries
  in the `headers` vector, while still being a block that the peer already
  has header information for.
- Nodes MAY choose a `fork_point` based on their own active chain,
  even if that results in the `headers` vector repeating some headers
  that the peer already knows.
- Nodes SHOULD NOT advertise stale tips that are not
  recent[^rat-maxheight], that diverge significantly from the active
  tip[^rat-maxforklen], or that do not meet minimum proof-of-work
  thresholds[^rat-denialofservice].
- Nodes SHOULD avoid advertising the same tip to the same peer repeatedly
  via multiple `staletip` messages.
  - As a consequence, nodes SHOULD respect the `prefers_blocks` settings;
    that is, if a peer sets `prefers_blocks` to `false`, it SHOULD relay
    stale headers immediately without waiting to obtain the block data,
    and conversely, for a peer that sets `prefers_blocks` to `true`, a
    node that does attempt to obtain block data SHOULD defer sending the
    `staletip` message to that peer until it has obtained the block data.

### Receiving STALETIP Messages

- Validate `fork_point` is known block
- Optional: validate headers chain is short and recent
- Validate headers chain from fork point
- Add valid headers to db
- Cache stale tip, and advertise it to other peers
- Optional: request block data, if `have_block` was indicated

## Considerations

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

## Reference Implementation

- Bitcoin Core branch: [link to branch]
- Key files: staletips.h, staletips.cpp, net_processing.cpp changes

## Test Vectors

- [To be added: example STALETIP messages]
- [To be added: example CompressedBlockHeader encoding]

## Copyright

This BIP is licensed under the 3-clause BSD license.

[BIP434]: https://github.com/bitcoin/bips/blob/master/bip-0434.md

[^rat-compressedheader]: ... compressed headers rationale
[^rat-maxheight]: ... `MAX_HEIGHT_DELTA` rationale
[^rat-maxforklen]: ... `MAX_FORK_LENGTH` rationale
[^rat-denialofservice]: ... min pow rationale, and avoiding header spam
