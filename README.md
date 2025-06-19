# Bitcoin: From Theory to Code - An Implementation-Level Commentary

This document provides an in-depth, code-level technical commentary on Satoshi Nakamoto's Bitcoin whitepaper. It connects the high-level concepts from the paper to the specific C++ implementations within the Bitcoin Core client, bridging the gap between theory and engineering practice.

## Table of Contents
1. [Introduction](#1-introduction)
2. [Transactions](#2-transactions)
3. [Timestamp Server](#3-timestamp-server)
4. [Proof-of-Work](#4-proof-of-work)
5. [Network](#5-network)
6. [Incentive](#6-incentive)
7. [Reclaiming Disk Space](#7-reclaiming-disk-space)
8. [Simplified Payment Verification](#8-simplified-payment-verification)
9. [Combining and Splitting Value](#9-combining-and-splitting-value)
10. [Privacy](#10-privacy)
11. [Calculations](#11-calculations)
12. [Conclusion](#12-conclusion)
13. [Advanced Topics & Historical Perspective](#advanced-topics--historical-perspective)

---

## Abstract

> A purely peer-to-peer version of electronic cash would allow online payments to be sent directly from one party to another without going through a financial institution. Digital signatures provide part of the solution, but the main benefits are lost if a trusted third party is still required to prevent double-spending. We propose a solution to the double-spending problem using a peer-to-peer network. The network timestamps transactions by hashing them into an ongoing chain of hash-based proof-of-work, forming a record that cannot be changed without redoing the proof-of-work. The longest chain not only serves as proof of the sequence of events witnessed, but proof that it came from the largest pool of CPU power. As long as a majority of CPU power is controlled by nodes that are not cooperating to attack the network, they'll generate the longest chain and outpace attackers. The network itself requires minimal structure. Messages are broadcast on a best effort basis, and nodes can leave and rejoin the network at will, accepting the longest proof-of-work chain as proof of what happened while they were gone.

## 1. Introduction

> Commerce on the Internet has come to rely almost exclusively on financial institutions serving as trusted third parties to process electronic payments. While the system works well enough for most transactions, it still suffers from the inherent weaknesses of the trust based model. Completely non-reversible transactions are not really possible, since financial institutions cannot avoid mediating disputes. The cost of mediation increases transaction costs, limiting the minimum practical transaction size and cutting off the possibility for small casual transactions, and there is a broader cost in the loss of ability to make non-reversible payments for non-reversible services. With the possibility of reversal, the need for trust spreads. Merchants must be wary of their customers, hassling them for more information than they would otherwise need. A certain percentage of fraud is accepted as unavoidable. These costs and payment uncertainties can be avoided in person by using physical currency, but no mechanism exists to make payments over a communications channel without a trusted party.
>
> What is needed is an electronic payment system based on cryptographic proof instead of trust, allowing any two willing parties to transact directly with each other without the need for a trusted third party. Transactions that are computationally impractical to reverse would protect sellers from fraud, and routine escrow mechanisms could easily be implemented to protect buyers. In this paper, we propose a solution to the double-spending problem using a peer-to-peer distributed timestamp server to generate computational proof of the chronological order of transactions. The system is secure as long as honest nodes collectively control more CPU power than any cooperating group of attacker nodes.

### **Commentary: The Promise, The Problem, and The Synthesis**

This introduction lays out Bitcoin's entire thesis in three parts: the promise of a peer-to-peer system, the core problem it must solve (double-spending), and the high-level overview of the solution (a grand synthesis of five key components).

#### The Promise of a Peer-to-Peer System
> *"A purely peer-to-peer version of electronic cash would allow online payments to be sent directly from one party to another without going through a financial institution."*

This opening statement is Bitcoin's declaration of independence. It's more than a vision; every word is supported by specific code and protocol rules.

*   **"Peer-to-Peer" Implementation**: The Bitcoin network has no central server. It's a flat network of thousands of peer nodes.
    *   **Network Layer**: Nodes discover and connect to each other through logic found in `src/net.cpp` and `src/net_processing.cpp`. Each node maintains a list of peers and exchanges information directly.
    *   **Information Propagation**: The spread of transactions and blocks relies on the **Gossip Protocol**. When a node has new information, it announces it to its peers with an `inv` message. Nodes that don't have the data request it with `getdata`. This mechanism prevents broadcast storms and allows information to propagate efficiently without a central coordinator.

*   **"Without a financial institution" Implementation**: Bitcoin makes the role of a traditional bank obsolete by decentralizing its three core functions: custody, authorization, and settlement.
    1.  **Custody**: Users hold their own funds via **wallets**. The **HD Wallet (BIP 32/39)** technology is key here. A user's funds are tied not to an intermediary, but to a **master seed** and the private keys derived from it. The private key *is* ownership. As the saying goes: "Not your keys, not your coins."
    2.  **Authorization**: Authorization is a purely cryptographic process. The **Script system** (`src/script/interpreter.cpp`) is at its heart. A transaction is valid if and only if the spender can provide a valid `scriptSig` to unlock the `scriptPubKey` of a previous output. The core operation, **`OP_CHECKSIG`**, doesn't care who you are, only that your digital signature corresponds to the public key.
    3.  **Settlement**: Settlement is achieved through a decentralized **consensus mechanism**, the process by which all honest nodes agree on a single transaction history.

#### The Core Problem: Double-Spending
> *"Digital signatures provide part of the solution, but the main benefits are lost if a trusted third party is still required to prevent double-spending."*

*   **What "Double-Spending" Is**: In the digital realm, any information can be perfectly copied. **Double-spending** refers to a malicious actor sending the same "digital coin" to two or more different recipients. For example, Alice pays Bob, and then immediately pays Carol with the *same* funds.
*   **Why Digital Signatures Are Insufficient**: A digital signature proves **ownership**, but it cannot prevent the owner from creating two separate, validly signed transactions that both attempt to spend the same funds. Without a global, unified view of all transactions, there's no way to know which one came first.

#### The Solution: A Grand Synthesis
> *"...a solution to the double-spending problem using a peer-to-peer network... The network timestamps transactions by hashing them into an ongoing chain of hash-based proof-of-work..."*

Bitcoin's solution is an elegant system of five pillars working in synergy:

1.  **The Public Ledger (The Blockchain)**: A global, append-only ledger that everyone can see, making all transactions public and any double-spend attempts transparent.
2.  **The UTXO Model**: Instead of accounts, Bitcoin uses an Unspent Transaction Output model, where each "coin" is a unique, indivisible object. This simplifies double-spend detection: you just need to check if a specific UTXO has already been spent.
3.  **The Timestamp Server (The Chain)**: By including the previous block's hash (`hashPrevBlock` in `CBlockHeader`), blocks are chronologically linked, creating a definitive history that is computationally difficult to reorder.
4.  **Proof-of-Work (The Cost)**: By requiring miners to expend real-world energy to find a `nNonce` that results in a valid block hash (verified by `CheckProofOfWork`), the protocol makes it costly and difficult to add new blocks, securing the ledger against malicious additions.
5.  **The Longest Chain Rule (The Consensus)**: This simple rule—that the chain with the most cumulative proof-of-work (`nChainWork`) is the authoritative one—allows the decentralized network to spontaneously agree on a single version of history, resolving any temporary conflicts.

**In summary**: The UTXO model defines what a double-spend is, the public ledger makes it visible, the timestamp chain orders it, proof-of-work makes that order costly to forge, and the longest chain rule allows everyone to agree on the final order.

---

## 2. Transactions

> We define an electronic coin as a chain of digital signatures. Each owner transfers the coin to the next by digitally signing a hash of the previous transaction and the public key of the next owner and adding these to the end of the coin. A payee can verify the signatures to verify the chain of ownership.
>
> ![Transaction Chain](https://bitcoin.org/img/bitcoin-paper/transaction_chain.png)
>
> The problem of course is the payee can't verify that one of the owners did not double-spend the coin. A common solution is to introduce a trusted central authority, or mint, that checks every transaction for double spending. After each transaction, the coin must be returned to the mint to issue a new coin, and only coins issued directly from the mint are trusted not to be double-spent. The problem with this solution is that the fate of the entire money system depends on the company running the mint, with every transaction having to go through them, just like a bank.
>
> We need a way for the payee to know that the previous owners did not sign any earlier transactions. For our purposes, the earliest transaction is the one that counts, so we don't care about later attempts to double-spend. The only way to confirm the absence of a transaction is to be aware of all transactions. In the mint based model, the mint was aware of all transactions and decided which arrived first. To accomplish this without a trusted party, transactions must be publicly announced [1], and we need a system for participants to agree on a single history of the order in which they were received. The payee needs proof that at the time of each transaction, the majority of nodes agreed it was the first received.

### **Commentary: Transactions and The Script System**

The `CTransaction` class in Bitcoin Core is the direct implementation of the "chain of digital signatures" concept.

*   **Core Data Structure**: `CTransaction` in `bitcoin/src/primitives/transaction.h`
*   **Data Structure Analysis**:

| Member Variable | C++ Type                    | Brief Description                                                                 |
|:----------------|:----------------------------|:----------------------------------------------------------------------------------|
| `version`       | `const uint32_t`            | Transaction version number. Enables protocol upgrades like SegWit and Taproot.    |
| `vin`           | `const std::vector<CTxIn>`  | Transaction inputs, each referencing a previous transaction's output (a UTXO).    |
| `vout`          | `const std::vector<CTxOut>` | Transaction outputs, which create new UTXOs that can be spent in the future.    |
| `nLockTime`     | `const uint32_t`            | A timelock that specifies the earliest time or block height the transaction can be mined. |

*   **The Script System**: The actual "digital signature chain" is enforced by Bitcoin's Script language.
    *   Each output (`vout`) contains a locking script (`scriptPubKey`) that defines the conditions for spending.

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

## 3. Timestamp Server

> The solution we propose begins with a timestamp server. A timestamp server works by taking a hash of a block of items to be timestamped and widely publishing the hash, such as in a newspaper or Usenet post [2-5]. The timestamp proves that the data must have existed at the time, obviously, in order to get into the hash. Each timestamp includes the previous timestamp in its hash, forming a chain, with each additional timestamp reinforcing the ones before it.
>
> ![Timestamp Chain](https://bitcoin.org/img/bitcoin-paper/timestamp_chain.png)

### **Commentary: The Blockchain Implementation**

The "chain of timestamps" is the blockchain itself. This concept is implemented in the `CBlockHeader` data structure.

*   **Core Data Structure**: `CBlockHeader` in `bitcoin/src/primitives/block.h`.
*   **The Link**: The `hashPrevBlock` member of the block header contains the hash of the preceding block. This direct reference creates the chain, forming an ordered, chronological sequence of transaction batches. Each new block acts as a new "timestamp," reinforcing the entire history before it. Tampering with a block would change its hash, breaking the link and invalidating all subsequent blocks, which is the foundation of the blockchain's immutability.

---

## 4. Proof-of-Work

> To implement a distributed timestamp server on a peer-to-peer basis, we will need to use a proof-of-work system similar to Adam Back's Hashcash [6], rather than newspaper or Usenet posts. The proof-of-work involves scanning for a value that when hashed, such as with SHA-256, the hash begins with a number of zero bits. The average work required is exponential in the number of zero bits required and can be verified by executing a single hash.
>
> For our timestamp network, we implement the proof-of-work by incrementing a nonce in the block until a value is found that gives the block's hash the required zero bits. Once the CPU effort has been expended to make it satisfy the proof-of-work, the block cannot be changed without redoing the work. As later blocks are chained after it, the work to change the block would include redoing all the blocks after it.
>
> ![Proof-of-Work Chain](https://bitcoin.org/img/bitcoin-paper/proof-of-work_chain.png)
>
> The proof-of-work also solves the problem of determining representation in majority decision making. If the majority were based on one-IP-address-one-vote, it could be subverted by anyone able to allocate many IPs. Proof-of-work is essentially one-CPU-one-vote. The majority decision is represented by the longest chain, which has the greatest proof-of-work effort invested in it. If a majority of CPU power is controlled by honest nodes, the honest chain will grow the fastest and outpace any competing chains. To modify a past block, an attacker would have to redo the proof-of-work of the block and all blocks after it and then catch up with and surpass the work of the honest nodes. We will show later that the probability of a slower attacker catching up diminishes exponentially as subsequent blocks are added.
>
> To compensate for increasing hardware speed and varying interest in running nodes over time, the proof-of-work difficulty is determined by a moving average targeting an average number of blocks per hour. If they're generated too fast, the difficulty increases.

### **Commentary: Securing the Chain with Energy**

Proof-of-Work (PoW) is the mechanism that makes the timestamp chain secure and hard to rewrite. It brilliantly solves the problem of decentralized consensus by making representation proportional to computational power, or "one-CPU-one-vote."

*   **Implementation**:
    *   **Block Header Fields**: The `CBlockHeader` is the data structure that is hashed. The `nBits` field defines the difficulty target in a compact format. The `nNonce` is the number that miners iterate to change the block header's hash and find a valid solution.
    *   **Validation Function**: The `CheckProofOfWork` function in `src/pow.cpp` is the core of PoW validation. It takes the block's hash and `nBits`, decodes `nBits` into the full 256-bit target, and returns `true` only if the block hash is less than or equal to this target.
*   **Merkle Root Integration**: The `hashMerkleRoot` field in the header is crucial. It's the root of a Merkle tree built from all transaction IDs in the block. This efficiently commits all transactions to the block's proof-of-work. Changing even a single transaction would alter the Merkle root, thus invalidating the PoW and requiring the immense computational work to be redone. This is what secures the transaction history.

---

## 5. Network

> The steps to run the network are as follows:
> 1. New transactions are broadcast to all nodes.
> 2. Each node collects new transactions into a block.
> 3. Each node works on finding a difficult proof-of-work for its block.
> 4. When a node finds a proof-of-work, it broadcasts the block to all nodes.
> 5. Nodes accept the block only if all transactions in it are valid and not already spent.
> 6. Nodes express their acceptance of the block by working on creating the next block in the chain, using the hash of the accepted block as the previous hash.
>
> Nodes always consider the longest chain to be the correct one and will keep working on extending it. If two nodes broadcast different versions of the next block simultaneously, some nodes may receive one or the other first. In that case, they work on the first one they received, but save the other branch in case it becomes longer. The tie will be broken when the next proof-of-work is found and one branch becomes longer; the nodes that were working on the other branch will then switch to the longer one.
>
> New transaction broadcasts do not necessarily need to reach all nodes. As long as they reach many nodes, they will get into a block before long. Block broadcasts are also tolerant of dropped messages. If a node does not receive a block, it will request it when it receives the next block and realizes it missed one.

### **Commentary: The Network Lifecycle in Code**

This section describes the full lifecycle of transactions and blocks, from creation to consensus.

*   **Propagation (Steps 1 & 4)**: The network uses a gossip protocol (`inv`, `getdata`, `block`) to efficiently propagate new transactions and blocks without overwhelming nodes with redundant data.
*   **Validation and Acceptance (Step 5)**: The `ChainstateManager::ProcessNewBlock` function in `src/validation.cpp` orchestrates the rigorous validation process a new block must pass. This includes checking the proof-of-work (`CheckProofOfWork`), validating every single transaction (`CheckTransaction` and `Consensus::CheckTxInputs`), and ensuring it connects to a known block in the chain.
*   **Consensus (Step 6)**: The "longest chain" is the one with the most cumulative work (`nChainWork`). When a new block arrives that makes an alternative chain "heavier," a **chain reorganization** is automatically triggered to switch to the new, more authoritative chain. This ensures the decentralized network always converges on a single history.

---

## 6. Incentive

> By convention, the first transaction in a block is a special transaction that starts a new coin owned by the creator of the block. This adds an incentive for nodes to support the network, and provides a way to initially distribute coins into circulation, since there is no central authority to issue them. The steady addition of a constant of amount of new coins is analogous to gold miners expending resources to add gold to circulation. In our case, it is CPU time and electricity that is expended.
>
> The incentive can also be funded with transaction fees. If the output value of a transaction is less than its input value, the difference is a transaction fee that is added to the incentive value of the block containing the transaction. Once a predetermined number of coins have entered circulation, the incentive can transition entirely to transaction fees and be completely inflation free.
>
> The incentive may help encourage nodes to stay honest. If a greedy attacker is able to assemble more CPU power than all the honest nodes, he would have to choose between using it to defraud people by stealing back his payments, or using it to generate new coins. He ought to find it more profitable to play by the rules, such rules that favour him with more new coins than everyone else combined, than to undermine the system and the validity of his own wealth.

### **Commentary: The Economic Engine**

The incentive structure is what powers and secures the entire network. It's implemented through the special **Coinbase Transaction**.

*   **Uniqueness**: It is the only transaction that does not have a real input; it creates new value. Its `vin` references a null `prevout`, checked by the `IsCoinBase()` method.
*   **The Reward**: The output value of the coinbase transaction is the sum of two components:
    1.  **Block Subsidy**: A fixed amount of new bitcoin, defined by the protocol and calculated by `GetBlockSubsidy` based on the block height. This amount halves approximately every four years (210,000 blocks).
    2.  **Transaction Fees**: The sum of all fees from every other transaction included in the block. The fee for a single transaction is the value of its inputs minus the value of its outputs.
*   **Security Model**: This incentive mechanism ensures that it is almost always more profitable for a miner (even a majority one) to play by the rules and collect the block rewards than to attack the network, which would devalue the very coins they are trying to acquire.

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

## 7. Reclaiming Disk Space

> Once the latest transaction in a coin is buried under enough blocks, the spent transactions before it can be discarded to save disk space. To facilitate this without breaking the block's hash, transactions are hashed in a Merkle Tree [7][2][5], with only the root included in the block's hash. Old blocks can then be compacted by stubbing off branches of the tree. The interior hashes do not need to be stored.
>
> ![Merkle Tree](https://bitcoin.org/img/bitcoin-paper/merkle_tree.png)
>
> A block header with no transactions would be about 80 bytes. If we suppose blocks are generated every 10 minutes, 80 bytes * 6 * 24 * 365 = 4.2MB per year. With computer systems typically selling with 2GB of RAM as of 2008, and Moore's Law predicting current growth of 1.2GB per year, storage should not be a problem even if the block headers must be kept in memory.

### **Commentary: Pruning and SPV**

This section describes two key concepts for efficiency and light clients.

*   **Merkle Tree**: As previously analyzed, the `hashMerkleRoot` in the block header is the root of a Merkle tree built from all the transaction IDs (`txid`) in the block.
*   **Pruning**: A full node can operate in "pruning" mode, where it discards old, spent transaction data from its disk to save space, while still retaining the block headers and a set of recent blocks to fully validate the chain.
*   **SPV (Simplified Payment Verification)**: The use of a Merkle root is the critical innovation that allows for light wallets. An SPV client only needs the chain of 80-byte block headers, not the entire multi-gigabyte blockchain, to verify a payment with a high degree of security.

## 8. Simplified Payment Verification

> It is possible to verify payments without running a full network node. A user only needs to keep a copy of the block headers of the longest proof-of-work chain, which he can get by querying network nodes until he's convinced he has the longest chain, and obtain the Merkle branch linking the transaction to the block it's timestamped in. He can't check the transaction for himself, but by linking it to a place in the chain, he can see that a network node has accepted it, and blocks added after it further confirm the network has accepted it.
>
> ![SPV](https://bitcoin.org/img/bitcoin-paper/simplified_payment_verification.png)
>
> As such, the verification is reliable as long as honest nodes control the network, but is more vulnerable if the network is overpowered by an attacker. While network nodes can verify transactions for themselves, the simplified method can be fooled by an attacker's fabricated transactions for as long as the attacker can continue to overpower the network. One strategy to protect against this would be to accept alerts from network nodes when they detect an invalid block, prompting the user's software to download the full block and alerted transactions to confirm the inconsistency. Businesses that receive frequent payments will probably still want to run their own nodes for more independent security and quicker verification.

### **Commentary: The Light Client Security Model**

This section elaborates on the SPV model. The security of an SPV client relies on the assumption that the chain with the most proof-of-work (the one whose headers it has) is the honest chain. It trusts the majority of the network's hash power to validate transactions correctly. While this is a very strong security assumption, it's not as absolute as a full node, which validates every transaction for itself. This trade-off is what makes lightweight, mobile-friendly Bitcoin wallets possible.

## 9. Combining and Splitting Value

> Although it would be possible to handle coins individually, it would be unwieldy to make a separate transaction for every cent in a transfer. To allow value to be split and combined, transactions contain multiple inputs and outputs. Normally there will be either a single input from a larger previous transaction or multiple inputs combining smaller amounts, and at most two outputs: one for the payment, and one returning the change, if any, back to the sender.
>
> ![Fan-out](https://bitcoin.org/img/bitcoin-paper/combining_splitting_value.png)
>
> It should be noted that fan-out, where a transaction depends on several transactions, and those transactions depend on many more, is not a problem here. There is never the need to extract a complete standalone copy of a transaction's history.

### **Commentary: The UTXO Model in Practice**

This section describes the practical application of the UTXO model.
*   **Inputs and Outputs**: A transaction consumes one or more UTXOs as inputs and creates one or more new UTXOs as outputs.
*   **Change Outputs**: Because UTXOs must be spent in their entirety, most transactions require a "change" output. If you want to pay 0.1 BTC but only have a 0.5 BTC UTXO, your transaction will consume the 0.5 BTC input and create two outputs: one paying 0.1 BTC to the recipient and another paying the remaining 0.4 BTC (less fees) back to a new address you control. This logic is handled automatically by modern wallets.

## 10. Privacy

> The traditional banking model achieves a level of privacy by limiting access to information to the parties involved and the trusted third party. The necessity to announce all transactions publicly precludes this method, but privacy can still be maintained by breaking the flow of information in another place: by keeping public keys anonymous. The public can see that someone is sending an amount to someone else, but without information linking the transaction to anyone. This is similar to the level of information released by stock exchanges, where the time and size of individual trades, the "tape", is made public, but without telling who the parties were.
>
> ![Privacy Models](https://bitcoin.org/img/bitcoin-paper/privacy.png)
>
> As an additional firewall, a new key pair should be used for each transaction to keep them from being linked to a common owner. Some linking is still unavoidable with multi-input transactions, which necessarily reveal that their inputs were owned by the same owner. The risk is that if the owner of a key is revealed, linking could reveal other transactions that belonged to the same owner.

### **Commentary: Pseudonymity in Practice**

Bitcoin is pseudonymous, not anonymous.
*   **Public Keys as Pseudonyms**: The protocol itself only knows about public keys (or their hashes, i.e., addresses). There is no direct link to a real-world identity.
*   **Address Reuse**: The whitepaper correctly identifies the core privacy risk: address reuse. If a user receives multiple payments to the same address, all those payments are publicly linked. If that address is ever tied to the user's identity, their entire transaction history for that address is revealed.
*   **Wallet's Role (BIP 32/44)**: Modern HD wallets are designed to mitigate this. They automatically generate a new address for every incoming transaction and for every change output, making it much harder to link transactions to a common owner. The `change` path (`m/44'/0'/0'/1/x`) is specifically for this purpose.

## 11. Calculations

> We consider the scenario of an attacker trying to generate an alternate chain faster than the honest chain. Even if this is accomplished, it does not throw the system open to arbitrary changes, such as creating value out of thin air or taking money that never belonged to the attacker. Nodes are not going to accept an invalid transaction as payment, and honest nodes will never accept a block containing them. An attacker can only try to change one of his own transactions to take back money he recently spent.
>
> The race between the honest chain and an attacker chain can be characterized as a Binomial Random Walk. The success event is the honest chain being extended by one block, increasing its lead by +1, and the failure event is the attacker's chain being extended by one block, reducing the gap by -1.
>
> The probability of an attacker catching up from a given deficit is analogous to a Gambler's Ruin problem. Suppose a gambler with unlimited credit starts at a deficit and plays potentially an infinite number of trials to try to reach breakeven. We can calculate the probability he ever reaches breakeven, or that an attacker ever catches up with the honest chain, as follows[8]:
>
> p = probability an honest node finds the next block
> q = probability the attacker finds the next block
> q_z = probability the attacker will ever catch up from z blocks behind
>
> ![Gambler's Ruin Formula](https://bitcoin.org/img/bitcoin-paper/gamblers_ruin.png)
>
> Given our assumption that p > q, the probability drops exponentially as the number of blocks the attacker has to catch up with increases. With the odds against him, if he doesn't make a lucky lunge forward early on, his chances become vanishingly small as he falls further behind.
>
> We now consider how long the recipient of a new transaction needs to wait before being sufficiently certain the sender can't change the transaction. We assume the sender is an attacker who wants to make the recipient believe he paid him for a while, then switch it to pay back to himself after some time has passed. The receiver will be alerted when that happens, but the sender hopes it will be too late.
>
> The receiver generates a new key pair and gives the public key to the sender shortly before signing. This prevents the sender from preparing a chain of blocks ahead of time by working on it continuously until he is lucky enough to get far enough ahead, then executing the transaction at that moment. Once the transaction is sent, the dishonest sender starts working in secret on a parallel chain containing an alternate version of his transaction.
>
> The recipient waits until the transaction has been added to a block and z blocks have been linked after it. He doesn't know the exact amount of progress the attacker has made, but assuming the honest blocks took the average expected time per block, the attacker's potential progress will be a Poisson distribution with expected value:
>
> ![Poisson Distribution Formula](https://bitcoin.org/img/bitcoin-paper/poisson_distribution.png)
>
> To get the probability the attacker could still catch up now, we multiply the Poisson density for each amount of progress he could have made by the probability he could catch up from that point:
>
> ![Attacker Catch-up Probability Formula](https://bitcoin.org/img/bitcoin-paper/attacker_catch-up_probability.png)
>
> Rearranging to avoid summing the infinite tail of the distribution…
>
> ![Attacker Catch-up Probability Simplified Formula](https://bitcoin.org/img/bitcoin-paper/attacker_catch-up_probability_simplified.png)
>
> Converting to C code…
>
> ```
> #include <math.h>
> double AttackerSuccessProbability(double q, int z)
> {
> 	double p = 1.0 - q;
> 	double lambda = z * (q / p);
> 	double sum = 1.0;
> 	int i, k;
> 	for (k = 0; k <= z; k++)
> 	{
> 		double poisson = exp(-lambda);
> 		for (i = 1; i <= k; i++)
> 			poisson *= lambda / i;
> 		sum -= poisson * (1 - pow(q / p, z - k));
> 	}
> 	return sum;
> }
> ```
>
> Running some results, we can see the probability drop off exponentially with z.
>
> ```
> q=0.1
> z=0    P=1.0000000
> z=1    P=0.2045873
> z=2    P=0.0509779
> z=3    P=0.0131722
> z=4    P=0.0034552  
> z=5    P=0.0009137
> z=6    P=0.0002428
> z=7    P=0.0000647
> z=8    P=0.0000173
> z=9    P=0.0000046
> z=10   P=0.0000012
> 
> q=0.3
> z=0    P=1.0000000
> z=5    P=0.1773523
> z=10   P=0.0416605
> z=15   P=0.0101008
> z=20   P=0.0024804
> z=25   P=0.0006132
> z=30   P=0.0001522
> z=35   P=0.0000379
> z=40   P=0.0000095
> z=45   P=0.0000024
> z=50   P=0.0000006
> ```
>
> Solving for P less than 0.1%…
>
> ```
> P < 0.001
> q=0.10   z=5
> q=0.15   z=8
> q=0.20   z=11
> q=0.25   z=15
> q=0.30   z=24
> q=0.35   z=41
> q=0.40   z=89
> q=0.45   z=340
> ```

### **Commentary: The Economics of Security**

This section provides the mathematical foundation for Bitcoin's security model, proving that the probability of an attacker successfully reversing a transaction decreases exponentially with each subsequent block (confirmation). The "6 block" rule of thumb is a practical application of this calculation, representing a point where the cost of an attack becomes astronomical and the probability of success becomes negligible for any attacker who doesn't control a majority of the hash rate.

---

## 12. Conclusion

> We have proposed a system for electronic transactions without relying on trust. We started with the usual framework of coins made from digital signatures, which provides strong control of ownership, but is incomplete without a way to prevent double-spending. To solve this, we proposed a peer-to-peer network using proof-of-work to record a public history of transactions that quickly becomes computationally impractical for an attacker to change if honest nodes control a majority of CPU power. The network is robust in its unstructured simplicity. Nodes work all at once with little coordination. They do not need to be identified, since messages are not routed to any particular place and only need to be delivered on a best effort basis. Nodes can leave and rejoin the network at will, accepting the proof-of-work chain as proof of what happened while they were gone. They vote with their CPU power, expressing their acceptance of valid blocks by working on extending them and rejecting invalid blocks by refusing to work on them. Any needed rules and incentives can be enforced with this consensus mechanism.

### **Commentary: A Final Synthesis**

The conclusion masterfully summarizes the entire system. It reiterates how the combination of digital signatures, a peer-to-peer network, proof-of-work, and the longest chain rule creates a robust, decentralized, and secure system for electronic cash, all without relying on a trusted third party. Our journey through the Bitcoin Core codebase has shown how each of these high-level concepts was translated into robust, battle-tested C++ code.

---

## Advanced Topics & Historical Perspective

### Advanced Topics
*   **P2SH**: Pay-to-Script-Hash
*   **Timelocks**: Absolute and Relative Timelocks
*   **Segregated Witness**: SegWit

### Historical Perspective
*   **Bitcoin Cash**: A hard fork that aimed to increase block size
*   **Bitcoin SV**: A hard fork that aimed to remove all protocol limits
