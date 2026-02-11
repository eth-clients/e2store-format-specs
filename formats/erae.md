# EraE files

 EraE files are themselves e2store files. For more information on this format,
 see https://github.com/status-im/nimbus-eth2/blob/stable/docs/e2store.md.

The format is designed to be compatible for both pre-merge execution layer data
and post-merge. It is intended to be a companion to the Era format.

## Specification

The format can be summarized with the following expression:

       eraE := Version | CompressedHeader+ | CompressedBody+ | CompressedReceipts+ | Proofs+ | TotalDifficulty* | other-entries* | Accumulator? | BlockIndex

Each basic element is its own e2store entry:

       Version            = { type: 0x6532, data: nil }
       CompressedHeader   = { type: 0x03,   data: snappyFramed(rlp(header)) }
       CompressedBody     = { type: 0x04,   data: snappyFramed(rlp(body)) }
       CompressedSlimReceipts = { type: 0x0a,   data: snappyFramed(rlp([tx-type, post-state-or-status, cumulative-gas, logs])) }
       TotalDifficulty    = { type: 0x06,   data: uint256(header.total_difficulty) }
       Proof              = { type: 0x0b    data: snappyFramed(rlp([proof-type, ssz(BlockProofHistoricalHashesAccumulator) | ssz(BlockProofHistoricalRoots) | ssz(BlockProofHistoricalSummaries)]))}
       AccumulatorRoot    = { type: 0x07,   data: hash_tree_root(List(HeaderRecord, 8192)) }
       Index              = { type: 0x6732, data: index }

A few notes on individual elements:

- `CompressedReceipts` is optional and a different format than the consensus EIP-2718 format to optimize internal use in clients.
- `Proof` is optional but if included, provides the coresponding proof for each `CompressedHeader` in the file corresponding to the Portal Network proofs specification[^1]. It's possible to have multiple proof types in the same file at fork boundaries.
- `TotalDifficulty` is optional and little-endian encoded.
- `AccumulatorRoot` is optional and only defined for epochs with pre-merge data.
- `HeaderRecord` is defined in the Portal Network specification[^2].
- `other-entries` is a placeholder for other potential objects to be added to EraE.

`BlockIndex` stores relative offsets to the components of each block entry. This allows `O(1)` access to all components of a block. The format is:

       block-index := starting-number | indexes | indexes | indexes ... | component-count | count

Where `indexes` represents the index of each block component:

       indexes := header-index | body-index | receipts-index | difficulty-index? | proof-index?
       
The value `component-count` is the number of indexes stored by `indexes`. It should be in the range of 3-5, depending on whether `TotalyDifficulty` and `Proofs` are present 

All values in the block index are little-endian `uint64`.

`starting-number` is the first block number in the archive. Every index is a defined relative to index's location in the file. The total number of block entries in the file is recorded in count.

Due to the accumulator size limit of 8192, the maximum number of blocks in an Era batch is also 8192. This is also the value of `SLOTS_PER_HISTORICAL_ROOT`[^3] on the Beacon chain, so it is nice to align on the value.

### Merge transition

There are some small differences between pre-merge and post-merge `eraE` files:
- `TotalDifficulty` should only be encoded for `eraE` files pre-merge. For the epoch where the merge occurs, fill all remaining post-merge blocks with the final total difficulty of the chain.
- `AccumulatorRoot` should only be encoded for `eraE` files pre-merge. For the epoch where the merge occurs, compute the root for an incomplete epoch where only pre-merge blocks are recorded in the accumulator. 

[^1]: https://github.com/ethereum/portal-network-specs/blob/master/history/history-network.md#block-header
[^2]: https://github.com/ethereum/portal-network-specs/blob/master/history/history-network.md#the-historical-hashes-accumulator
[^3]: https://github.com/ethereum/consensus-specs/blob/44ae6e661d9beac383f4a1f33be74259bae93c85/presets/mainnet/phase0.yaml#L42
