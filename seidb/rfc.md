# Authors
@yiming @spaceinvader

# Overview

The purpose of this RFC document is to propose the design of the next generation of Cosmos SDK storage V2, a more optimized and an overhaul of the current Cosmos IAVL based storage (storage v1), to overcome the flaws and shortcomings that have been observed and exposed over the past few years.

The high level idea is mostly borrowed from [ADR-065: Store V2 | Cosmos SDK](https://docs.cosmos.network/main/architecture/adr-065-store-v2) proposal. In this document, we will discuss in more detail and present a few feasible solutions around the problems we see specifically in sei to tackle the state bloat issue.

# Problem Statement

Here are the problems we are trying to solve in this design.

### Write Amplification

Write amplification is defined as the ratio between write size occupied and the amount of "useful" data. In other words it measures how much metadata is needed to maintain the data structures, the smaller the better.

In the current storage V1, the whole IAVL tree is persisted into the database. In a detached atlantic-2 node which has only the most recent version and fully compacted, the disk space occupied by `application.db` is 25GB. With IAVL dump, we got the actual total "useful" data size is 10GB. So the write amplification is roughly 2.5x, in other words, we need 15GB of metadata in order to maintain 10GB of actual application state, which is very inefficient.

### Storage Growth

Due to write amplification, disk usage grows a lot. In atlantic-2 testnet for example, archive node storage usage is growing more than 150 GB/day, or 1TB/week. The storage space growth rate would also keep increasing as the state of the chain keeps growing.

Another issue with the current design is that even with aggressive pruning turned on, the disk usage will still keep growing due to the orphan nodes and inefficient compaction.

These issues would lead to several bad impacts:

* Archive nodes would become more and more expensive to maintain
* Database operations would become slower and slower
* RPC Nodes can’t run for a long time due to disk would fill up quickly

### Slow Operations

One of the biggest pain points and risks is that almost all critical chain level operations become super slow when the state size of the chain grows above a certain size.

* State Sync (IAVL Import) becomes slow. Snapshot size is growing bigger and bigger, thus restoring from snapshot becomes slower and slower, from a few seconds to a few hours.
* Snapshot creation (IAVL Export) takes a couple of hours to complete. This leads to the latest snapshot height being too far from the latest block height, thus it would take much longer time for the node to catch up after state sync finishes.
* Rollback a single block could take a couple of hours or up to a day to complete.
* IAVL dump takes couple of hours to finish, making debugging difficult and troublesome

### Performance Degradation Over Time

State bloat would also lead to performance degradation over time. Performance issues are usually not surfacing out right after doing a state sync, but as the node runs longer, the performance of the node would keep degrading.

* RPC nodes would start falling behind after running a while
* Archive nodes can not keep up with the blockchain, historical queries from archive node are slow
* Validator nodes would start to see bad signing performance or block time increase after running a few weeks
* Compaction in the background consumes CPU cycles and also adds lock contention, which also hurts performance
* Aggressive Pruning would cause the TX processing time and block time to increase, as well as signing performance to be degraded

# Requirements

[P0] Improve write amplification and storage amplification, reducing overall storage usage

[P0] Improve snapshot creation and snapshot restore performance, reducing the time to bring up a new PRC node and catch up to latest

[P0] Boost rollback and IAVL dump performance

[P0] Improve chain reliability so that nodes can run longer without restart or re-statesync

[P0] Enhance sustainability so that chain can run without and performance and operational concerns even with the same state bloat issue in atlantic-2

[P1] Make blocks speed faster, faster to commit new blocks

[P1] Improve database performance, overall faster to get, set, and iterate over keys to serve historical queries

# Current Approach

IAVL tree is a versioned, snapshottable and immutable AVL+ Tree. An AVL Tree is a self balanced binary search tree, all operations are O(Log(n))

![|480x270](./current.gif)

In the current SDK, IAVL is responsible for both state storage and state commitment, running an archive node becomes increasingly expensive as disk space grows exponentially.

The reason why storage amplification is so high is also because to achieve state commitment, we need to persist a lot of intermediate branch nodes apart from all the leaf nodes which contain the key/value pairs, and each node contains a lot of metadata. Here’s the current data structure of a tree Node: https://github.com/cosmos/iavl/blob/master/node.go#L59

Each module has its own underline IAVL tree and its own root. All the tree roots together will compose a top level root hash, which will be used as the block commit hash (app hash).

There are a few issues with the current design:

* We have to store way more data than we actually need
* The key format of IAVL nodes is a hash of the node. It does not take advantage of data locality on LSM-Tree
* Nodes are stored with the random hash value, so it increases the number of compactions and makes it difficult to find the node
* The orphan nodes are used to manage node removal in the current design and allow the deletion of removed nodes for the specific version from the disk. It needs to track every time when updating the tree and also requires extra storage to store orphans

# Proposed Design

Here we propose to separate the concerns of state commitment (SC), which is needed for consensus, and state storage (SS) to store key value pairs, which is needed for historical queries and state machines.

By separating SS and SC, the SS layer will be responsible for direct access to data in the form of (key, value) pairs, whereas the SC layer (IAVL) will be responsible for committing blocks and providing Merkle proofs. This will allow us to optimize against primary use cases and access patterns to the state.

We will replace the original IAVL with a more optimized SC implementation, which could just be an in-memory implementation of IAVL SMT tree, the goal is to make sure all state commitment related operations are fast, reliable and light.

As for the state storage layer, we will provide a pluggable interface, which will allow specific applications to plug-in any database they prefer, if not using the default implementation. The goal is to have a scalable, compact and fast data store to serve historical queries.

## State Commitment (SC)

The proposed solution for state commitment is to adopt MemIAVL, which was invented by Cronos and recently already proved to work in their production environment for a while.

### MemIAVL Architecture

The high level idea of MemIAVL is that each time when a new block get committed, we will extract all the change sets from the transactions of that block, and then we will apply those changes for the current in-memory IAVL tree, result in a new version of the tree for the latest block, so that we can get the Merkle root hash for the block commitment. In this commit process, everything is in memory. Each mem node has a data structure like this:
```
type MemNode struct {

height uint8

size int64

version uint32

key []byte

value []byte

left Node

right Node

hash []byte

}
```

However, we do need to have some level of persistence, so that we can quickly recover the in-memory state when the node is crashed. To solve this problem, MemIAVL introduces a WAL file and a tree snapshot.

First, every snapshot interval, we will take a new snapshot of the current in-memory tree and persist the snapshot on disk. We will save up to a certain number of recent snapshots and old snapshots will be pruned.

The difference between this tree snapshot and the original IAVL snapshot is that when we recover and load the snapshot, we will use Mmap to load the snapshot instead of scanning the disk and loading all the data to rebuild the whole tree.

Second, for every block, we will asynchronously write all the change sets to a Write-Ahead-Log (WAL) file. The WAL file can be used for recovery and catch up when needed. For example, if the node got crashed at height 1300, and we have a snapshot at height 1000, we can recover the node by loading from the snapshot first and then replay all the changes for the remaining 300 blocks in the WAL file to catch up to the latest block.

### File Format

#### WAL File
```
version: 8

size: 8 // size of whole payload

payload:

delete: 1

keyLen: varint-uint64

key // if delete is false

[

valueLen: varint-uint64

value

]

repeat with next version
```
#### Snapshot Format

* Metadata file, 16 bytes:
```
magic: 4

format: 4

version: 4
```
* Nodes file, array of fixed size(16+32 bytes) nodes, the node format is like this:
```
# branch node

height : 1

pretrees : 3

version : 4

size : 4

key leaf : 4

hash : [32]byte

# Leaf node

version : 4

key length : 4

kv offset : 8

hash : [32]byte
```
* nodes are written with post-order depth-first traversal, so the root node is always placed at the end
* kvs files, sequence of leaf node key-value pairs:
```
keyLen: varint-uint64

key

valueLen: varint-uint64

value

*repeat*
```
### Export

Creating/Exporting a state-sync snapshot is the same process as the current cosmos SDK, and the state-sync snapshot it creates has the exact same format as the existing cosmos SDK, so it will be fully compatible with our current chain. It is possible to use an existing state-sync snapshot to restore a node backed by memIAVL.

### Import

To restore from a state-sync snapshot, we will be loading the snapshot file to rebuild the in-memory IAVL tree. This process will be much faster than the current state sync because there’s no disk writes involved at all.

### Node Types

Based on different node types and purposes, we can have different ways to set up the storage layer.

For Validator nodes, since we don’t need to serve historical queries, we can cache everything in memory and only keep the latest few blocks, so we can run that with MemIAVL without any database.

For RPC nodes and Archive nodes, since we need to serve as historical queries, we want to have a persistent layer for historical versions, so we will go with MemIAVL and some sort of Database.

### Advantages

* Better write amplification, we only need to write the change sets in real time which is much more compact than IAVL nodes, IAVL snapshot can be created in much lower frequency.
* Better read amplification, the IAVL snapshot is a plain file, the nodes are referenced with offset, the read amplification is simply 1.
* We don't need to keep too many old IAVL snapshots, because the state store will handle the historical key-value queries.
* Super fast state sync because everything are loaded into the memory instead of into golevelDB

### Disadvantages

* To serve historical proofs, we need to load the whole tree from snapshot and catch up to a certain height, which could be slow (a few hundred of ms)
* Since the tree is purely in memory, this approach could cause a lot of memory consumption and might cause OOM if the state becomes really huge. We need to make sure that the node can at least hold all the state in memory within a single snapshot interval
* Snapshot rewrite might cause IO contention

### Benchmark Result

Atlantic-2

* Num of Chunks: 365
* State Snapshot Size (Compressed): 3.65GB
* application.db size (after sync): 42GB
* memiavl.db size: 20GB

||State Snapshot Creation|State Sync|Block Sync Rate|Rollback|
| --- | --- | --- | --- | --- |
|IAVL v0.19|3 hours - 7 hours|52 min|9 blocks/s|1-2 hours|
|MemIAVL|7 min - 40 min|6 min|18 blocks/s|3 secs|
|IAVL v1|1.5 hours|61 min|11 blocks/s|4 secs|

Pacific-1

* Num of Chunks: 16
* State Snapshot Size (Compressed) : 160MB
* application.db size (after sync): 2.7GB
* memiavl.db size: 1.1GB

||State Snapshot Creation|State Sync|Block Sync Rate|Rollback|
| --- | --- | --- | --- | --- |
|IAVL v0.19|5 min|40 secs|18 blocks/s||
|MemIAVL|20 secs|20 secs|40 blocks/s||

## State Store (SS)

Since State Store will be only used to serve historical queries, we want to choose a more optimized and suitable database to serve such queries in a highly scalable, storage efficient and fast manner.

### Benchmark Dimensions

* Storage size to persist N historical blocks
* Compaction ratio and performance
* Get/Set/Delete latency & throughput when more than N blocks stored
* Iterate latency & throughput when more than N blocks stored
* Reverse iterate latency & throughput when more than N blocks stored
* Range queries latency & throughput when more than N blocks stored

### Benchmark Result (TBD)

||Storage Size|Latency|Throughput|Compaction|
| --- | --- | --- | --- | --- |
|RocksDB(VersionDB)|||||
|PebbleDB|||||
||||||
|SQLLite|||||

#### DB Benchmark References

* Notional PebbleDB v Goleveldb Benchmarking: https://github.com/notional-labs/cosmosia/issues/81
* Notional Rocksdb Benchmarking: https://github.com/notional-labs/cosmosia/issues/60
* General PebbleDB v RocksDb v Leveldb benchmark: https://github.com/utsaslab/pebblesdb/blob/master/benchmark.md
* Generator: https://github.com/kocubinski/iavl-bench/blob/main/bench/gen.go

### Change Stream

The way we feed data into the database is through cosmos SDK change stream feature. This can be done asynchronous and we just need to register a listener https://docs.cosmos.network/v0.45/architecture/adr-038-state-listening.html so that each time a new change is available for a block, it will be also applied to the state store in an async manner.

One concern of this approach is whether applying the database would be slower than the block generation speed, so that the state store ends up with higher and higher lag and becoming inconsistent with the latest state.

This generally should not be a concern at all, because we are just competing disk writes with network P2P consensus, and it is believed that in most scenarios, writing some key values to the local database should be much faster than reaching the consensus to generate a new block.

However, if this does become an issue, we can also add some throttling mechanism so slow down the chain to wait for the database operations if the lag is too high.

# Alternatives Considered

## Non-IAVL Based SC

### State Commitment

The point of having Merkle root is so that anyone can take the data stored at your node and apply a function to it to arrive at the Merkle root hash that you allege to have (which is presumably in consensus with the majority nodes out there).

There are other ways to achieve this than computing a global root hash of all the data. One possible scheme would be:

MH(T) = Hash(MH(T-1), Root hash of all changed entries in T)

Note that the second parameter to the hash function needs to be deterministic, which can be achieved by feeding all changed entries into a tree in a deterministic order (e.g. ascending order of the keys), which is similar to the memIAVL change sets.

### State Storage

The alternative storage scheme is as following

* Key: `<key>-<version>-<tombstone>`
* Value: `<value>`

The only metadata used in this schema is the version number (8 bytes) and tombstone (1 byte) for each key value entry.

Examples to illustrate the scheme:

* Write a bankbalanceseiabcdefg:1000usei pair at height 3000 will result in an entry of bankbalanceseiabcdefg-3000-0:1000usei written into LevelDB
* Delete key bankbalanceseixyz at height 5000 will result in an entry of bankbalanceseixyz-5000-1:"" written into LevelDB

The underlying SSTable will be populated similar to:

……
bankbalanceA-1-0 500
bankbalanceA-7-0 300
bankbalanceB-1-0 200
bankbalanceB-3-0 1000
bankbalanceB-5-1
bankbalanceC-1-0 100
……

### Merkle Proof

Now say if you suspect something fishy going on with your validator and want to verify key `k` at version `T`, you can first find the last changed version of `k` on your validator's node (denoted as `T'`) that predates `T`. Then you can get the consensus Merkle root for T'-1 and the consensus Merkle root for T' from the network. FInally you can get all the changed entries at height T' on your validator's node, and verify with the function above.

Biggest Concern

We have to do a lot of range queries for TX execution

## IAVL v1

IAVL v1 is the latest IAVL version cosmos SDK community has recently released. There are a bunch of optimizations that have been made in the new IAVL version. More details can be found in https://github.com/cosmos/iavl/blob/master/docs/architecture/adr-001-node-key-refactoring.md

This is currently being discussed and benchmarked in the storage working group as another possible implementation. However the downside is that this implementation doesn’t separate the SC and SS layer, and will still persist all the tree nodes into the database, so it might now help a lot to reduce the disk space usage.**
# References

Storage V2: https://docs.cosmos.network/main/architecture/adr-065-store-v2

MemIAVL: https://github.com/crypto-org-chain/cronos/blob/main/memiavl/README.md

# Community Discussion

- [Github Discussions for this RFC](https://github.com/sei-protocol/rfc/discussions/6)