# E2HS files

 E2HS files are themselves e2store files. For more information on this format,
 see the [documentation](https:github.com/status-im/nimbus-eth2/blob/stable/docs/e2store.md).

 The overall structure of an E2HS file follows closely the structure of an Era file
 which contains consensus Layer data (and as a byproduct, EL data after the merge).

 The structure can be summarized through this definition:

	e2hs := Version | block-tuple* | other-entries* | BlockIndex
	block-tuple :=  CompressedHeaderWithProof | CompressedBody | CompressedReceipts

 Each basic element is its own entry:

	Version                     = { type: [0x65, 0x32], data: nil }
	CompressedHeaderWithProof   = { type: [0x03, 0x01], data: snappyFramed(ssz(header_with_proof)) }
	CompressedBody              = { type: [0x04, 0x00], data: snappyFramed(rlp(body)) }
	CompressedReceipts          = { type: [0x05, 0x00], data: snappyFramed(rlp(receipts)) }
	BlockIndex                  = { type: [0x66, 0x32], data: block-index }

 The `header_with_proof` type definition can be found in the Portal Network [documentation](https://github.com/ethereum/portal-network-specs/blob/650b65f5893930bbf4b0f08edd53fae19f0938ba/history/history-network.md#block-header).

 `BlockIndex` stores relative offsets to each compressed block entry. The
 format is:

	block-index := starting-number | index | index | index ... | count

 `starting-number` is the first block number in the archive. Every index is a
 defined relative to index's location in the file. The total number of block
 entries in the file is recorded in count.

 `other-entries` is an extension point for future record types in the `e2hs` format. The positioning of these allows the indices to continue to be looked up from the back of the group.

 Every E2HS file must contain a contiguous section of 8192 blocks.

 ### File Naming Convention
 `<config-name>-<era-number>-<short-hash>.e2hs`

 Where:
 - `config-name`: Network configuration identifier (e.g., mainnet, sepolia) 
 - `era-number`: Sequential epoch identifier
 - `short-hash`: first 4 bytes of the last block header's hash included in this file
