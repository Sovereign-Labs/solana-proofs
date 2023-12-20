## Background
In order to prove data availability, we have an on-chain program that accepts chunks, calculates the merkle root once all the chunks are received on-chain and then updates a Program Derived Address (PDA) with the merkle root.
Solana does not have full state commitments every block, instead it has a commitment to the accounts that were modified within that block as part of the BankHash.
Since the PDA is modified everytime a full rollup blob is seen by solana (i.e. all the chunks), the root of the chunks will be committed to in the BankHash.
Details about how the PDA is modified are available in [README](README.md)

### BankHash
* The BankHash is the commitment chosen by the staked validators to vote on
* The BankHash is created by hashing the following components
```
let mut hash = hashv(&[
    self.parent_hash.as_ref(),
    accounts_delta_hash.0.as_ref(),
    &signature_count_buf,
    self.last_blockhash().as_ref(),
]);
```
 * `parent_hash` refers to the parent bankhash
 * `accounts_delta_hash` refers to the merkle root of the modified accounts in that block (these are sorted by account address)
 * `signature_count_buf` is the number of signatures in the block
 * `last_blockhash` is the "blockhash" - it's different from the bankhash and refers to the last PoH tick after interleaving all the transactions together.

### Note about terminology
* The naming in the solana labs client is slightly confusing with 3 terms (blockhash, bankhash, slothash)
* The names are also used inconsistently in geyser vs RPC
* In RPC we have blockhash and previous_blockhash which both refer to the PoH ticks (?)
* In geyser, we have the following structure
```
pub struct ReplicaBlockInfoV2<'a> {
    pub parent_slot: Slot,
    pub parent_blockhash: &'a str,
    pub slot: Slot,
    pub blockhash: &'a str,
    pub rewards: &'a [Reward],
    pub block_time: Option<UnixTimestamp>,
    pub block_height: Option<u64>,
    pub executed_transaction_count: u64,
}
```
* We have `parent_blockhash` and `blockhash` but when we see how the values are set in the code - 
```
  block_metadata_notifier.notify_block_metadata(
      bank.parent_slot(),
      &bank.parent_hash().to_string(),
      bank.slot(),
      &bank.last_blockhash().to_string(),
      &bank.rewards,
      Some(bank.clock().unix_timestamp),
      Some(bank.block_height()),
      bank.executed_transaction_count(),
  )

    fn notify_block_metadata(
        &self,
        parent_slot: u64,
        parent_blockhash: &str,
        slot: u64,
        blockhash: &str,
        rewards: &RwLock<Vec<(Pubkey, RewardInfo)>>,
        block_time: Option<UnixTimestamp>,
        block_height: Option<u64>,
        executed_transaction_count: u64,
    );
```
* If you observe - `parent_blockhash` is being set to `bank.parent_hash()` while `blockhash` is being set to `bank.last_blockhash()`
* This is slightly un-intuitive and also different from RPCs, so we would recommend anyone using these structures to get familiar with what they actually represent and not go purely by the naming convention


## Geyser Plugin
* We need to prove that the blob has been published to solana. This is accomplished by running a geyser plugin inside the solana validator.
* The geyser plugin tracks account updates as blocks are executed and merkle proofs are generated against the `accounts_delta_hash`
* The proofs generated for a Pubkey being monitored are of the form
```rust
#[derive(Clone, Debug, BorshSerialize, BorshDeserialize)]
pub enum AccountDeltaProof {
    /// Simplest proof for inclusion in the account delta hash
    InclusionProof(Pubkey, (Data, Proof)),
    /// Adjacency proof for non inclusion A C D E, non-inclusion for B means providing A and C
    NonInclusionProofInner(Pubkey, ((Data, Proof), (Data, Proof))),
    /// Left most leaf and proof
    NonInclusionProofLeft(Pubkey, (Data, Proof)),
    /// Right most leaf and proof. Also need to include hashes of all leaves to verify tree size
    NonInclusionProofRight(Pubkey, (Data, Proof, Vec<Hash>)),
}
```
* The code exists for `InclusionProof`, as well as the `NonInclusionProof`s, but only the inclusion is verified currently.

### Running the Geyser Plugin
* Build the geyser plugin - this is a `.dylib` (or `.so`) that implements the plugin interface and runs inside the solana validator
```bash
cd adapters/solana/account_proof_geyser
cargo build --release
```
* The plugin needs to be built with the same rust version used to build the solana validator. We have a `rust-toolchain.toml` pinning the rust version
* The dynamic lib should be found in `target/release`
```
ls -lahtr target/release/libaccount*
-rw-r--r--  1 username  staff   422B Oct 22 05:58 target/release/libaccount_proof_geyser.d
-rwxr-xr-x  1 username  staff   3.6M Oct 24 05:19 target/release/libaccount_proof_geyser.dylib
-rw-r--r--  1 username  staff    12M Oct 24 05:19 target/release/libaccount_proof_geyser.rlib
```
* The file we care about is `target/release/libaccount_proof_geyser.dylib`
* Build the solana test validator
```bash
git clone git@github.com:solana-labs/solana.git
git checkout tags/v1.16.15
./cargo build --release --bin solana-test-validator
```
* Update `adapater/solana/config.json`
```json
{
    "libpath": "~/sovereign/adapters/solana/account_proof_geyser/target/release/libaccount_proof_geyser.dylib",
    "bind_address": "127.0.0.1:10000",
    "account_list": ["SysvarS1otHashes111111111111111111111111111"]
}
```
 * Change libpath to point to the full path for `libaccount_proof_geyser.dylib`
 * We can leave `account_list` as `SysvarS1otHashes111111111111111111111111111` for now because this is just an example and WIP
* Run the validator with the geyser config
```bash
~/solana/target/release/solana-test-validator --geyser-plugin-config config.json
```
* Once the validator starts up, you can run the tcp client to fetch the Inclusion proofs for `SysvarS1otHashes111111111111111111111111111` each block
```bash
cd adapters/solana/da_client/
cargo run --release --bin simple_tcp_client
    Finished dev [unoptimized + debuginfo] target(s) in 2.36s
     Running `target/debug/simple_tcp_client`
Proof verification succeeded for slot 36172
Proof verification succeeded for slot 36173
Proof verification succeeded for slot 36174
Proof verification succeeded for slot 36175
```

## Work Remaining
* Rigorous testing for merkle proof generation
* Testing for account update processing
  * Currently, the plugin monitors updates as they arrive, moves them to different hashmaps based on SLot updates for "processed" and "confirmed"
  * This works locally, but production validators fork a lot before confirmation, so we need to test this under load to ensure that we're generating proofs correctly
* Test cases for non inclusion proofs (Inclusion has some tests but Non inclusion doesn't)
* `verify_leaves_against_bankhash` needs to updated for Non inclusion proofs using the adjacency checks `are_adjacent` and `is_first`
  * `is_last` is particularly interesting since proving that a leaf is the first leaf is trivial, but proving last leaf is more complicated since we don't have a commitment to the number of leaves in the tree (i.e. the number of accounts updated)
* The `da_client` PDA needs to be plugged into the `simple_tcp_client` as well as the geyser plugin. This would require non inclusion proofs to work
* Currently, the geyser plugin has a simple tcp server that does only one thing - stream account deltas and their inclusion or non-inclusion proofs. We need to replace this with a more comprehensive GRPC server
