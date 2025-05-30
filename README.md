# E2store Format Specifications
A collection of format specifications built on top of [e2store](https://github.com/status-im/nimbus-eth2/blob/613f4a9a50c9c4bd8568844eaffb3ac15d067e56/docs/e2store.md#era-files)

Links to download [e2store](https://github.com/status-im/nimbus-eth2/blob/613f4a9a50c9c4bd8568844eaffb3ac15d067e56/docs/e2store.md#era-files) formats hosted by the community https://github.com/eth-clients/history-endpoints


Historical data providers offer **three primary storage formats**:
- [**Era1**(Pre-Merge Execution History)](./formats/era1.md) - Archive nodes providing **execution layer history** before The Merge (ETH1).
- [**Era**(Beacon Chain History)](./formats/era.md) - Stores data from the genesis of the Beacon Chain onwards. Can be used by Execution layer clients for history **from The Merge onward**, including historical block data.
- [**E2SS**(Execution State)](./formats/e2ss.md) - **State snapshots** for execution clients.
- [**E2HS**(Execution Layer History)](./formats/e2hs.md) - **full execution layer history** for execution clients, provides data from genesis to latest, headers are accompanied by proofs of canonicalness.
- **Erb**(Blob) - Era file equivalent for blob sidecars [ ⚠️ Under Development ].

## E2store Types
No e2store type may be reused. A list of all defined E2store types can be found at [types/README.md](./types/README.md)

# E2store Library Implementations

| Name  |  Language | Known Supported Formats (may support more check documentation)  | Short Description  |
|---|---|---|---| 
| [trin-e2store](https://github.com/ethereum/trin/tree/master/crates/e2store)  | Rust | [Era](./formats/era.md), [Era1](./formats/era1.md), [E2SS](./formats/e2ss.md), [E2HS](./formats/e2hs.md) |   | 
| [@ethereumjs/e2store](https://github.com/ethereumjs/ethereumjs-monorepo/tree/master/packages/e2store) | Javascript | [Era](./formats/era.md), [Era1](./formats/era1.md), [E2HS](./formats/e2hs.md) |   |
| [go-ethereum](https://github.com/ethereum/go-ethereum/tree/master/internal/era) | Go | [Era1](./formats/era1.md) |   |
| [nimbus-eth1](https://github.com/status-im/nimbus-eth1/blob/master/portal/eth_data/era1.nim) | Nim | [Era1](./formats/era1.md) |   |
| [nimbus-eth2](https://github.com/status-im/nimbus-eth2/blob/stable/ncli/era.nim) | Nim | [Era](./formats/era.md) |   |
| [nimbus-e2store.py](https://github.com/status-im/nimbus-eth2/blob/stable/ncli/e2store.py) | Python | [Era](./formats/era.md) |   |
| [Besu](https://github.com/hyperledger/besu/tree/main/util/src/main/java/org/hyperledger/besu/util/era1) | Java | [Era1](./formats/era1.md) |   |
