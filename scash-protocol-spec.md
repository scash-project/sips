# Scash Protocol

```
Authors: Simon Liu
Created: 2024-02-16
License: BSD-2-Clause
```

Scash (s/atoshi/.cash) is an experimental digital currency based on the Bitcoin protocol.

Scash is designed for mining on home computers.

Scash has been implemented as a new chain option on top of v26 of the Bitcoin Core open source software.

This document details the changes required to add support for Scash to Bitcoin related software.


#### Table of contents

1. [Proof of work](#1-pow)
1. [Block hashing](#2-block-hashing)
1. [Block validation](#3-block-validation)
1. [Algorithm performance](#4-performance)
1. [Difficulty adjustment](#5-difficulty-adjustment)
1. [Transactions](#6-transactions)
1. [Chain parameters](#7-chain-params)
1. [JSON-RPC fields](#8-json-rpc)

## 1. Proof of work algorithm

### 1.1 RandomX

Scash replaces Bitcoin's SHA256 proof of work algorithm with [RandomX](https://github.com/tevador/RandomX), a proof of work algorithm designed to close the gap between general-purpose CPUs and specialized hardware. Scash uses v1.2.1 of RandomX.

### 1.2 Parameters

Scash customizes the standard configuration parameters for RandomX, with the following change:

|Parameter|Description|Value|
|---------|-----|-------|
|`RANDOMX_ARGON_SALT`|Argon2 salt|`"RandomX-Scash\x01"`|

### 1.3 Epoch

The key `K` input to the RandomX algorithm changes over time.

Scash divides time into epochs.

The epoch `E` is calculated as `unix_timestamp / epoch_duration` using integer math.

Scash defines the epoch duration as follows:

|Chain|Seconds|
|---|---|
|Scash| `7 * 24 * 60 * 60`|
|Scash Testnet| `7 * 24 * 60 * 60`|
|Scash Regtest| `1 * 24 * 60 * 60`|

### 1.4 Key Derivation

To derive key `K` to be used as input to the RandomX algorithm, perform the following steps:

1. Compute epoch `E` as an integer value.
2. Generate a seed string `S` by substituting epoch `E` into string `"Scash/RandomX/Epoch/%d"` (`E` replaces `%d`).
3. Key `K` is the digest of `SHA256(SHA256(S))`.

Note: Using an epoch-based predefined sequence of keys instead of deriving keys from historical blockchain data enables light clients to fully validate the proof of work in a block header without requiring any contexual data from the blockchain.

## 2 Block hashing

### 2.1 Block header

The Scash block header extends the Bitcoin block header with a new field `hashRandomX` to store the result of executing the RandomX algorithm.

| Field | Size (Bytes) |
|---|---|
| Version | 4 |
| hashPrevBlock | 32 |
| hashMerkleRoot | 32 |
| Time | 4 |
| Bits | 4 |
| Nonce | 4 |
| hashRandomX | 32 |

### 2.2 Block hash

The block hash algorithm remains the same as Bitcoin, double SHA256 over the entire block header: `SHA256(SHA256(block_header)`.

The block hash is not used to validate proof of work.

### 2.3 RandomX hash

RandomX hashing requires two input values:

* Key `K` with a size of 32 bytes
* Block header `H` with a size of 112 bytes
  * set the `hashRandomX` field to null (all zero)

and outputs a 256-bit result `R`.

The hashing process consists of the following steps:

1. Compute the epoch `E` from the block header field `Time`.
1. Compute the seed string `S` from epoch `E`.
1. Derive key value `K` from the seed string `S`.
1. Result `R` is calculated by executing the RandomX algorithm with key `K` and block header `H`.

Result `R` is stored in the block header field `hashRandomX`.

### 2.3 RandomX commitment

Calculating the RandomX commitment value requires two input values:

* Hash value `hashRandomX` with a size of 32 bytes, for a block header `H`
* Block header `H` with a size of 112 bytes
  * set the `hashRandomX` field to null (all zero)

and outputs a 256-bit result `CM`.

The commitment value `CM` is transient and is not stored in the block header.

## 3 Block validation

A block header has valid proof of work when both:

1. The RandomX commitment value `CM` for the block header meets the current target.
2. The RandomX hash value `hashRandomX` in the block header is verified.

### 3.1 Commitment value meets target 

In Bitcoin, the block hash must be lower than or equal to the current target `T`, a large 256-bit number, for the block to be accepted by the network.

In Scash, instead of using the block hash, the RandomX commitment value `CM` must be lower than or equal to the current target `T`:
- `CM <= T`, meets the target
- `CM > T`, does not meet the target

### 3.2 Verifying RandomX hash value

To verify the block header field `hashRandomX`, perform the RandomX hashing algorithm over the block header, as described earlier.

Compare the result `R` against the value in the block header:
- `R == hashRandomX`, block header is valid
- `R != hashRandomX`, block header is invalid

### 3.3. Light clients

Light clients may choose to only verify the commitment `CM` and skip the more computationally expensive verification of `hashRandomX` when processing block data from trusted sources.

Trusted sources such as full nodes must perform full verification of both values before accepting blocks, given it is possible to construct a block header with a fake `hashRandomX` which generates a valid commitment `CM`.

## 4 Algorithm performance

The Scash node software provides options to adjust RandomX performance.

| Option | Purpose | Default |
|---|---|:---:|
| randomxfastmode | Enable fast mode | false |
| randomxvmcachesize | Number of epochs/VMs to cache | 2 |

### 4.1 Fast mode

The RandomX library provides a fast mode but this greatly increases the amount of memory used by the RandomX algorithm:
- Fast mode - requires 2080 MiB of shared memory, suitable for mining.
- Light mode - requires 256 MiB of shared memory, but runs slower.

### 4.2 Caching

Every key `K` requires it's own uniquely initialized RandomX virtual machine to execute the RandomX algorithm.

Since block timestamps `Time` are not guaranteed to increase monotonically, caching can eliminate the expense of recreating a RandomX virtual machine when epochs are out of order.


## 5 Difficulty adjustment

The Scash network has the same difficulty adjustment algorithm as Bitcoin, but the result of difficulty adjustment calculations will differ slightly.

### 5.1 Algorithm

Scash does not change the difficulty adjustment algorithm. Just like Bitcoin, the network tries to produce a new block every 10 minutes and modifies the target every 2016 blocks, with difficulty changing by no more than a factor of 4 either way to prevent large changes.

### 5.2 Calculation

Scash performs the same difficulty adjustment calculations as Bitcoin, but fixes an [off-by-one bug](https://bitcointalk.org/index.php?topic=43692.msg521772#msg521772) which resulted in difficulty being calculated incorrectly over 2015 blocks instead of 2016 blocks.

## 6 Transactions

The Scash node software makes a number of changes to the default behaviour of the Bitcoin node software. These changes impact transaction propagation across the Scash network.

### 6.1 Replace-by-fee

The Scash node software disables replace-by-fee (RBF).

The mempool returns to first-seen-rule behaviour and rejects conflicting transactions.

- Option `-mempoolfullrbf` disabled
- Option `-walletrbf` disabled
- RPC argument `replaceable` disabled

### 6.2 Data carrier

The Scash node software disables the `-datacarrier` option.

Non-coinbase transactions containing an `OP_RETURN` transaction output remain consensus valid and can be mined into blocks, but they will not be relayed by the node.

Scash discourages the use of the network for storing data not required to validate a transaction.

### 6.3 Ordinals Inscriptions

The Scash node software considers a transaction as non-standard when the following dead code patterns are detected in Tapscript:
- `OP_FALSE OP_IF`
- `OP_NOTIF OP_TRUE`

Non-standard transactions remain consensus valid and can be mined into blocks, but they will not be relayed by the node.

Scash discourages the use of the network for storing data not required to validate a transaction.

## 7 Chain parameters

The Scash chain modifies the default Bitcoin chain parameters.

### 7.1 Network handshake

Scash adds `0x01` to each of the network magic bytes used by Bitcoin, to define the following defaults:

| Chain | Magic bytes |
|---|---|
| Scash | `0xfa 0xbf 0xb5 0xda` |
| Scash Testnet | `0x0c 0x12 0x0a 0x08` |
| Scash Regtest | `0xfb 0xc0 0xb6 0xdb` |

### 7.2 Network ports

Scash adds `10` to the default Bitcoin port numbers, to define the following defaults:

| Chain | RPC | Network | Tor |
|---|---|---|---|
| Scash | `8342` | `8343` | `8344` |
| Scash Testnet | `18342` | `18343` | `18344` |
| Scash Regtest | `18453` | `18454` | `18455` |

### 7.3 Bech32 prefix

Scash updates the human readable prefix for Bech32 addresses as follows:

| Chain | Prefix |
|---|:---:|
| Scash | scash |
| Scash Testnet | tscash |
| Scash Regtest | rscash |

## 8 JSON-RPC fields

The Scash node software adds RandomX data fields to the following API endpoints.

| API  | Key | Value | Description
|---|---|---|---|
| `getblock` | `"rx_cm"` | Hex string | RandomX commitment value |
|  | `"rx_hash"` | Hex string | RandomX hash value |
|  | `"rx_epoch"` | Integer | Epoch |
| `getblocktemplate` | `"rx_epoch_duration"` | Integer | Epoch duration in seconds |



