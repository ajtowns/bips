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

 * `https://github.com/ajtowns/bitcoin/tree/202601-staletips`

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
- Nodes SHOULD limit the number of stale tips they advertise, and in particular
  SHOULD NOT advertise stale tips that are not recent[^rat-maxheight],
  that diverge significantly from the active tip[^rat-maxforklen], or
  that do not meet minimum proof-of-work thresholds[^rat-denialofservice].
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

#### Active Tip Announcements

In order for peers to send you `staletip` messages, they must be able
to track which block is your active tip in order to correctly deduce if
you already know about the fork point corresponding to new stale tips.

Currently, many node implementations make a small optimisation in
their block announcement handling that conflicts with this requirement;
namely, they will not announce a block to a peer if they have already
processed an announcement for the same block from that peer at the time
they validate the block. That said, due to delays in obtaining block data
and validating the block, it is usually the case that two well-connected
peers will announce the same new block to each other.

Nodes implementing this BIP SHOULD always announce their new active tip
to all peers. To minimise additional bandwidth, they MAY do so via an
`inv` message (rather than a `headers` or compact block message), however.

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

### Optionality

Node software implementing this BIP SHOULD provide a configuration option
to disable it entirely.

### Test Networks

Making this feature available on test networks raises additional concerns,
as they are less protected by proof of work, and thus may have a larger
attack surface for denial of service issues. As a result, node software
implementing this BIP may choose to do so only for mainnet.

#### Signet

The signet network is designed around the assumption that all valid
blocks are signed, and thus much lower proof of work is needed. However,
because the block signatures are included in the coinbase transaction,
it cannot be verified if only the block's headers are available. As such,
when implementing this BIP for signet:

 * Nodes MUST NOT send `staletip` messages when they do not have the
   corresponding block.
 * Nodes MAY disconnect peers that send a `staletip` message with
   `have_block` set to `false`.
 * Nodes SHOULD set `prefers_blocks` to true when negotiating the feature,
   as they will always need to download the full block for a stale tip
   to validate the signature.

In addition, when the difficulty is low, it is possible that varying the
`nBits` field of a valid block will generate another valid block (see
[#33266][#33266]). As such,

 * Nodes SHOULD consider variant headers (where the previous block and
   merkle root are both the same) to be duplicates, and only advertise
   the first seen variant as a stale tip.

While nodes MAY also deduplicate such variant headers when receiving them
via `staletip` messages, if they do not perform the same deduplication
logic when receiving such headers via standard `headers` or `block`
relay, that provides little protection. Deduplicating when sending `staletip`
messages is primarily aimed at avoiding nodes implementing this BIP from
being used to amplify attacks.

Only variant tips need to be deduplicated: it is possible (though
unlikely) that valid, signed blocks may be descendants of variant
blocks. These blocks will not themselves be variants, as they will have
different previous blocks.

#### Testnet3, Testnet4

While testnet3 and testnet4 blocks are proof of work based like mainnet,
they both allow for low difficulty blocks:

 - testnet3 allows minimum difficulty blocks when a block's timestamp
   is 20 minutes after the previous block's timestamp, and resets the
   difficulty on an ongoing basis if this occurs for the last block in
   a retarget period.
 - testnet4 updates these rules (as specified in [BIP 94][BIP94]), basing
   the difficulty calculation for the first block in a new retarget period
   on the difficulty of the first block in the previous retarget period,
   rather than the previous block, avoiding a reset.

As a result, nodes could see many valid stale tips with minimum difficulty,
either due to such tips being created with long timestamps, or, on testnet3,
because the difficulty has been reset. To avoid this scenario, when
implementing this BIP on testnet3 or testnet4:

 * Nodes SHOULD only advertise stale tips when the stale tip itself has
   a difficulty greater than the minimum difficulty.
 * Nodes MAY advertise stale tips only when the stale tip itself has
   a difficulty greater than some higher threshold (eg 1,000,000).

## Backward Compatibility

This BIP introduces a new P2P message (`staletip`) and relies on [BIP
434][BIP434] for feature negotiation to avoid impacting nodes that do
not support this BIP. Nodes that do not support this BIP will at most
see more frequent notifications of when their peers update to a new
tip that they have already announced.

## Privacy Impact

Sharing stale tips raises two potential concerns related to privacy.

The first concern is related to fingerprinting, ie that a node's
behaviour when sharing stale tips may make it easier for peers to
distinguish that node from others on the network, perhaps allowing
attackers to relate a node's onion address with its IPv4 address, for
instance. Since it is expected that stale tips will propagate fairly
efficiently and consistently (meaning that behaviour differences will
be small), and that they will remain exceedingly rare (meaning that it
will be hard to observe patterns of behaviour), this is not expected to
be a significant problem in practice. For users that remain concerned,
disabling stale tip relay entirely is probably the best approach.

The second concern is for miners with private transaction pools. In
that case, miners have a selection of transactions that they wish to
mine themselves without giving other miners the opportunity to compete
with them, only publishing them when they construct a block with
valid proof-of-work that includes them. Relaying those transactions
via `staletip` announcements will largely be harmful to a private
transaction pool operator, as it will not help them win the stale block
race (it will only allow nodes to reorg slightly faster in the event that
they do win), but if they lose the stale block race it will make their
private transactions available to competing miners. This is a particular
concern if the reason for the block being stale is due to inefficiencies
in updating ASIC miners to a new work target -- in which case there
is almost no chance of winning the stale block race. As such, miners
operating private transaction pools should probably disable stale tip
relay entirely on the nodes they use for processing their mined blocks.

## Reference Implementation

- Bitcoin Core branch: https://github.com/ajtowns/bitcoin/tree/202601-staletips
- Key files: `staletips.h`, `staletips.cpp`; also `net_processing.cpp` changes

## Test Vectors

### Signet Stale Branch (19 blocks)

This test vector is from signet, where a 19-block stale branch occurred at height 287767-287785.

| Field             | Value |
| ----------------- | ----- |
| Network           | Signet |
| Fork point height | 287766 |
| Fork point hash   | `00000012602fde2eaf33a90523f42fb07ca854c1d26108782dc4592a80507e1c` |
| Stale tip height  | 287785 |
| Stale tip hash    | `0000000024ff924ff932668d497bba7da9157559a68d9c87d2f28d22e5e4a001` |
| Branch length     | 19 |
| `have_block`      | `false` |

Serialized `staletip` payload (946 bytes):

```
1c7e50802a59c42d780861d2c154a87cb02ff42305a933af2ede2f60120000001300000020
67aa2937b235e209f1ad40f37b17dcad8ca02d1fb9cf4c4928902eab3ff853a6cda16e6916
58151d81ad7b0400000020721ab6f36042842bab4583f98e15181b646c47c7b9dfb07a204c
4e5152e54e97fda46e691658151dd2dcec1800000020ddc6b628101b404f78d446591a623e
12bb7e7cd2ae1593d0cc6ef082e19fa81014a66e691658151d46070e10000000208be5b960
7b20ac0163b885881aa6018ef60a5637b93a9cd85981515a169a31c36aab6e691658151de4
e10600000000206c974ef806b2e6a21cc9a53fedcbd5d5e203b4929a1bd7c5859ec77d0a75
5c1d64ae6e691658151d2a2e130e0000002039505fa2ee91f5e5b2540b4a9a7f02ed58d79f
c387412cf2520a39ee16bf435f6eb16e691658151db29c830a00000020411bce69cda2b239
c74faea1f0cfaceabd90377114dc0ace14e2a58858cd4f9a91b46e691658151d4c2d3f0500
00002096464a2ec42fea8419a5c6adcc41ca7dc3f53461cd0afc880f76498aa8eedbf993b5
6e691658151d5e99700b00000020c3cad611ee3c7bffec3f80fc83f68192ca0a86e8c89050
f8353480310d080caf08b76e691658151dbc61a6150000002079e36b5a06bb5a524a2f5bad
6725451613c354cbf64f9a0283f77fe90782d59fcbba6e691658151d6c669300000000204c
69fd467818994e7ff885181e7451d23869a7fd19447dad7a95398077ed7774dcba6e691658
151dc9c7ff02000000208ea9a871b12cd45226ab6df280ec46b8c9ca196b98dc8adc27718c
ab333e19a50abc6e691658151df6bae902000000209c182979424a5c85d55d5bddbf3b9421
b01b7810edf95ad2529552dd2eec80848ebd6e691658151d439e5c0000000020cfb9f6d164
3613dfdd23a6e1eabadac38c392f985f5dc05b00ef682c428922c96cbe6e691658151d278b
4d1400000020eea983b8d64947368de4f7626531a3134d18f20a8d87462ba011a4f56a53b6
ac4fbf6e691658151ddae8640800000020140bc70b1a3282fbebf9d48ff40a9b73ab7c76ad
522799cee73315e9c21aab50dcc06e691658151dedaae90f000000209b91a056fe089429f2
ba4298e764acb4177249ce7d68e6883836f70d1d9218b355c16e691658151d28a432060000
0020b26e80c6a12cc4595556d2318fda564ced5d2a526c7b77f57a9897a56c10539056c16e
691658151dc64fb50000000020369d36495749c96d36260d839abfa703a611aad7b3016342
ef05d0412f42221e6ac56e691658151d8f8bfb0f00
```

Payload breakdown:
- Bytes 0-31: `fork_point` (uint256, little-endian)
- Byte 32: vector length (0x13 = 19)
- Bytes 33-944: 19 `CompressedHeader` (48 bytes each)
- Byte 945: `have_block` (0x00 = `false`)

### BCH Fork Headers (18 blocks)

This test vector uses mainnet headers from the BCH fork. Blocks 478559-478576
have valid Bitcoin headers (meeting proof-of-work requirements), but are
invalid Bitcoin blocks (block 478559 exceeds the 1MB block size limit).
Block 478577 changed difficulty under BCH's Emergency Difficulty Adjustment
rules, making it an invalid Bitcoin header as well.

| Field             | Value |
| ----------------- | ----- |
| Network           | Mainnet |
| Fork point height | 478558 |
| Fork point hash   | `0000000000000000011865af4122fe3b144e2cbeea86142e8ff2fb4107352d43` |
| Stale tip height  | 478576 |
| Stale tip hash    | `000000000000000001416af072f8989829f4c60a1a9658e1cec08411798e4ffa` |
| Branch length     | 18 |
| `have_block`      | `false` |

Serialized `staletip` payload (898 bytes):

```
432d350741fbf28f2e1486eabe2c4e143bfe2241af65180100000000000000001200000020
abaa4bd8a48c1c6bc08ee39b66065e5e9484304cab8b56d5eed3e40b1ac996c899c4805935
47011822ca4ae80000002082afc8ef7eb41a4ecac1fea46983742e491f804ad662e3745ab9
c6c4297d8a0862c980593547011840a772cb0200002058874e50628fdf83aeea4e8cbc7ade
946e9ba14bcb1d8ffb28c3daf8ade84df65fca805935470118e2f5100300000020111b85f9
d3b969a1f7ff3d50af08893c500edfc5623b96dbeab6daf16a5164a40ace805935470118c4
f4240a0000002040a045063b551b61d6a1c9db6d3231e2d7403185bbb2332ae1f66db24aac
7fa288d8805935470118f15dd76b0000002070cb14529e8757c359c2e8b1e987f6eee6fbc4
472ee9ad4a2e5df6905c19d6d70bed80593547011885ae00d0000000209653314c1d73e463
0bb485fb25ce7a2583cec7c3ccfc27a6d24163be1e9fb19530f4805935470118f17ad2c500
000020cdf48b8e7ac6bf3a51d1878ee3ff7e6fd0022926dd69cc5cc8d9126e77c4dba809f5
8059354701188114e83600000020890cf1dc60edbf0fd4fb667f28ac785849c031d8b24d5e
5a0af56ee2bd8a739bf51081593547011812f32a9600000020ff5244613ad20fdc39b7ee6f
4fbc7016432d2dbf45c2a950c59665b39c3954b5b525815935470118aa790d6600000020ae
ed520e7c1693de5cfe7531e7d3e73dff7858b09cb6e1ec29229a75c3da2b92453e81593547
011830a4314d000000202e4a4054e64c3f5810c23ec0144d9793aab2d5a7d77d1660eee24d
3d55e8b715b543815935470118da1c00e00000002073152af68778a98fd984a158aeb29d28
e094e23c3a7dff02260c345791e52498c3fb8159354701188d5abed90000002022606e744a
29f9d4a67ff1fcd2f0e31300ddbd145f8f1db8a68270bfbde77dd88fff8159354701186af1
75f8000000209f5db27969fecc0ef71503279069b2df981ba545592a7b425f353b5060e77f
3e7e13825935470118da70378e000000202f0d316b08350f5cd998c6a11762d10adb9f951b
5f79ce2a073f8187c05f561f1b1c8259354701184834c62300000020cf8fc3bad8dad139a3
dd6a30481d87e1f760122573168002cc9ef7a58fc53ad387848259354701188a3b54f70000
00200eae92d9b46d81a011a79726a802d4eb195a7af8b70a09b0e115c391968c50d51c8a82
5935470118cd786d1300
```

Payload breakdown:
- Bytes 0-31: `fork_point` (uint256, little-endian)
- Byte 32: vector length (0x12 = 18)
- Bytes 33-896: 18 `CompressedHeader` (48 bytes each)
- Byte 897: `have_block` (0x00 = `false`)

## Copyright

This BIP is licensed under the 3-clause BSD license.

[BIP94]: https://github.com/bitcoin/bips/blob/master/bip-0094.mediawiki
[BIP324]: https://github.com/bitcoin/bips/blob/master/bip-0324.md
[BIP434]: https://github.com/bitcoin/bips/blob/master/bip-0434.md
[#33266]: https://github.com/bitcoin/bitcoin/issues/33266
[#19858]: https://github.com/bitcoin/bitcoin/pull/19858

[^rat-compressedheader]: Omitting the previous block hash from each header
    saves 32 bytes per header (40%), as this field can be reconstructed
    from the preceding headers in the message. This does not apply to
    the first header, of course, which is why the fork point must be
    included explicitly. We avoid attempting to omit `nBits` or compress
    `nTime` or `nVersion`, as reconstructing them is significantly more
    complicated, for comparatively much less potential gain.

[^rat-maxheight]: A window of 1000 blocks (about 7 days) provides a
    reasonable balance between limiting resource usage, and maximising
    propagation potential. By limiting the distance to the tip to being no
    older than the previous retarget period, we ensure that the difficulty
    of creating a stale block is comparable to creating a new tip, making
    it uneconomical to use this as a spamming mechanism. By choosing
    a longer period, we provide the opportunity for nodes interested in
    stale block data for research purposes plenty of opportunity to obtain
    the data. In particular, many nodes will tend to discover it through
    the periodic blocks-only connections (see Bitcoin Core PR [#19858]).

[^rat-maxforklen]: Limiting the branch length to 20 headers avoids nodes
    being burdened with tracking deliberate chain splits. When
    incompatible consensus rules are enforced, the majority chain need not
    continue sharing the minority's fork beyond a short window. Twenty
    blocks is sufficient to capture the short-term stale blocks useful
    for reorg detection, while avoiding long-term tracking of persistent
    chain splits.

[^rat-denialofservice]: Limiting stale branches to 20 headers bounds the
    per-message size to under 1kB. Combining this with a limit of 10 tracked
    stale tips, this ensures that total bandwidth when relaying all known
    stale tips to a new peer is under 10kB.

[^rat-ignoreinvalid]: Ignoring rather than punishing allows nodes to apply
    different limits for what stale tips they accept. If one node
    uses stricter limits than its peers, this avoids the risk that
    honest announcements from those peers will cause disconnection and
    potentially network partitions.  This is also consistent with how
    invalid headers are handled.
