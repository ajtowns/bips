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

The most immediate and practical benefit is that when there is a stale
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

The `staletip` feature data is a boolean, `prefers_blocks`, indicating
whether the node advertising the feature prefers to collect the full
block data associated with a stale tip (when `true`, encoded as `\x01`),
or just the header information (when `false`, encoded as `\x00`). For
future compatibility, nodes SHOULD ignore any additional feature data
that may be provided.

### The `staletip` message

The `staletip` message is defined as a message with the ASCII message type
`staletip` and a payload of:

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

The `headers` field is ordered from oldest (the header immediately
following the fork point) to newest (the stale tip itself).

Because stale tips are very rare, this BIP does not reserve a 1-byte
[BIP 324][BIP324] message type ID for the `staletip` message.

#### `CompressedHeader` Format

The `CompressedHeader` is a 48 byte structure equivalent to
regular block header serialization, that omits the previous block's
hash[^rat-compressedheader]. That is:

| Size | Name          | Type       | Description |
| ---- | ------------- | ---------- | ----------- |
|    4 | `version`     | `int32_t`  | Block version information
|   32 | `merkle_root` | `uint256`  | The Merkle root of the block's transactions
|    4 | `time`        | `uint32_t` | The block's timestamp
|    4 | `bits`        | `uint32_t` | The calculated difficulty target being used for this block
|    4 | `nonce`       | `uint32_t` | The nonce used to generate this block

### Sending `staletip` messages

Nodes implementing this BIP MAY send `staletip` messages to advertise
recent stale tips they are aware of. If so,

- Nodes SHOULD send `staletip` messages advertising recent stale tips
  that they are aware of to peers that support the `staletip` feature.
- `staletip` messages SHOULD NOT be sent to peers that have not indicated
  support for the `staletip` feature.
- Nodes SHOULD NOT advertise stale tips that violate header consensus rules
  (invalid version, invalid timestamps, invalid difficulty changes,
  insufficient proof of work).
- Nodes MUST NOT send `staletip` messages if they are not sure that the
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
  - As a consequence, nodes SHOULD respect the `prefers_blocks` setting.
    That is, if a peer sets `prefers_blocks` to `false`, stale tips SHOULD
    be relayed to that peer immediately, without waiting to obtain the
    block data.  And conversely, for a peer that sets `prefers_blocks` to
    `true`, a node that does attempt to obtain block data SHOULD defer
    sending the `staletip` message to that peer until it has obtained
    the block data.
- Nodes that have the block data corresponding to the tip block (and will
  provide that data to peers that request it) SHOULD set `have_block`
  as `true`. Nodes without the full block data, or that are unwilling
  to relay the data to peers, SHOULD set `have_block` as `false`.

### Receiving `staletip` messages

Nodes implementing this BIP MAY process `staletip` messages from peers
to gain more knowledge about stale tips. If so,

- Nodes SHOULD advertise support for the feature, by sending the `feature`
  message with the `staletip` feature id and feature data as defined above,
  prior to sending the `verack` message.
- Nodes SHOULD reject (ignore) `staletip` messages where the `fork_point` is
  not known, and MAY disconnect the sending peer if this occurs. (Note
  that it was specified above that the sending peer MUST be sure the
  receiver knows the `fork_point` block before sending a `staletip`
  message).
- Nodes SHOULD reject `staletip` messages when the `headers` vector
  is empty, and MAY disconnect the sending peer if this occurs.
- When processing a `staletip` message, nodes MUST ensure that a
  denial of service vector[^rat-denialofservice] is not created. One way
  of achieving this is to mirror the checks applied to the content of
  the `headers` message. However it is also possible to be more strict,
  requiring that:
   * the `fork_point` is a recent block[^rat-maxheight]
   * the number of entries in `headers` is small[^rat-maxforklen], and,
   * that the headers meet minimum proof-of-work thresholds[^rat-denialofservice].
- Nodes MAY ignore messages that violate denial of service checks, or MAY
  partially process headers until the limits are reached. Nodes SHOULD
  NOT disconnect or otherwise punish peers that send a message that
  exceeds the denial of service limits.
- After receiving a `staletip` message that passes any denial of
  service checks, nodes SHOULD reconstruct the block headers from the
  `CompressedHeader` encoding, validate the headers, and add any new
  valid headers to their block database.
- Nodes SHOULD ignore any headers found to be invalid, and SHOULD NOT disconnect
  or otherwise punish peers for relaying invalid headers[^rat-ignoreinvalid].
- If `have_block` is `true`, nodes that prefer to collect the full block
  data SHOULD request that data in the normal way (eg, by sending a
  `getdata` message).
- Nodes that receive a new stale tip SHOULD announce that tip to their
  peers.

#### Reconstructing headers

Headers may be reconstructed from a `staletip` message via the following
algorithm:

```
    std::vector<CBlockHeader> headers;
    uint256 prev_hash = staletip_msg.fork_point;
    for (const auto& ch : staletip_msg.headers) {
        headers.emplace_back({
            .nVersion = ch.version,
            .hashPrevBlock = prev_hash,
            .hashMerkleRoot = ch.merkle_root,
            .nTime = ch.time,
            .nBits = ch.bits,
            .nNonce = ch.nonce,
        });
        prev_hash = headers.back().GetHash();
    }
```

Note that headers are reconstructed in order, from oldest (closest to
the fork point), to newest (the stale tip itself).

## Considerations

### Privacy Considerations

- [To be written]

### Test Network Considerations

#### Signet

- [To be written: require block data, variant header validation]

#### Testnet3, Testnet4

- [To be written: require the stale tip to have meaningful proof of work, versus being a minimum difficulty block?]

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

- [To be added: example `staletip` messages]

## Copyright

This BIP is licensed under the 3-clause BSD license.

[BIP324]: https://github.com/bitcoin/bips/blob/master/bip-0324.md
[BIP434]: https://github.com/bitcoin/bips/blob/master/bip-0434.md

[^rat-compressedheader]: ... compressed headers rationale
[^rat-maxheight]: ... `MAX_HEIGHT_DELTA` rationale
[^rat-maxforklen]: ... `MAX_FORK_LENGTH` rationale
[^rat-denialofservice]: ... min pow rationale, and avoiding header spam
[^rat-ignoreinvalid]: ... why ignore invalid messages instead of punishing?
