# Erc files

`.erc` files are special instances of `.e2s` files that follow a more strict content format optimised for reading and long-term storage and distribution.

The blocks stored are *blinded* - the execution part of the block is replaced with a root of the execution payload, at the cost of not containing the execution layer history. The execution layer data referenced by the payloads of an era is available from [Ere files](ere.md) - note that ere files are numbered by execution block number, not by era, so the payloads of a given era may span one or more ere files, and pre-merge eras have no corresponding execution blocks at all.

Erc files contain groups consisting of a state and the blinded blocks that led up to it, limited to `SLOTS_PER_HISTORICAL_ROOT` slots each.

In examples, we assume the mainnet configuration: `SLOTS_PER_HISTORICAL_ROOT == 8192`.

Each era is identified by when it ends. Thus, the genesis era is era `0`, followed by era `1` which ends when slot `8192` has been processed.

## File name

`.erc` file names follow a simple convention: `<config-name>-<era-number>-<short-era-root>.erc`:

* `config-name` is the `CONFIG_NAME` field of the runtime configuration (`mainnet`, `prater`, `ropsten`, `sepolia`, etc)
* `era-number` is the number of the _first_ era stored in the file - for example, the genesis era file has number 0 - as a 5-digit 0-filled decimal integer
* `short-era-root` is the first 4 bytes of the appropriate root, lower-case hex-encoded (8 characters):
    * For genesis era: use `state.genesis_validators_root` field from the genesis state
    * For pre-Capella forks: use the last historical root from the last state in the era file (`state.historical_roots[era - 1]`)
    * For post-Capella forks: use the last historical summary from the last state in the era file (`state.historical_summaries[era - 1 - state.historical_roots.len()]`)

Erc files with multiple eras use the era number of the lowest era stored in the file, and the root of the highest era.

An erc file containing the mainnet genesis is thus named `mainnet-00000-4b363db9.erc`, and the era after that `mainnet-00001-40cf2f3c.erc`.

## Structure

An `.erc` file is structured in the following way:

```
erc := group+
group := Version | block* | era-state | other-entries* | slot-index(block)? | slot-index(state)
block := CompressedSignedBlindedBeaconBlock
era-state := CompressedBeaconState
```

The `block` entries of a group include all blinded blocks leading up to the era transition in slot order. For example, the group representing era `1` contains blocks from slot `0` up to and including block `8191`. Empty slots are skipped.

The `era-state` is the state in the era transition slot. The genesis group contains only the genesis state but no blocks. For example, the group representing era `1` contains the canonical state of slot `8192`.

`slot-index(block)` is a `SlotIndex` entry with `count = SLOTS_PER_HISTORICAL_ROOT` for the `CompressedSignedBlindedBeaconBlock` entries in that era, pointing out the offsets of each block in the era. It is omitted for the genesis era.

`slot-index(state)` is a `SlotIndex` entry with `count = 1` for the `CompressedBeaconState` entry of that era, pointing out the offset where the state entry begins.

`other-entries` is an extension point for future record types in the erc file. The positioning of these allows the indices to continue to be looked up from the back of the group.

The structure of the erc file gives it the following properties:

* the indices at the end are fixed-length: they can be used to discover the beginning of an era if the end of it is known
* the start slot field of the state slot index identifies which era the group pertains to
* the state in the erc file is the end state after having applied all the blocks in the era and, if applicable, the block at the first slot - the `block_roots` entries in the state can be used to verify the roots of the blinded blocks, since blinding does not change the root of the block - either to verify the integrity of the erc file or to quickly load block roots without computing them.
* each group in the erc file is a full, independent erc file - groups can freely be split and combined

## Blinded blocks

A blinded block is a beacon block with the execution payload replaced by the root of the payload. The exact encoding depends on the fork, derived from the slot of the block as with any other block entry:

* pre-Bellatrix forks have no execution payload, and the block is stored as-is - there is nothing to blind
* from Bellatrix to the fork before Gloas, `body.execution_payload` is replaced by `body.execution_payload_root: Root` - the `hash_tree_root` of the `ExecutionPayload`
* from Gloas onwards, the execution payload is not part of the beacon block at all, so the block is already blinded and is stored as-is

The entry is a signed blinded block:

```python
class SignedBlindedBeaconBlock(Container):
    message: BlindedBeaconBlock
    signature: BLSSignature
```

where `BlindedBeaconBlock` is the `BeaconBlock` of the corresponding fork with the modifications above.

Because SSZ merkleizes a container field to its 32-byte root, replacing `execution_payload` with a `Root` field holding `hash_tree_root(execution_payload)` does not change the merkleization of the block. The `hash_tree_root` of a blinded block is thus exactly the `hash_tree_root` of the full block it was blinded from - the same root that is stored in `state.block_roots` and signed by the proposer.

## Reading erc files

```python
def read_erc_file(name):
  # Print contents of an erc file, backwards
  with open(name, "rb") as f:

    # Seek to end of file to figure out the indices of the state and blocks
    f.seek(0, 2)

    groups = 0
    while True:
      if f.tell() < 8:
        break

      (start_slot, state_index_start, state_slot_offsets) = read_slot_index(f)

      print(
        "State slot:", start_slot,
        "state index start:", state_index_start,
        "offsets", state_slot_offsets)

      # The start of the state index record is the end of the block index record, if any
      f.seek(state_index_start)

      # This can underflow! Python should complain when seeking - ymmv
      prev_group = state_index_start + state_slot_offsets[0] - 8
      if start_slot > 0:
        (block_slot, block_index_start, block_slot_offsets) = read_slot_index(f)

        print(
          "Block start slot:", block_slot,
          "block index start:", block_index_start,
          "offsets", len(block_slot_offsets))

        if any((x for x in block_slot_offsets if x != 0)):
          # This can underflow! Python should complain when seeking - ymmv
          prev_group = block_index_start + [x for x in block_slot_offsets if x != 0][0] - 8

      print("Previous group starts at:", prev_group)
      # The beginning of the first block (or the state, if there are no blocks)
      # is the end of the previous group
      f.seek(prev_group) # Skip header

      groups += 1
    print("Groups in file:", groups)
```

## Verifying erc files

To verify the internal consistency of an erc file, the following checks should be made to verify that an erc file is valid for a given network:

* each group follows the given structure of erc files with regards to blocks, states and their indices
  * offsets within indices must point to entries of the correct kind that can be decompressed and deserialized
  * erc file readers must be prepared to handle malicious inputs, including out-of-range offsets, invalid length prefixes and other trivial errors
  * unknown record types should be ignored, but it is recommended that verifiers report their size and tag
* the state is loadable and consistent with the given runtime configuration
* the root of each block in the erc file matches that of `state.block_roots` - since blinding does not change the root of the block, the root is computed from the blinded block contents as usual - if a slot is empty according to the block index, this should be confirmed by verifying that
  `state.get_block_root_at_slot(empty_slot - 1) == state.get_block_root_at_slot(empty_slot)` except for the first slot of the era which, if possible, should be verified against `era - 1`
  * the genesis erc file does not have any blocks
* the signature of each block can be verified by the keys in the given state (or any newer state).

The `execution_payload_root` field of blinded blocks cannot be verified against the era state alone - the state only commits to the payload of the latest slot. It can instead be verified against the execution layer data, for example by cross-checking it with the execution block from the [`.ere` file](ere.md) covering the corresponding execution block number.

Extended verification consists of verifying a list of erc files against a particular history anchored in a checkpoint or a head block. Verification starts from a well-known finalized checkpoint for a slot within the era, using `anchor_state_root = checkpoint_state.state_roots[0]` as anchor and walking the erc files as a linked list.

For each erc file:

* verify that `hash_tree_root(state) == anchor_state_root`
  * this anchors the era in a particular history, starting from the given state root - the state root is available from any state within the anchor era.
* verify the internal consistency of the era, as above
* set `anchor_state_root == state.state_roots[0]`

# FAQ

## Why blinded blocks at all?

The execution payload is part of the beacon block and takes up space, but it is not needed to follow the consensus layer history - the consensus state transition only commits to its root.

The Gloas fork has already moved the execution payload out of the beacon block, where it is delivered separately as an `ExecutionPayloadEnvelope`. Blinded blocks generalize this to the earlier forks, where the payload is still part of the block. Erc and [ere](ere.md) files thus cover the whole chain history together, for every fork: erc files store the consensus blocks, and ere files store the execution payloads they reference (numbered by execution block number, so an era's payloads may span several ere files).

## Why snappy (sz) framed compression?

* The networking protocol uses snappy framed compression, avoiding the need to re-compress data to serve blocks
* Each entry in the file can be decompressed independently (and partially!)
* It's fast and compresses decently

## Why `SLOTS_PER_HISTORICAL_ROOT` blocks per state?

The state stores the block root of the latest `SLOTS_PER_HISTORICAL_ROOT` blocks - storing one state per that many blocks allows verifying the integrity of the blocks easily against the given state, and ensures that all block and state root information remains available, for example to validate states and blocks against `historical_roots`.

## Why include the state at all?

This is a tradeoff between being able to access state data such as validator keys and balances directly vs recreating it by applying each block one by one from genesis. Given an erc file, it is possible to start processing the consensus chain from there onwards.

## Why the weird file name?

Historical roots for the entire beacon chain history are stored in the state - thus, with a recent state one can quickly judge if an erc file is part of the same history, which is useful for example when performing checkpoint sync.

The genesis erc file uses the genesis validators root for two reasons: it allows disambiguating otherwise similar chains and the genesis state does not yet have a historical root to use.

The era numbers are zero-filled so that they trivially can be sorted - 5 digits is enough for 99999 eras or ~312 years.

Using the first era number and the last root allows a reading application to quickly determine the range of data in the erc file.

## Why does blinding not change the root of the block?

In SSZ, a container field contributes its 32-byte `hash_tree_root` to the merkleization of its parent - the raw contents of the container are not part of the parent's tree. Replacing the field with a `Bytes32` field holding the same value thus produces an identical merkle chunk, and the root of the block is unchanged. This is what makes blinded blocks verifiable against `state.block_roots` and their proposer signatures checkable without ever seeing the payload.

## How long is an era?

An era is typically `8192` slots (in the mainnet configuration), or roughly 27.3 hours.

## What happens after Gloas?

From Gloas onwards, the execution payload is no longer part of the beacon block - the block is already blinded by the protocol. Erc files store such blocks as-is, and their roots and signatures can be verified as described above.

## Which state should be stored in the era file?

The state transition function in ethereum does 3 things: slot processing, epoch processing and block processing, in that order. In particular, the slot and epoch processing is done for every slot and epoch, but the block processing may be skipped. When epoch processing is done, all the epoch-related fields in the state have been written, and a new epoch can begin - it's thus reasonable to say that the epoch processing is the last thing that happens in an epoch and the block processing happens in the context of the new epoch.

The protocol favours the state root with the block applied, as both `BeaconState.state_roots` and `BeaconBlock.state_root`, thus era files follow suit.

The alternative that was considered is to store the state without the block applied - this has several advantages:

* the era file to be used both for future histories with and without a block at the beginning
* no special case is needed when replaying blocks from era files - all are applied in the order they appear in the era file

In the end though, the applied block state is used throughout in the protocol - given a block, the state root in the block is computed with the data from the block applied and this later gets stored in `state_roots` which forms the basis for `historical_roots`. In API:s such as the beacon API, the canonical state root of a slot is the state with the block of that slot applied, if it is part of the canonical history given by the head.
