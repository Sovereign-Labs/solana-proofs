---
Title: "The Copy-on-Chain Mechanism for SPV Light Clients on Solana"
Created: "2023-11-24"
Authors: Dubbelosix, Preston Evans
---

## Abstract

At Sovereign Labs, in the course of research that required attestations on the Solana blockchain state, we have identified a novel way to build on-chain SPV ([Simple Payment Verification](https://docs.solana.com/proposals/simple-payment-and-state-verification)) light clients for the Solana blockchain. Crucially, this method does not necessitate changes to the Solana consensus protocol. We have developed a proof-of-concept, available in this repository, which aims to expedite the development of on-chain light clients on Solana by reducing the risks typically associated with modifications to the core protocol.

This solution utilizes the Copy-on-Chain (`copy`) program to generate a hash of the Solana account state. This hash is then included in the `accounts_delta_hash`, which is part of the `BankHash` attested by the validator set.  Using these proofs, users can submit a transaction to directly verify the state of an account without needing to fully trust the account state information communicated by an intermediary RPC provider.

## Background

The goal of a light client is to reduce trust assumptions when using centralized RPC endpoints. Rather than relying on a RPC provider to truthfully communicate information about the particular state of certain accounts (i.e. user’s balances, bridge accounts, etc), when a RPC provider communicates certain information about Solana state, users can request further cryptographic proof and an attestation from validators to directly check its veracity. For more background on this subject and its usecases, we recommend reading this documentation about [interchain transaction verification](https://docs.solana.com/proposals/interchain-transaction-verification) from Solana Labs and this [introduction to light clients by a16z](https://a16zcrypto.com/posts/article/an-introduction-to-light-clients/).

When we refer to the “state” of an account, we refer to the data stored in the account fields shown in the general account structure below:

```rust
use solana_sdk::pubkey::Pubkey;
use solana_sdk::account::Epoch;

pub struct Account {
    /// lamports in the account
    pub lamports: u64,
    /// data held in this account
    pub data: Vec<u8>,
    /// the program that owns this account. If executable, the program that loads this account.
    pub owner: Pubkey,
    /// this account's data contains a loaded program (and is now read-only)
    pub executable: bool,
    /// the epoch at which this account will next owe rent
    pub rent_epoch: Epoch,
}
```

For each new block, all accounts where state has been modified are included in the 16-ary merkle tree whose leaves are ordered and the root is called the `accounts_delta_hash`. The  `accounts_delta_hash` is further hashed into the `BankHash` along with the `parent_bankhash`, `num_sigs`, and `blockhash`. The `BankHash` is a cryptographic commitment that Solana validators vote on for each slot. Each BankHash is calculated as follows:
```
hashv(&[
    parent_bankhash
    accounts_delta_hash,
    num_sigs,
    blockhash,
])
```

## The Copy-on-Chain Program

Because an account may not change every slot, a mechanism is required to ensure its latest state can still be included in a recent `BankHash` for cryptographic proof generation. This is where the Copy-on-Chain (`copy`) program comes in. It takes an account's state, computes its hash, and copies it into a `CopyAccount`. The alteration of the `CopyAccount` ensures the account's state becomes part of the slot's `accounts_delta_hash`, and therefore also part of the`BankHash`, against which a proof can be generated for the user. Checking for an account's modification and its presence in the `BankHash` is equivalent to checking transaction status because an account is only modified when the transaction is successful. In summary, the purpose of the `copy` program is that calculating and writing the hash ends up modifying `CopyAccount` during a slot and commits to the hash stored in `CopyAccount` as part of the `accounts_delta_hash` which is then rolled into the `BankHash`.

* The `copy` program takes two main accounts as input
  * A global scoped PDA - `CopyAccount`
  * The account whose state we want the attestation for - `SourceAccount`

* The `copy` program has a single instruction
  1) `copy_hash` reads the fields of `SourceAccount`
  2) The contents of the `SourceAccount` are hashed
  3)  The hash is written into the `CopyAccount` data field
  Note: If there are multiple calls to `copy_hash` in the same block, the hashes are rolled together

* This means that we can now produce 
  1) A proof of `CopyAccount`'s state to the `accounts_delta_hash`
  2) A proof for `accounts_delta_hash` as part of the `BankHash`
  3) A set of validators that have attested to the above `BankHash`
    * Note that since each `BankHash` commits to the previous one, we don't need votes on the specific `BankHash` itself, but we can use any subsequent vote as well
    * The above is made somewhat easier because we have the `SlotHashes` on-chain account which contains a vector of recent `BankHashes`
    * This means we don't need to chain the BankHashes - we can instead take the vote on a BankHash and if the SlotHashes account for that block contains our `BankHash`, that should be sufficient.

* The below diagram indicates what the structure might look like when we want an attestation for 4 validators for `BankHash (n)`
  - BankHash (n) is associated with the slot in which the CopyAccount is created by the `copy` program.
  - Validators 1 and 2 land their votes in the slot that follows, which includes BankHash (n) in its `SlotHashes` field.
  - Validators 3 and 4 land their votes in later slots but these still include BankHash (n) in the `SlotHashes` field as well.
  - This same logic can be applied to a larger portion of the validator set to obtain supermajority guarantees assuming stake information is retrieved.
```
                                          Validator 1's vote
                                          Validator 2's vote        Validator 3's vote       Validator 4's vote
                                                 ||                         ||                      ||
BankHash (n-1)   <-   BankHash (n)    <-     BankHash (n+1)     <-     BankHash (n+2)    <-     BankHash (n+3)     <-     BankHash (n+4)
                          ||                     ||                         ||                      ||
                      CopyAccount            SlotHashes                 SlotHashes               SlotHashes              
                          ||               (should contain            (should contain         (should contain
                     SourceAccount          BankHash (n)               BankHash (n))            BankHash (n))
                        
```

## Current Prototype

1. [Geyser plugin](https://docs.solana.com/developing/plugins/geyser-plugins) to monitor updates and generate proofs

2. On-chain program to provide the copy hash functionality

3. The client will:
   * Get the state of an account from the RPC
   * Submit the copy_hash instruction
   * Open a connection to the geyser plugin and stream the proofs

## Implementation Notes and Needed Improvements

In our light client design, proofs are dynamically generated as they are strictly tied to the current block. This is because we can only create proofs to the root for the present block. If the Copy-on-Chain program doesn't replicate the state hash of interest in the subsequent block, the accounts_delta_hash will omit the commitments relevant to the targeted account(s). Although this approach might increase storage and processing demands, the centralization of the prover isn't a concern. Censorship remains the sole risk factor, given its f+1 vector, and the task of generating proofs can be efficiently integrated with a specific RPC or full node. Currently, for this proof of concept, we are utilizing a Geyser plugin for its simplicity and convenience.

Regarding the `copy` account, there can be a signer PDA so that each signer can have their own `copy` account or there can be more global shared `copy` accounts. If multiple tranasctions in the same slot hit the Copy-on-Chain program to prove the state of multiple accounts, they are rolled together as `account_hash = hash(account_hash | new)`. This allows individual accounts referenced by the Copy-on-Chain transactions in the block to each have their own proof.

In this initial implementation, we have configured the validators for capturing votes through a geyser configuration. This setup doesn't represent the ideal supermajority of votes we aim to achieve ultimately. However, it does reduce the level of trust a user must place in the data received from an RPC provider. Our goal is to enhance this to encompass a broader quorum of votes, informed by a known validator set and the stake weight of each validator. While this enhancement could await a SIMD implementation like the proposed stake sysvar, it could also be potentially incorporated into the [Epoch Accounts Hash](https://docs.solana.com/implemented-proposals/epoch_accounts_hash#:~:text=This%20will%20be%20known%20as,issues%20within%201%2D2%20epochs.), which doesn't require consensus changes but merits further investigation.

It's important to note that the current implementation doesn't incorporate sessions and is intended purely as a Proof of Concept (PoC). Additionally, the Merkle proof generation logic requires more testing to ensure that it's reliable. Equally crucial is the testing of the logic that manages account, slot, and block updates, transitioning them from raw to processed to confirmed states. There is potential for a more streamlined implementation that could directly integrate into the validator, attaching hooks for confirmed slots, which could simplify the overall process.

## Future Work

### Verifying State Consistency with Non-Inclusion Proofs

The current proof-of-concept only seeks to verify the state of an account at a single point in time. However, a natural requirement for SPV systems is to monitor the state of an account over a longer period. This can also be accommodated in our design. Since the set of modified accounts committed in the `accounts_delta_hash` is ordered lexicographically, it is possible to generate succinct non-inclusion proofs which show that a particular account was *not* modified in a particular block. By verifying one such proof for each block, a online light client can know the current state of an account, even if that account has not been modified for an arbitrary length of time. 

### Alternate Methods of Finding Account State

As described above, a light client can watch for changes to an account's state without sending any on-chain transactions. However, knowing that an account's state hasn't changed is not particularly useful unless you know the what the initial state was. In the previous section, we've described one method for finding that initial state - by using the copy-on-chain program. This program is useful because it only requires *read* access to the target account. However, in many circumstances it may be possible for a user to write to the account they wish to monitor - for example, by sending a single lamport to it. Since any change to the account state will cause it to be included directly in the `accounts_delta_hash`, this method is just as effective as the on-chain copy. However, it has the drawback of requiring write access to the account in question, which may increase contention. 

One drawback of the approach described in this document (and one which is shared by other proposed SPV proposals) is that any attempt to *read* account state usually requires at least one on-chain transaction. However, this need not always be the case. For "popular" accounts which experience frequent writes, clients may be content to simply wait for the next external interaction with the account. And for less frequently touched accounts, clients always have the option of waiting for an epoch boundary to query the state. (Since Solana creates a state commitment every epoch, RPC nodes can be modified to generate a merkle proof of an account's state at the epoch boundary without changes to consensus. However, this will require software changes.)

## Comparison with Existing Design

We close out this document by comparing our approach with the existing proposal for SPV on Solana. In particular, our proposal improves on the existing in three key ways:
- It functions better under write contention
- It provides better throughput, since it enables unlimited reads against an account for each write
- It does *not* require consensus changes

### Write Contention

One major drawback of the existing SPV proposal is that it functions poorly under write contention. By way of reminder, the existing Solana SPV solution requires a light client to submit a transaction verifying that the current state of an account has some particular value, and then check the transaction status to see if that assertion held. Unfortunately, this method can't account for very recent changes to an account's state; any writes to the account are racing with the read and can cause it to fail. Since many of the most "important" accounts on Solana are written several times per block, this SPV method is likely to require numerous retries before succeeding. On the other hand, our method works *better* when accounts change frequently, since these accounts show up in the `accounts_delta_hash` without any need for interaction by the client.

### Throughput

A second drawback to the existing SPV proposal is that it requires reads to go through consensus. This is wasteful, since there is already global consensus on the data to be read! Suppose
that the number of SPV client reads `S` and the number of validators is `V`. Under the current proposal, the total work performed is `Ω(S*V)`, even if all of the SPV clients are interested in the same account! In our regime, the work in that case will be `Ω(S + V)`, and the linear work can be offloaded to validators which are not participating in consensus. 

### Required Consensus Changes

Finally, the existing SPV proposal requires a consensus modification to add transaction receipts. On the other hand, our proposal works with the existing block structure.
