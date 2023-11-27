## About
This work is a byproduct of research that Sovereign was doing for a different feature. A subset of that research was useful for accomplishing attestations on state, so this is a WIP / POC to test out that idea and apply it for solana validator attestations on account state. The code is untested and mainly serves to illustrate the idea.

This is currently a WIP and documentation will be updated as other components are added (vote verification, committee determination)

## Background

* We want a subset of selected validators (or quorum) to sign off on the "state" of a specific account
* Every solana account has the general structure
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
* When we say "state" of the account, we refer to a commitment of all the above fields

## BankHash Commitments
* In solana validators vote on a BankHash for every slot. Each BankHash is calculated as follows
```
hashv(&[
    parent_bankhash
    accounts_delta_hash,
    num_sigs,
    blockhash,
])
```
* The `accounts_delta_hash` is a 16-ary merkle root of the accounts that have been modified in the current block
* Since the account we care about might not be modified in a specific slot (or even in the epoch), we make use of the `copy` on-chain program
* The `copy` program takes two main accounts as input
  * A global scoped PDA - `CopyAccount`
  * The account whose state we want the attestation for - `SourceAccount`
* The `copy` program has a single instruction
  * `copy_hash` which reads the fields of `SourceAccount`
  * Hashes the contents of the `SourceAccount` 
  * Writes the hash into `CopyAccount`
  * If there are multiple calls to `copy_hash` in the same block, the hashes are rolled together
* The point of the `copy` program is that calculating and writing the hash ends up modifying `CopyAccount` during a slot and commits to the hash stored in `CopyAccount` as part of the `accounts_delta_hash` which is then rolled into the `BankHash`
* This means that we can now produce 
  * A proof of `CopyAccount`s state to the `accounts_delta_hash`
  * A proof for `accounts_delta_hash` as part of the `BankHash`
  * A set of validators that have attested to the above `BankHash`
    * Note that since each `BankHash` commits to the previous one, we don't need votes on the specific `BankHash` itself, but we can use any subsequent vote as well
    * The above is made somewhat easier because we have the `SlotHashes` on-chain account which contains a vector of recent `BankHashes`
    * This means we don't need to chain the BankHashes - we can instead take the vote on a BankHash and if the SlotHashes for that BankHash contains our `BankHash`, that should be sufficient.

* The below diagram indicates what the structure might look like when we want an attestation for 4 validators for `BankHash2`
```
                                            validator2_vote              
                                            validator1_vote            validator3_vote         validator4_vote
                                                 ||                         ||                      ||
BankHash1  <-       BankHash2       <-      BankHash3          <-        BankHash4      <-        BankHash5       <-     BankHash6
                       ||                        ||                         ||                      ||
                    CopyAccount               SlotHashes                 SlotHashes               SlotHashes              
                       ||                    (should contain             (should contain        (should contain
                    SourceAccount             BankHash2                   BankHash2)             BankHash2)
                        
```

* The above proof structure should provide a reasonable amount of trust minimization through validator attestation
* The goal here is to not impact consensus protocol and not require any changes
* CLAIM
  * checking for an account's modification and it's presence in the BankHash is equivalent to checking transaction status because a non-gas account is only modified when the transaction is successful
  * since we need to fire a transaction for SPV anyway and cause a "write" the above is equivalent to using transaction status
  * using transaction status however has one benefit, which is that the proof generation doesn't need to be "in-flight"
    * reason: most RPCs store transaction history in the ledger, so it should be simple to fetch all the historical transactions needed to generate a merkle proof, but this is an implementation detail - they can just as easily store the merkle proofs for the `CopyAccount` for every block when its modified in-flight
  * prover centralization is not an issue (censorship is the only vector and its f+1) 
  * The above can be bundled with a specific RPC or a full node

## Some Implementation Details
* We use a geyser plugin for PoC, but this can be baked into the code for any RPC wishing to use this
* We need to generate the proofs in-flight because we can only generate the proofs to the root in the current block. By the next block, its already too late since we don't have the values of the accounts
* We currently set the validators whose votes we want to capture as part of the geyser config
  * This can be improved to capture a quorum of votes
  * This means the client needs to know the validators set and their stake distribution
  * This can wait until we have the stake sysvar, or it can theoretically be done as part of the Epoch accounts hash
* The current implementation doesn't have sessions and is meant purely as a PoC
* The merkle proof generation logic needs to be tested thoroughly
* The logic to handle account, slot, block updates and change them from raw -> processed -> confirmed needs to be tested
  * Locally, we never have forks, but in a production setting this happens frequently enough
  * A simpler implementation might be possible in plugging in directly into the validator and attaching the hooks for confirmed slots
  * Geyser was chosen just for the PoC


## Components
1. Geyser plugin to monitor updates and generate proofs
2. On-chain program to provide the copy hash functionality
3. Client to
   * Get the state of an account from the RPC
   * Submit the copy_hash instruction
   * Open a connection to the geyser plugin and stream the proofs
