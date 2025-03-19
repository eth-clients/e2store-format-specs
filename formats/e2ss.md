# E2SS files

The format for storing full flat Execution state snapshots.

 Filename:

 ```text
 <network-name>-<block-number>-<short-state-root>.e2ss
 ```

 Type definitions:

 ```text
 e2ss := Version | CompressedHeader | account*
 account :=  CompressedAccount | CompressedStorage*

 Version             = { type: 0x6532, data: nil }
 CompressedHeader    = { type: 0x0300,   data: snappyFramed(rlp(header)) }
 CompressedAccount   = { type: 0x0800,   data: snappyFramed(rlp(Account)) }
 CompressedStorage   = { type: 0x0900,   data: snappyFramed(rlp(Vec<StorageItem>)) }

 Account             = { address_hash, AccountState, raw_bytecode, storage_entry_count }
 AccountState        = { nonce, balance, storage_root, code_hash }
 StorageItem         = { storage_index_hash, value }
 ```

 CompressedStorage can have a max of 10 million storage items, records must be filled before
 creating a new one, and must be sorted by storage_index_hash across all entries.
