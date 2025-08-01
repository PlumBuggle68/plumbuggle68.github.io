---
title: Bitcoin Core-Dinals
---

Ordinals In Bitcoin Core
----------

## Introduction

Ordinal theory has changed the way people think about satoshis by enabling them to be individually tracked and valued. This makes the vision of "NFTs on Bitcoin" possible. However, indexing Ordinals currently presents significant technical challenges. Most existing indexers come with excessive dependencies and don’t operate their own full nodes. By integrating an Ordinal indexer directly into Bitcoin Core, the need for external dependencies is reduced, the indexer can interface seamlessly with the most widely used node software, and the result will be a more accessible and open-source solution.

## What are Ordinals?
Ordinals are a method of assigning unique identities to individual satoshis, the smallest units of bitcoin. This allows each satoshi to be tracked, transferred, and inscribed with data, such as images or text—effectively enabling NFTs on the Bitcoin blockchain.

[https://docs.ordinals.com]

Ordinal theory defines multiple ways to reference Ordinals:

```
Integer notation: 2099994106992659 The ordinal number, assigned according to the order in which the satoshi was mined.

Decimal notation: 3891094.16797 The first number is the block height in which the satoshi was mined, the second the offset of the satoshi within the block.

Degree notation: 3°111094′214″16797‴. We'll get to that in a moment.

Percentile notation: 99.99971949060254% . The satoshi's position in Bitcoin's supply, expressed as a percentage.

Name: satoshi. An encoding of the ordinal number using the characters a through z.
```

For context, my indexer tracks and refers to Ordinals by their "Integer Notation" which is the most common way to reference them.

## What is the Ordinals Index?

The Ordinals Index is an addition to Bitcoin Core that maintains a complete or partial database of satoshi movements, allowing you to:

- Find the satoshi's in a given outpoint
- Query which transaction output currently holds a specific ordinal
- Find all historical locations of a given ordinal

## How It Works: The Technical Architecture

### Data Structures

The index is built around a couple data structures:

```cpp
struct SatoshiRange {
    uint64_t start; 
    uint64_t end;
    uint64_t Size() const { return end - start; }
};

struct TxOutputSatoshiEntry {
    std::vector<SatoshiRange> ranges;
    int block_height;
    bool spent = false;
};
```

Each transaction output is associated with a vector of `SatoshiRange`s that define which ordinals it contains. This range-based approach is common in indexers as it allows a representation of a range without needing to store millions of satoshi numbers in the DB.

### The Satoshi Flow Algorithm

The heart of the index is the `CustomAppend` function, which processes each block and tracks satoshi movements:

1. **Input Pooling**: For each transaction, collect all satoshi ranges from the inputs of a transaction
2. **Output Assignment**: Distribute pooled satoshis to outputs using the "first-in-first-out" principle
3. **Fee Calculation**: Remaining satoshis become transaction fees
4. **Coinbase Processing**: New satoshis are minted and combined with collected fees

Here's a simplified version of the core algorithm:

```cpp
// Pool all inputs
std::vector<SatoshiRange> pool;
for (const auto& txin : tx->vin) {
    TxOutputSatoshiEntry prev_entry;
    if (m_db->ReadOrdinalRanges(txin.prevout.hash, txin.prevout.n, prev_entry)) {
        for (const auto& r : prev_entry.ranges) {
            pool.push_back(r);
        }
    }
}

// Assign to outputs
for (size_t vout_index = 0; vout_index < tx->vout.size(); ++vout_index) {
    uint64_t sats = tx->vout[vout_index].nValue;
    auto assigned = SkimRanges(pool, sats);
    pool = assigned.second;
    
    TxOutputSatoshiEntry entry{assigned.first, block_height};
    m_db->WriteOrdinalRanges(tx->GetHash(), vout_index, entry);
}

// The skim ranges function returns a pair of satoshi range vectors, the first is the ranges that have been skimmed. The second is the remaining pool. :)

```

### Database Design

The index uses LevelDB with a key-value structure:

- **Keys**: `(DB_ORDINDEX, (txid, vout))` - Uniquely identifies each transaction output
- **Values**: `TxOutputSatoshiEntry` - Contains the satoshi ranges and metadata
- **Special Key**: `(DB_ORDINDEX, "lastordinal")` - Tracks the highest ordinal number

## RPC Commands: Query Interface

The index exposes three main RPC commands for querying ordinal data:

### 1. `getordinalrangesbytxoutput`

**Purpose**: Get all ordinal ranges contained in a specific transaction output

**Usage**:
```bash
bitcoin-cli getordinalrangesbytxoutput "txid" 0
```

**Response**:
```json
{
  "ranges": [
    {
      "start": "123456789",
      "end": "123456889"
    }
  ],
  "spent": false,
  "inscription": false,
  "block_height": 123456
}
```

**Implementation**: Directly queries the database using the transaction ID and output index as the key. Returns comprehensive metadata including spent status, inscription presence, and block height.

**Error Handling**:
- Invalid txid format (length != 64, non-hex characters): `RPC_INVALID_PARAMETER` (-8)
- Invalid vout (negative values): `RPC_INVALID_PARAMETER` (-8)
- Ordinal index not available: `RPC_METHOD_NOT_FOUND` (-32601)

### 2. `gettxoutputsbyordinal`

**Purpose**: Find all transaction outputs (historical and current) that contain a specific ordinal

**Usage**:
```bash
bitcoin-cli gettxoutputsbyordinal 123456789
```

**Response**:
```json
[
  {
    "txid": "abc123...",
    "vout": "0"
  },
  {
    "txid": "def456...",
    "vout": "1"
  }
]
```

**Implementation**: Iterates through the entire database, checking if the specified ordinal falls within any stored range. Returns all historical locations of the ordinal.

**Error Handling**:
- Ordinal index not available: `RPC_METHOD_NOT_FOUND` (-32601)
- Function execution error: `RPC_INVALID_ADDRESS_OR_KEY` (-5)

### 3. `getordinalposition`

**Purpose**: Get the current (most recent unspent) location of a specific ordinal ( Requires "ordindexrewritespent" )

**Usage**:
```bash
bitcoin-cli getordinalposition 123456789
```

**Response**:
```json
{
  "txid": "abc123...",
  "vout": "0"
}
```

**Implementation**: Similar to `gettxoutputsbyordinal` but only returns unspent outputs. Requires the `--ordindexrewritespent` option to be enabled.

**Error Handling**:
- Ordinal index rewrite spent not enabled: `RPC_INVALID_PARAMETER` (-8)
- Ordinal index not available: `RPC_METHOD_NOT_FOUND` (-32601)
- Ordinal not found or function error: `RPC_INVALID_ADDRESS_OR_KEY` (-5)

## Command Line Arguments

The index is controlled by several command-line arguments:

### `--ordindex`
- **Default**: `false`
- **Purpose**: Enables the ordinals index
- **Usage**: `bitcoind --ordindex`

### `--ordindexprune`
- **Default**: `false`
- **Purpose**: Automatically prunes spent outputs older than 6 blocks
- **Usage**: `bitcoind --ordindex --ordindexprune`
- **Benefit**: Significantly reduces database size for applications that only need current positions

### `--ordindexrewritespent`
- **Default**: `false`
- **Purpose**: Marks spent outputs in the database instead of deleting them
- **Usage**: `bitcoind --ordindex --ordindexrewritespent`
- **Requirement**: Required for `getordinalposition` to work properly
- **Benefit**: Enables precise tracking of current vs. historical positions

## Performance Optimizations

### Range-Based Storage
Instead of storing individual satoshis, the index uses ranges to dramatically reduce storage requirements and improve query performance.

### Pruning Options
The pruning feature allows users to trade historical completeness for reduced storage requirements.

## Tutorial: Running Your Own Ordinals Index

### Prerequisites
- Bitcoin Core development environment
- Sufficient disk space (the full index can be quite large)
- Patience for initial blockchain scanning

### Step 1: Build Bitcoin Core with Ordinals Support

```bash
git clone https://github.com/PlumBuggle68/bitcoin.git
cd bitcoin
git checkout bitcoin-core-dinals

# Build Bitcoin Core
cmake -B build
cmake --build build -j$(nproc)
```

### Step 2: Initialize the Index

For a full historical index:
```bash
./build/src/bitcoind --ordindex --ordindexrewritespent
```

For a pruned index (current positions only):
```bash
./build/src/bitcoind --ordindex --ordindexprune
```

### Step 3: Wait for Initial Sync

The initial sync will take considerable time as it processes the entire blockchain:

```bash
# Monitor progress
tail -f ~/.bitcoin/debug.log | grep "Finished indexing block height"
```

### Step 4: Query the Index

Once synced, you can start querying:

```bash
# Find current position of ordinal #50000000
bitcoin-cli getordinalposition 50000000

# Get all satoshis in the genesis block coinbase
bitcoin-cli getordinalrangesbytxoutput "4a5e1e4baab89f3a32518a88c31bc87f618f76673e2cc77ab2127b7afdeda33b" 0

# Find all historical locations of a specific ordinal
bitcoin-cli gettxoutputsbyordinal 123456789
```

### Step 5: Integration Examples

**Find inscriptions on specific satoshis**:
```bash
# Get the current location of ordinal #123456789
LOCATION=$(bitcoin-cli getordinalposition 123456789)
TXID=$(echo $LOCATION | jq -r '.txid')
VOUT=$(echo $LOCATION | jq -r '.vout')

# Get the raw transaction to examine for inscriptions
bitcoin-cli getrawtransaction $TXID true
```

**Track satoshi movements**:
```bash
# Get complete history of a satoshi
bitcoin-cli gettxoutputsbyordinal 123456789 | jq -r '.[] | "\(.txid):\(.vout)"'
```

**Examine ordinal ranges in detail**:
```bash
# Get detailed information about ordinal ranges in a transaction output
bitcoin-cli getordinalrangesbytxoutput "txid" 0 | jq '.ranges[] | "Ordinal \(.start) to \(.end)"'
```

## Performance Considerations

### Storage Requirements
- **Full Index**: Can require 100+ GB depending on blockchain size
- **Pruned Index**: Significantly smaller, typically 10-20% of full size

### Memory Usage
- The index uses configurable caching
- Default cache size should be sufficient for most use cases
- Consider increasing cache size for high-query-volume applications

### Query Performance
- Single output queries (`getordinalbytxoutput`): Very fast (direct key lookup)
- Ordinal searches (`gettxoutputsbyordinal`, `getordinalposition`): Slower (full database scan)
- Consider application-level caching for frequently queried ordinals

## Real-World Applications

This ordinals index enables numerous applications:

1. **Ordinal Explorers**: Build web interfaces to track satoshi histories
2. **Inscription Tools**: Locate and display inscribed content
3. **Trading Platforms**: Verify ordinal ownership and transfers
4. **Analytics**: Analyze ordinal distribution and movement patterns
5. **Wallet Integration**: Display ordinal holdings and enable transfers

## Future Enhancements

Several improvements could further enhance the index:

- **Incremental Updates**: More efficient handling of blockchain reorganizations
- **Query Optimization**: Secondary indexes for common query patterns
- **API Extensions**: Additional query methods and filtering options
- **Inscription Detection**

## Challanges of Development

### Why not recursive query?
  In early stages of development it became clear that a recursive query would not be the best option for the indexer. The underlying issue is that coinbase transactions contain fees from the last block. Every time the recursive function ran into a coinbase, it would have to query all of the transactions in the last block. Pulling sat ranges for coinbases from an api or similar is obviously a no go so recording a DB seemed to be the obvious option.

### Why not recursive pruning?
  As I developed the indexer on testnet, I realized that if I didn't add a pruning option, my laptop would explode. Initially, the idea was that I could recursively prune back through parent transactions after an output was spent. But when I began implementing it, I realized that it would be inefficient. In the future, it's possible I add recursive pruning as an option because it allows pruning to be enabled without a reindex. (See amazing cool drawing) Instead I decided to choose the simplest option. Currently, pruning mode caches pointers to spent transaction in the DB, then erases them after 6 blocks of confirmation (Protection against re-org? May need changes...)

![](<../images/recursive_pruning.png>)

## Conclusion

The Bitcoin Ordinals Index represents a significant step forward in making ordinal data accessible and queryable.

The combination of pruning options and RPC interfaces makes it suitable for research applications requiring complete historical data, hobby projects for goofy inscriptions, and production systems needing current state information.

---

