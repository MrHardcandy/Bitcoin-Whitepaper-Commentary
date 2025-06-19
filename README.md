# Bitcoin: From Theory to Code - An Implementation-Level Commentary

This document provides an in-depth, code-level technical commentary on Satoshi Nakamoto's Bitcoin whitepaper. It connects the high-level concepts from the paper to the specific C++ implementations within the Bitcoin Core client, bridging the gap between theory and engineering practice.

## Table of Contents
1. [Introduction: The Promise and Core Problem of Bitcoin](#introduction-the-promise-and-core-problem-of-bitcoin)
2. [Section 2: Transactions](#section-2-transactions)
3. [The Script System: Bitcoin's Programmability](#the-script-system-bitcoins-programmability)
4. [Sections 3 & 4: Blocks, Timestamp Server, and Proof-of-Work](#sections-3--4-blocks-timestamp-server-and-proof-of-work)
5. [Sections 5 & 6: Network and Incentive](#sections-5--6-network-and-incentive)
6. [Advanced Topic: P2SH (Pay-to-Script-Hash)](#advanced-topic-p2sh-pay-to-script-hash)
7. [Advanced Topic: Timelocks](#advanced-topic-timelocks)
8. [Advanced Topic: Segregated Witness (SegWit)](#advanced-topic-segregated-witness-segwit)
9. [Wallet Technology: The Gateway to User Experience](#wallet-technology-the-gateway-to-user-experience)
10. [Historical Perspective: Major Upgrades & Hard Forks](#historical-perspective-major-upgrades--hard-forks)

---

## Introduction: The Promise and Core Problem of Bitcoin

> **Whitepaper Quote**: "A purely peer-to-peer version of electronic cash would allow online payments to be sent directly from one party to another without going through a financial institution."

This opening statement is Bitcoin's declaration of independence. It's more than a vision; every word is supported by specific code and protocol rules.

### "Peer-to-Peer" Implementation
The Bitcoin network has no central server. It's a flat network of thousands of peer nodes.
*   **Network Layer**: Nodes discover and connect to each other through logic found in `src/net.cpp` and `src/net_processing.cpp`. Each node maintains a list of peers and exchanges information directly.
*   **Information Propagation**: The spread of transactions and blocks relies on the **Gossip Protocol**. When a node has new information, it announces it to its peers with an `inv` message. Nodes that don't have the data request it with `getdata`. This mechanism prevents broadcast storms and allows information to propagate efficiently without a central coordinator.

### "Without a financial institution" Implementation
Bitcoin makes the role of a traditional bank obsolete by decentralizing its three core functions: custody, authorization, and settlement.

1.  **Custody**:
    *   **Traditional Bank**: The bank holds the customer's funds.
    *   **Bitcoin Implementation**: Users hold their own funds via **wallets**. The **HD Wallet (BIP 32/39)** technology is key here. A user's funds are tied not to an intermediary, but to a **master seed** and the private keys derived from it. The private key *is* ownership. As the saying goes: "Not your keys, not your coins."

2.  **Authorization**:
    *   **Traditional Bank**: The bank decides whether a transfer can occur based on account information and policy.
    *   **Bitcoin Implementation**: Authorization is a purely cryptographic process, not a human-intervened one. The **Script system** (`src/script/interpreter.cpp`) is at its heart. A transaction is valid if and only if the spender can provide a valid `scriptSig` to unlock the `scriptPubKey` of a previous output. The core operation, **`OP_CHECKSIG`**, doesn't care who you are, only that your digital signature corresponds to the public key. The power to authorize transactions lies solely with the holder of the private key.

3.  **Settlement**:
    *   **Traditional Bank**: A bank maintains a central ledger and updates balances to finalize transactions.
    *   **Bitcoin Implementation**: Settlement is achieved through a decentralized **consensus mechanism**, the process by which all honest nodes agree on a single transaction history. This is detailed below.

### The Core Problem: Double-Spending

> **Whitepaper Quote**: "Digital signatures provide part of the solution, but the main benefits are lost if a trusted third party is still required to prevent double-spending."

*   **What "Double-Spending" Means**:
    In the digital realm, any piece of information can be perfectly and costlessly duplicated. A digital cash transaction is also just a piece of data. **Double-spending** refers to a malicious actor sending the same "digital coin" (the same piece of transaction data or a variant of it) to two or more different recipients. For example, Alice pays Bob with one bitcoin and then immediately pays Carol with the *same* bitcoin.

*   **Why Digital Signatures Alone Are Insufficient**:
    A digital signature proves **ownership and authenticity**. It can prove that a transaction was indeed authorized by Alice, the legitimate owner of the private key.
    However, it cannot stop Alice from creating **two or more separate, validly signed transactions** that both attempt to spend the same UTXO.
    *   Transaction 1: Alice -> Bob (spending UTXO_X) -> Valid signature
    *   Transaction 2: Alice -> Carol (spending UTXO_X) -> Valid signature
    When Bob and Carol receive their respective transactions, they both see a cryptographically perfect transaction. Neither has a way of knowing if Alice sent another transaction spending the same funds to someone else. Without a global, unified perspective, it's impossible to determine which transaction was "first" and thus the only legitimate one. This is why a trusted third party (like a bank) is needed to maintain a single ledger.

### The Solution: A Grand Synthesis

> **Whitepaper Quote**: "...a solution to the double-spending problem using a peer-to-peer network... The network timestamps transactions by hashing them into an ongoing chain of hash-based proof-of-work..."

Bitcoin's solution is not a single technology but an elegant system of five pillars working in synergy.

1.  **The Public Ledger (The Blockchain)**
    *   **Concept**: The Bitcoin blockchain is a global, shared ledger that anyone can view and verify. All transactions are eventually recorded publicly. This transparency is the first line of defense: any double-spend attempt will be publicly recorded, leaving it nowhere to hide.

2.  **The UTXO Model (The Unspent Transaction Output Model)**
    *   **Concept**: Bitcoin uses a **UTXO model**, not a bank-like "account/balance" model. Every available unit of money is a discrete, independent "coin" or "note" (a UTXO).
    *   **Synergy**: This model vastly simplifies double-spend detection. When a new transaction is broadcast, nodes don't need to check Alice's "account balance." They only need to check if the specific **UTXO** being referenced still exists in the set of unspent transaction outputs. Once a UTXO is included in a confirmed transaction, it's removed from the valid set. Any subsequent transaction attempting to spend the **same UTXO** will be immediately rejected as its input is invalid.

3.  **The Timestamp Server (The Chain)**
    *   **Concept**: This is the **ordering mechanism** that prevents double-spends. By including the `hashPrevBlock` field in the `CBlockHeader`, each block is cryptographically linked to the one before it, forming an unbreakable chronological chain.
    *   **Synergy**: This chain provides a definitive, unique **order of events** for all transactions. Even if Alice broadcasts two double-spend transactions simultaneously, they cannot both be recorded in the same valid history due to the blockchain's linear structure. Only one will ultimately be accepted. Tampering with this order requires rewriting the entire history, a feat made computationally difficult by the next pillar.

4.  **Proof-of-Work (The Cost)**
    *   **Concept**: This imposes a **physical cost** on the "right to write" to the ledger. By requiring miners to constantly adjust a `nNonce` to find a hash below the target defined by `nBits`, the protocol ensures that adding a new page (block) to the public ledger is difficult and consumes real-world resources (electricity and computation). This process is validated by the `CheckProofOfWork` function.
    *   **Synergy**: Proof-of-work ties purely digital information to real-world physical cost. It makes the act of creating history expensive, thus preventing an attacker from easily creating an alternative "forged" history to replace the real one. It introduces the principle of "one-CPU-one-vote" into the digital realm.

5.  **The Longest Chain Rule (The Consensus)**
    *   **Concept**: This is a simple yet powerful **consensus algorithm**. When faced with two or more valid blockchain forks, all honest nodes follow a simple rule: always consider and extend the chain with the most cumulative proof-of-work (`nChainWork`) as the single, authoritative version of history.
    *   **Synergy**: This is the final glue that holds the system together. It provides a mechanism for the decentralized network to spontaneously agree on a single version of history without a central coordinator or voting. Even if temporary forks occur (e.g., two miners find a block at nearly the same time), the longest chain rule ensures the network will eventually converge to a single chain, definitively invalidating one of the double-spend transactions. The **chain reorganization (reorg)** process is the code-level implementation of this rule.

**In Synthesis**: The UTXO model defines what a "double-spend" is, the public ledger makes it visible, the timestamp chain gives it an order, proof-of-work makes that order costly to forge, and the longest chain rule allows everyone to agree on the final order. These five pillars together build a trustless system that robustly solves the double-spending problem.

### The Result: An Unforgeable, Costly-to-Alter History

> **Whitepaper Quote**: "...forming a record that cannot be changed without redoing the proof-of-work."

*   **Quantifying "Immutability"**:
    Bitcoin's "immutability" is not an absolute cryptographic certainty but an **economic and probabilistic one**. To alter history, an attacker's challenge is far greater than just recalculating the hash of a single block.

*   **The Attacker's Challenge**:
    Imagine an attacker wants to alter a transaction 6 blocks deep in the history. They must:
    1.  Modify the transaction in that block and re-find a valid proof-of-work for **that block**.
    2.  Because they changed the content of block #6, its hash has changed. This means the `hashPrevBlock` field in the next five blocks (5, 4, 3, 2, and 1) is now invalid.
    3.  Therefore, the attacker must also re-find a valid proof-of-work for **all five subsequent blocks**.
    4.  Most critically, they must do all of this **faster than the entire honest network is adding new blocks to the original chain**. Otherwise, the honest chain will continue to grow, and their fork will never become the "longest chain."

*   **Probabilistic Finality and the "6-Block Confirmation" Rule**:
    The smaller an attacker's share of the total network hash rate, the more their probability of successfully catching up to and overtaking the honest chain decreases exponentially with each new block.
    *   **1 Confirmation**: A small probability of a natural fork causing a transaction "rollback" still exists.
    *   **3 Confirmations**: An attacker would need a very significant amount of hash power (e.g., over 30-40%) and considerable luck to reverse this transaction.
    *   **6 Confirmations**: At this depth, the probability of an attacker successfully reversing this chain of six blocks is infinitesimally small, unless they control over 51% of the network's hash rate. This is why "6 block confirmations" is widely considered the gold standard for a transaction reaching **practical, economic finality**.

---

## Section 2: Transactions

Section 2 of the whitepaper describes a transaction as a "chain of electronic coins." The `CTransaction` class in Bitcoin Core is the direct code implementation of this concept.

*   **Core Data Structure**: `CTransaction`
*   **Header File Path**: `bitcoin/src/primitives/transaction.h`

### `CTransaction` Data Structure Analysis

| Member Variable | C++ Type                    | Brief Description                                                                 |
|:----------------|:----------------------------|:----------------------------------------------------------------------------------|
| `version`       | `const uint32_t`            | Transaction version number. Used to identify the rule set the transaction follows, enabling upgrades and soft forks. |
| `vin`           | `const std::vector<CTxIn>`  | A vector of transaction inputs. Each input references an output from a previous transaction. |
| `vout`          | `const std::vector<CTxOut>` | A vector of transaction outputs. Each output defines new "coins."                 |
| `nLockTime`     | `const uint32_t`            | Lock Time. A timestamp or block height before which the transaction is invalid.  |

### Comparison with the Whitepaper's Concept

*   **Direct Correspondence**:
    *   `vin` (inputs) and `vout` (outputs) are the direct embodiment of the whitepaper's concept. The "chain of digital signatures" is implemented by the `scriptSig` in a `vin` (which usually contains the signature) unlocking the `scriptPubKey` (the locking conditions) in a previous `vout`.

*   **Key Implementations Not Mentioned in the Whitepaper**:
    *   **`version`**: Allows for the introduction of new rules (like SegWit) without breaking network consensus, forming the basis for the protocol's evolution.
    *   **`nLockTime`**: Adds a time dimension to transactions, allowing for "future" transactions that are only valid after a certain point in time. This is fundamental for more complex contracts like HTLCs.

### Transaction Validation Logic
When a node receives a new transaction, it performs context-free basic checks.

*   **Core Function**: `CheckTransaction`
*   **File Path**: `bitcoin/src/consensus/tx_check.cpp`
*   **Main Checks**:
    *   Ensures inputs and outputs are not empty.
    *   Checks that the transaction size is within limits.
    *   Checks that output values are positive and within a reasonable range.
    *   Checks for duplicate inputs (preventing an inflation bug, CVE-2018-17144).

---

## The Script System: Bitcoin's Programmability
The validation of Bitcoin transactions relies on a powerful and flexible stack-based scripting language.

### Core Scripting Components
*   **Data Structure**: `CScript`, defined in `bitcoin/src/script/script.h`.
*   **Interpreter**: The `EvalScript` function, located in `bitcoin/src/script/interpreter.cpp`.
*   **Validation Entrypoint**: The `VerifyScript` function, also in `bitcoin/src/script/interpreter.cpp`.

### `scriptSig` and `scriptPubKey` Interaction
The validation process is analogous to fitting a key (`scriptSig`) into a lock (`scriptPubKey`).
1.  **Concatenate Script**: `<scriptSig> + <scriptPubKey>`
2.  **Execute Script**: The interpreter executes the combined script from beginning to end in a stack-based environment.
3.  **Determine Result**: If the script executes without error and the final element on the stack is `TRUE`, the validation is successful.

### Key Opcodes Analysis
| Opcode           | Description                                                                                                                                                                                                   |
|:-----------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `OP_DUP`         | **Duplicate**: Duplicates the top item on the stack.                                                                                                                                                          |
| `OP_HASH160`     | **Hash**: Performs a SHA-256 hash followed by a RIPEMD-160 hash on the top stack item. This is how Bitcoin addresses are generated.                                                                       |
| `OP_EQUALVERIFY` | **Equal & Verify**: Pops the top two items and compares them. If they are not equal, the script fails immediately.                                                                                          |
| `OP_CHECKSIG`    | **Check Signature**: Pops a public key and a signature. It uses the public key to verify that the signature is valid for the current transaction. This is the core of authorization.                             |

### P2PKH Script Execution Example
A typical Pay-to-Public-Key-Hash (P2PKH) transaction perfectly demonstrates how these opcodes work together.

1.  **The Lock (`scriptPubKey`)**: `OP_DUP OP_HASH160 <PubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
2.  **The Key (`scriptSig`)**: `<Signature> <PublicKey>`
3.  **Execution Steps**:
    *   The `scriptSig` pushes the signature and public key onto the stack.
    *   `OP_DUP` duplicates the public key.
    *   `OP_HASH160` hashes the top public key, yielding the spender-provided public key hash.
    *   The `<PubKeyHash>` from the `scriptPubKey` is pushed onto the stack.
    *   `OP_EQUALVERIFY` compares the two hashes. If they match, execution continues. This verifies the spender is the owner of the address.
    *   `OP_CHECKSIG` validates the signature, ensuring the transaction was authorized by the private key holder.
    *   The stack is left with a single `TRUE` value, and the validation succeeds.

---

## Sections 3 & 4: Blocks, Timestamp Server, and Proof-of-Work

### Core "Block" Data Structures
*   **Block Header**: `CBlockHeader`
*   **Full Block**: `CBlock` (inherits from `CBlockHeader` and adds a transaction list `vtx`)
*   **File Path**: `bitcoin/src/primitives/block.h`

### Block Header Structure Analysis
| Member Variable | C++ Type    | Brief Description                                                                                                                                                             |
|:----------------|:------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `nVersion`      | `int32_t`   | **Block Version**: Allows for new consensus rules to be introduced.                                                                                                         |
| `hashPrevBlock` | `uint256`   | **Hash of Previous Block**: The key that links blocks into a "chain," implementing the whitepaper's "timestamp server" concept.                                               |
| `hashMerkleRoot`| `uint256`   | **Merkle Root**: A single, fixed-length hash that represents all transactions in the block.                                                                                 |
| `nTime`         | `uint32_t`  | **Timestamp**: The approximate creation time of the block (Unix timestamp).                                                                                                   |
| `nBits`         | `uint32_t`  | **Difficulty Target (encoded)**: Defines the difficulty target that the block's hash must meet.                                                                             |
| `nNonce`        | `uint32_t`  | **Nonce**: The field that miners can freely change while searching for a valid hash.                                                                                          |

### The Merkle Root
*   **Purpose**: A Merkle Tree (a hash tree) is used to condense all transaction information in a block into a single, fixed-length hash.
*   **Advantage**: The key advantage is enabling **Simple Payment Verification (SPV)**. A light client can verify that a transaction is included in a block by downloading only the block headers and requesting a "Merkle proof" (the necessary hashes to connect the transaction to the root) from a full node.

### Proof-of-Work Validation Logic
*   **Core Function**: `CheckProofOfWork`
*   **File Path**: `bitcoin/src/pow.cpp`
*   **Functionality**: This function decodes the `nBits` from the block header into a 256-bit target and compares the block's hash to it. A block is valid only if its hash is **less than or equal to** the target.

---
## Sections 5 & 6: Network and Incentive

### Block Propagation - The "Gossip" Protocol
The Bitcoin network uses an efficient "gossip" protocol to propagate new blocks, saving bandwidth and resisting attacks.
1.  **`inv` (Inventory)**: A miner sends an `inv` message to its peers, containing only the hash of the new block.
2.  **`getdata`**: A node that receives the `inv` and doesn't have the block responds with a `getdata` message to request the full data.
3.  **`block`**: The original node then sends the full `block` message upon receiving the `getdata` request.
This process ensures nodes don't download data they already have.

### Block Validation and Acceptance
*   **Core Coordinating Function**: `ChainstateManager::ProcessNewBlock` (in `src/validation.cpp`)
*   **Main Validation Stages**:
    1.  **Check Block Structure (`CheckBlock`)**: Verifies PoW, size, timestamp, etc.
    2.  **Check Block Contents**: Validates all transactions within the block (signatures, scripts, UTXO availability).
    3.  **Connect to Blockchain (`AcceptBlock`)**: Updates the UTXO set and sets the new block as the new chain tip.

### The "Longest Chain is the Truth" Rule
In code, this is more accurately the chain with the **most cumulative proof-of-work**. Each block index `CBlockIndex` contains an `nChainWork` member.
*   **Chain Reorganization (Reorg)**: When an alternative fork's `nChainWork` surpasses the current main chain, a reorg automatically occurs:
    1.  **Disconnect Old Blocks**: Blocks are rolled back from the tip of the old main chain, and their transactions are returned to the mempool.
    2.  **Connect New Blocks**: Blocks from the new, longer chain are connected from the fork point, becoming the new main chain.
This automated process ensures the network always converges on a single, authoritative history.

### The Incentive - The Coinbase Transaction
The incentive for miners comes from the Coinbase transaction.
*   **Uniqueness**:
    *   **Input**: A coinbase transaction has no regular inputs; it creates new bitcoin "out of thin air." Its `prevout` field is null.
    *   **`scriptSig`**: Its content is customizable by the miner and is called "coinbase data."
*   **Reward and Fees**:
    *   **Block Subsidy**: The protocol-defined amount of new bitcoin created (e.g., initially 50 BTC, halving every 210,000 blocks). Calculated by the `GetBlockSubsidy` function.
    *   **Transaction Fees**: The sum of all (inputs - outputs) for every other transaction in the block.
    The miner combines these two values and pays them to their own address in the output of the coinbase transaction.

---

## Advanced Topic: P2SH (Pay-to-Script-Hash)

P2SH (BIP 16) dramatically improved Bitcoin's usability by shifting the responsibility for complex scripts from the sender to the receiver.

### P2SH Transaction Structure
*   **The Lock (P2SH `scriptPubKey`)**: `OP_HASH160 <20-byte hash of redeemScript> OP_EQUAL`
    It locks funds to the hash of a script.
*   **The Key (P2SH `scriptSig`)**: `<Signature(s)> <Full redeemScript>`
    To spend, one must provide both the signatures that satisfy the redeem script and the full redeem script itself.

### P2SH Validation Logic
This is a two-stage process triggered by special rules in the `VerifyScript` function.
1.  **Stage 1: Validate Script Hash**:
    *   The interpreter executes `<scriptSig> + <scriptPubKey>`.
    *   It checks if the hash of the provided `redeemScript` matches the hash in the `scriptPubKey`.
    *   If it matches and the script format is P2SH, it proceeds to stage two.
2.  **Stage 2: Validate Redeem Script**:
    *   The interpreter takes the `redeemScript` and treats it as the new script to be executed.
    *   It uses the signatures from the `scriptSig` as the inputs for this new script.
    *   The final result of this **second** script execution determines the transaction's validity.

---

## Advanced Topic: Timelocks
Timelocks (BIP 65/112) introduced a time dimension to Bitcoin scripts, becoming a cornerstone of Layer 2 protocols.

### `OP_CHECKLOCKTIMEVERIFY` (CLTV) - Absolute Timelock
*   **Purpose**: Locks a UTXO until a future **absolute point in time** (a timestamp or block height).
*   **Mechanism**: The CLTV opcode compares a value from the stack to the spending transaction's `nLockTime` field. `nLockTime` must be greater than or equal to the stack value.
*   **Use Case**: Trustless escrows, payment channel setup.

### `OP_CHECKSEQUENCEVERIFY` (CSV) - Relative Timelock
*   **Purpose**: Locks a UTXO for a **relative duration** (a number of blocks or seconds) from the moment it is confirmed.
*   **Mechanism**: The CSV opcode compares a value from the stack to the spending transaction's input's `nSequence` field.
*   **Use Case**: Payment channel penalty/dispute mechanisms and other contracts that depend on the sequence of events.

### HTLC (Hashed Time Locked Contract)
HTLCs are the building blocks of the Lightning Network, combining a hash lock with a timelock. An HTLC script has two spending paths:
1.  **Success Path**: The recipient (Bob) can spend the funds before a deadline if he provides a secret pre-image that matches a hash.
2.  **Timeout/Refund Path**: If the deadline passes and the recipient has not provided the secret, the original funder (Alice) can reclaim her money.

The refund path in an HTLC typically uses **CLTV (absolute timelock)** because it provides an unambiguous, globally-agreed-upon deadline for all parties in the contract.

---

## Advanced Topic: Segregated Witness (SegWit)
SegWit (BIP 141-144) is one of the most significant upgrades in Bitcoin's history.

### The "Transaction Malleability" Bug
*   **The Problem**: Before SegWit, the transaction signature (`scriptSig`) was part of the data used to calculate the transaction ID (`txid`). Since a signature could have multiple valid encodings, a third party could alter it without invalidating it, which would change the `txid`.
*   **The Danger**: This was fatal for Layer 2 protocols like the Lightning Network, which rely on the immutability of unconfirmed transaction IDs to chain contracts together.

### The SegWit Solution
*   **Core Idea**: **Segregate** the signature ("witness") data from the main transaction body and place it in a new `scriptWitness` structure.
*   **`txid` vs `wtxid`**:
    *   `txid`: Is now calculated from the transaction data *without* the witness. This makes it immutable and fixes the malleability bug.
    *   `wtxid`: Is a new hash of the full transaction *including* the witness, used for network transport and block construction.

### The Clever Soft Fork Implementation
SegWit hid its new rules from old nodes using an "anyone-can-spend" script pattern.
*   **P2WPKH `scriptPubKey`**: `OP_0 <20-byte public key hash>`
*   **Old Node's View**: Sees this as a script that can always be unlocked (with an empty `scriptSig`), so it doesn't reject it.
*   **New Node's View**: Recognizes this as a SegWit V0 program and looks for the `scriptWitness` data to validate the transaction according to the new, stricter rules.

### The Benefits of SegWit
*   **Effective Block Size Increase**: Introduced **Block Weight**, where non-witness data bytes count as 4 weight units, but witness data bytes only count as 1. With a 4 million weight unit limit, this allows blocks to contain nearly 4MB of transaction data, effectively increasing capacity.
*   **Script Versioning**: Opcodes `OP_0` through `OP_16` were reserved as witness program version numbers. This made future script upgrades (like Taproot, which used `OP_1`) much cleaner and simpler to implement as soft forks.

---

## Wallet Technology: The Gateway to User Experience

### HD Wallets (BIP 32 & BIP 44)
*   **Problem Solved**: Replaced the "Just a Bunch of Keys" (JBOK) model, which required backing up every single private key.
*   **Core Concepts**:
    *   **BIP 32**: Derives an entire tree of keys from a single **master seed** in a deterministic way. One backup, for life.
    *   **BIP 44**: Defines a standardized, multi-purpose tree structure: `m/purpose'/coin_type'/account'/change/address_index`, allowing for multi-currency and multi-account compatibility from a single seed.

### Mnemonic Code (BIP 39)
*   **Purpose**: Makes wallet backup and recovery user-friendly by encoding the binary seed into a human-readable **mnemonic phrase** of 12 or 24 common words.

### Behind-the-Scenes Transaction Construction
1.  **Coin Selection**: Because UTXOs are indivisible, the wallet must solve an optimization problem: which of the user's UTXOs should it use as inputs to meet the payment amount while minimizing fees and protecting privacy?
2.  **Fee Estimation**: The wallet estimates the required fee rate (in sat/vByte) by analyzing the mempool of its own full node or by querying a trusted third-party API.
3.  **Change Output**: Because UTXOs must be spent in their entirety, any amount leftover after the payment and fee are accounted for must be sent back to the user. The wallet automatically creates a "change" output to a new, internally-controlled address.

---

## Historical Perspective: Major Upgrades & Hard Forks

### Major Upgrades Within Bitcoin Core (BTC)
*   **P2SH**: To improve the usability and privacy of complex scripts.
*   **CLTV / CSV**: To enhance smart contract capabilities with absolute and relative timelocks.
*   **SegWit**: To fix transaction malleability, increase block capacity, and enable script versioning.
*   **Taproot**: To improve privacy, efficiency, and script flexibility with Schnorr Signatures and MAST.

### Major Hard Fork Projects

#### Bitcoin Cash (BCH)
*   **Motivation**: A fundamental disagreement on scaling. Proponents believed in scaling by directly **increasing the base-layer block size limit** to maintain low on-chain fees, better fulfilling the "peer-to-peer electronic cash" vision.
*   **Technical Differences**: Increased block size limit (to 32MB); removed SegWit; introduced new opcodes like `OP_CHECKDATASIG`.
*   **Philosophy**: Massive on-chain scaling is the best path to becoming a world currency.

#### Bitcoin SV (BSV)
*   **Motivation**: A belief that BCH was not radical enough in its pursuit of the "original Satoshi Vision." Proponents advocate for removing all protocol limits and letting the network scale freely via market forces.
*   **Technical Differences**: **Completely removed** the block size cap; advocates for "locking" the protocol to maintain extreme stability; restored all original opcodes.
*   **Philosophy**: To realize the "Metanet," a global, value-based internet built on the BSV blockchain, capable of handling all forms of data and commerce on-chain.

</rewritten_file>