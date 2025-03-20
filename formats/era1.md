# Era1 files

 Era1 files are themselves e2store files. For more information on this format,
 see https:github.com/status-im/nimbus-eth2/blob/stable/docs/e2store.md.

 The overall structure of an Era1 file follows closely the structure of an Era file
 which contains consensus Layer data (and as a byproduct, EL data after the merge).

 The structure can be summarized through this definition:

	era1 := Version | block-tuple* | other-entries* | Accumulator | BlockIndex
	block-tuple :=  CompressedHeader | CompressedBody | CompressedReceipts | TotalDifficulty

 Each basic element is its own entry:

	Version            = { type: 0x6532, data: nil }
	CompressedHeader   = { type: 0x0300, data: snappyFramed(rlp(header)) }
	CompressedBody     = { type: 0x0400, data: snappyFramed(rlp(body)) }
	CompressedReceipts = { type: 0x0500, data: snappyFramed(rlp(receipts)) }
	TotalDifficulty    = { type: 0x0600, data: uint256(header.total_difficulty) }
	Accumulator        = { type: 0x0700, data: accumulator-root }
	BlockIndex         = { type: 0x6632, data: block-index }

 Accumulator is computed by constructing an SSZ list of header-records of length at most
 8192 and then calculating the hash_tree_root of that list.

	header-record := { block-hash: Bytes32, total-difficulty: Uint256 }
	accumulator   := hash_tree_root([]header-record, 8192)

 BlockIndex stores relative offsets to each compressed block entry. The
 format is:

	block-index := starting-number | index | index | index ... | count

 starting-number is the first block number in the archive. Every index is a
 defined relative to index's location in the file. The total number of block
 entries in the file is recorded in count.

 Due to the accumulator size limit of 8192, the maximum number of blocks in
 an Era1 batch is also 8192.
