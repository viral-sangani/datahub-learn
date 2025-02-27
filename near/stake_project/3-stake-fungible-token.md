Plain old vanilla staking is a great way to earn more NEAR, but there are some trade-offs to be aware of. One of the main trade-offs is that the staked NEAR is locked because it is effectively put up as collateral to help secure the NEAR network. We can do better than that by unlocking its value. The [STAKE](https://github.com/oysterpack/oysterpack-near-stake-token) contract unlocks the value stored in the staked NEAR by transferring it into a [fungible token](2-fungible-token.md). You can now have your STAKE and trade it too.

In this tutorial, we'll learn about how staking works on NEAR. We'll see how the STAKE token unlocks value. We'll then apply what we learned from the [fungible token](2-fungible-token.md) tutorial and implement the fungible token core NEP-141 standard in my favorite programming language, Rust. We will take it step-by-step and divide it up into design and coding phases. Finally, we'll take the STAKE contract for a test drive and give you a chance to earn some NEAR. There's a lot to cover, so let's get started ...

## NEAR Staking 101

![](../https://github.com/figment-networks/datahub-learn/raw/master/assets/oysterpack-near-stake-token-basic-staking.png)

The NEAR protocol is a Proof-of-Stake \(PoS\) blockchain. Those who participate in staking to help secure the blockchain network earn staking rewards paid in NEAR tokens. There are two ways to participate:

## As a Validator

[Validators](https://docs.near.org/docs/validator/staking-overview#for-validators) are responsible for running and operating the validator nodes that are actively producing blocks and must come to a stake-weighted consensus about the valid state of the chain. On NEAR, the number of validator nodes per shard is limited \(currently 100 per shard\). Anyone can submit a proposal to become a validator, but validator seats are auctioned off to the highest bidders, i.e., those who stake the most NEAR.

## As a Delegator

Anyone can earn staking rewards by delegating their NEAR tokens to a validator through a [staking pool](https://github.com/near/core-contracts/tree/master/staking-pool) contract. Here's how it works:

1. Each deployed staking pool contract instance is owned by a single validator. 
2. Users deposit NEAR into the staking pool contract. Recall that validator seats are auctioned off to the highest bidder. Delegators pool their NEAR tokens with validators to help them win auction bids for validator seats. Staking rewards earned by the validator are shared with the delegator minus validator fees. 
3. While delegated NEAR is deposited in the staking pool, it is effectively locked. While being staked, the delegated NEAR is owned by the staking pool contract \(which is owned by the validator\). Effectively, delegators are lending their NEAR to validators through the staking pool contract. Delegators' return on investment is their share of staking rewards - assuming the validator acquires a seat and does his job. 
4. When delegators choose to withdraw their NEAR they must first unstake the NEAR tokens. The unstaked NEAR will remain locked within the staking pool for 4 epoch periods \(2 days\) before being eligible for withdrawal from the staking pool contract.

## ATTENTION: How Unstaking Affects Withdrawals

> One thing users need to be aware of and careful about is how the unstaking lock period functions in the current version of the staking pool contract. Each time the user submits a request to unstake NEAR it resets the lock period to 4 epochs \(2 days\). For example, if a user unstaked 1000 NEAR in epoch \(1\), then the 1000 NEAR will be available for withdrawal in epoch \(4\). What happens if the user submits another request to unstake 1 NEAR on epoch \(3\)? When you try to withdraw the 1000 NEAR in epoch \(4\), it is still locked because the lock period has been reset and extended for all unstaked NEAR. The 1001 NEAR will then be available for withdrawal in epoch \(7\).


For more information on how staking works on NEAR see:

* [Economics in a Sharded Blockchain - Validators section](https://near.org/papers/economics-in-sharded-blockchain/#validators)
* [Is NEAR a delegated proof of stake network](https://docs.near.org/docs/faq/economics_faq#is-near-a-delegated-proof-of-stake-network)
* [Staking Orientation](https://docs.near.org/docs/validator/staking-overview)

## You Can Have Your STAKE and Trade It Too

![](../https://github.com/figment-networks/datahub-learn/raw/master/assets/oysterpack-near-stake-token-STAKE-FT.png)

When you deposit NEAR into the STAKE token contract, it will delegate the NEAR to the staking pool for you. In return, you are issued STAKE tokens that grow in value via staking rewards issued through the staking pool contract. STAKE tokens unlock the value of the staked NEAR that is locked up by the staking pool contract by enabling the user to transfer the staked NEAR value to other accounts. The STAKE token contract adds even more value beyond letting you use staked NEAR as fungible tokens. However, we'll save that for future tutorials. For now, we'll stay focused on how the STAKE token implements [NEP-141](2-fungible-token.md).

## Why Rust for Smart Contracts

Rust's website sums it up in 3 words: performance, reliability, and productivity. For me it's a no-brainer. Over the past 20+ years, I have used many programming languages, and [Rust](https://www.rust-lang.org/) is the clear winner for me. For five years running, Rust has also been voted by developers as the [most loved language](https://insights.stackoverflow.com/survey/2020#technology-most-loved-dreaded-and-wanted-languages-loved) on Stackoverflow for good reason. It is becoming the preferred language of choice for the crypto space \(NEAR is written in Rust\) because it is a perfect fit to build highly performant, reliable, and secure blockchain software. On NEAR, for any serious or more complex smart contracts, especially within the DeFi space, I strongly encourage and recommend Rust to you.

## NEAR Rust SDK

To develop NEAR smart contracts, you will need to learn to use the [NEAR Rust SDK](https://crates.io/crates/near-sdk). I link to the latest and greatest tagged version on Github because:

* It provides a few features that are not yet released via crates.io to reduce boilerplate
  * `#[private]` macro for callbacks
  * `PanicOnDefault` used to derive `Default` implementation that panics. This is a helpful macro in case the contract is required to be initialized with either `#[init]` or `#[init_once]`.
  * Better unit testing support
* For simulation testing support

Below is how to link the project directly to a tagged version of the NEAR Rust SDK on Github: **Cargo.toml**

```text
[dependencies]
near-sdk = { git = "https://github.com/near/near-sdk-rs",  tag = "2.4.0" }

[dev-dependencies]
near-sdk-sim = { git = "https://github.com/near/near-sdk-rs",  tag = "2.4.0" }
```

That's pretty cool because if you are eager to start using new features then you don't need to wait until the crate is published to [https://crates.io](https://crates.io).

* **NOTE**: to view the corresponding Rust docs, you will need to clone the above git projects and generate the Rust docs locally.

## Show Me the Design

To keep the code clean we will first design the interfaces separate from the implementation. The contract API will be defined explicitly via an interface. In Rust, interfaces are called [traits](https://doc.rust-lang.org/book/ch10-02-traits.html). The design approach will be to leverage Rust strongly typed system to model the domain. The [NEP-141](2-fungible-token.md) standard defined the API using the lowest common denominator to keep the API programming language-neutral as much as possible. In doing so, we lost type safety. For example, numeric amounts were specified as `string` types. With Rust, we get type safety back. We will be working with a typed domain model that makes the code clear and precise.

![](../https://github.com/figment-networks/datahub-learn/raw/master/assets/oysterpack-near-stake-token-FT-NEP-141.png)

* see [Rust code](https://github.com/oysterpack/oysterpack-near-stake-token/blob/main/contract/src/interface/fungible_token.rs) on Github

**StakeTokenContract** implements the **FungibleToken** and **ResolveTransferCall** traits. It depends on the **TransferReceiver** interface for cross-contract calls.

**FungibleToken** trait

* Specifies the core fungible token API
* Instead of using native String types, I use specific domain type wrappers for `TokenAmount`, `Memo`, and `TransferCallMessage`
  * This leaves absolutely zero ambiguity in the code and enables the domain model to be encoded into the type system. You may think this is overkill for something so simple, but my advice is to never take shortcuts. If you are going to do something, then do it right in the first place. In addition, the beauty of Rust's zero-cost abstractions, is that we can leverage the type system for zero runtime costs \(if done right\).

**TransferReceiver** trait

* Represents the contract API required by the transfer receiver contract

**ResolveTransferCall** trait

* Specifies the **private** callback interface used as part of the transfer call workflow
* The keyword here to notice is **private** - it means that even though the function is exposed on the contract, only the contract itself is allowed to call the function. If any other account tries to call the private function, then it should fail.
* **NOTE**: the callback function signature is not explicitly defined by the FT standard \(NEP-141\). I am presenting to you my implementation, but you may choose to name your callback whatever you want

## Show Me the Code

You can refer to the full [source code](https://github.com/oysterpack/oysterpack-near-stake-token/blob/main/contract/src/contract/fungible_token.rs) on Github. I will be reviewing the most important parts of the code below.

We start by implementing the core fungible token interface on the contract:

```rust
#[near_bindgen]
impl FungibleToken for StakeTokenContract {
  // TODO
}
```

* [near\_bindgen](https://crates.io/crates/near-bindgen) generates the smart contract compatible with the NEAR blockchain at the WASM level
* The above reads as `StakeTokenContract` implements the `FungibleToken`  interface

Now we'll walk through each of the core contract API functions. For those of you who are Rust gurus or seasoned developers, please bear with me ... We will be pointing out Rust idioms and coding best practices that may seem obvious. Because we will be seeing actual Rust code for the first time, Rust idioms and coding practices will be described in more detail so that you can become familiar with my coding style. In future tutorials, I will assume this knowledge.

```rust
#[payable]
fn ft_transfer(
    &mut self,
    receiver_id: ValidAccountId,
    amount: TokenAmount,
    _memo: Option<Memo>,
) {
    assert_yocto_near_attached();
    assert_token_amount_not_zero(&amount);

    let stake_amount: YoctoStake = amount.value().into();

    let mut sender = self.predecessor_registered_account();
    self.claim_receipt_funds(&mut sender);
    sender.apply_stake_debit(stake_amount);
    // apply the 1 yoctoNEAR that was attached to the sender account's NEAR balance
    sender.apply_near_credit(1.into());

    let mut receiver = self.registered_account(receiver_id.as_ref());
    receiver.apply_stake_credit(stake_amount);

    self.save_registered_account(&sender);
    self.save_registered_account(&receiver);
}
```

* `#[payable]` marks the function to allow callers to attach NEAR to the function call. Recall that according to the specification, callers must attach exactly 1 yoctoNEAR to the function call as a security measure
* `&mut self` tells the rust compiler that the function will modify contract state
* [ValidAccountId](https://docs.rs/near-sdk/2.0.1/near_sdk/json_types/struct.ValidAccountId.html) comes from NEAR Rust SDK. It eliminates boilerplate code to validate the NEAR account ID. If the account ID is not a valid NEAR account ID, then the function call will fail fast
* `_memo` - the STAKE contract has no use for memo, and thus tells the rust compiler that it will not be used by using a naming convention, i.e., by prefixing the name with an `_`. The rust compiler is very strict and disciplined. By default, it will emit warnings for any sign of something possibly wrong with the code.
* The first thing the code does is perform some checks:
  * It checks to make sure exactly 1 yoctoNEAR is attached
  * It checks that the transfer amount is not zero
  * By this point in the code the `receiver_id` has already been validated by NEAR SDK
* Let's keep the contract function code as clean and readable as possible. The goal is to be able to read the code and easily understand the business logic. Implementation details or boilerplate should be separated out into other functions. If you come back to the code in 6 months and can't understand it or is hard to follow, then it's time to refactor and clean it up.
* Converting the token amount to `YoctoStake` is specific to the STAKE token business logic. The code is expecting the transfer amount to be specified in yocto scale - yoctoSTAKE is the smallest unit for the STAKE token, just like yoctoNEAR is the smallest unit for NEAR. That's a bit off-topic ... we'll revisit this in future tutorials
* NEP-141 requires that the accounts involved in the transfer must both be registered. The `predecessor_registered_account()`and `registered_account()` helper functions will lookup the accounts and panic if the account is not registered
* `claim_receipt_funds()` is specific to STAKE business logic and we'll skip this for now
* `sender.apply_near_credit(1.into())` - remember the sender was required to attach 1 yoctoNEAR to the function call. How the attached deposit is handled is not defined in the standard. In the STAKE contract, every registered account has a NEAR balance. Thus, the contract will credit the yoctoNEAR to the sender account \(because 1 yoctoNEAR is not zero\).
* The transfer amount is debited from the sender and then credited to the receiver
* In order to commit the transfer to the blockchain, we must remember to persist the state change to storage. Both the sender and receiver accounts are saved to storage.

{% hint style="info" %}
# Best Practices

1. You should always use [ValidAccountId](https://docs.rs/near-sdk/2.0.1/near_sdk/json_types/struct.ValidAccountId.html) as the contract function argument type
2. Contract API function should be easy to read to understand the business logic
3. Remember to commit contract state changes to storage
{% endhint %}

```rust
 #[payable]
fn ft_transfer_call(
    &mut self,
    receiver_id: ValidAccountId,
    amount: TokenAmount,
    msg: TransferCallMessage,
    _memo: Option<Memo>,
) -> Promise {
    self.ft_transfer(receiver_id.clone(), amount.clone(), _memo);

    ext_transfer_receiver::ft_on_transfer(
        env::predecessor_account_id(),
        amount.clone(),
        msg,
        receiver_id.as_ref(),
        NO_DEPOSIT.value(),
        self.ft_on_transfer_gas(),
    )
    .then(ext_resolve_transfer_call::ft_resolve_transfer_call(
        env::predecessor_account_id(),
        receiver_id.as_ref().to_string(),
        amount,
        &env::current_account_id(),
        NO_DEPOSIT.value(),
        self.resolve_transfer_gas(),
    ))
}
```

Transfer call is a little more interesting because it involves cross contract calls. It showcases how NEAR differs and stands out from other blockchains thanks to its sharded architecture.

![](../https://github.com/figment-networks/datahub-learn/raw/master/assets/oysterpack-near-stake-token-transfer-call.png)

* Transfer call workflow will first transfer the tokens - it simply delegates to `ft_transfer()` as we saw above
* Then it composes the async transfer call workflow using the [Promise](https://docs.rs/near-sdk/2.0.1/near_sdk/struct.Promise.html) abstraction provided by the NEAR Rust SDK
  * The Promise that is returned is not executed as part of the current block. The NEAR runtime will schedule the Promise to run async and kickoff the workflow in the next block. It will be executed on the shard that hosts the receiver account. Once the `ft_on_transfer` call completes on the receiver account, then the NEAR runtime will capture its output data and provide it as input data for the `ft_resolve_transfer_call`. The callback will be scheduled in the next block to run on the shard that hosts the STAKE token contract.
  * With cross contract calls gas considerations need to be taken into account. The FT standard states that `ft_transfer_call`must pass along all unused gas to the receiver contract. Technically speaking that is not possible. We can approximately pass along the unused gas. Getting gas right in cross contract calls requires experimentation. The approach I use is to measure the gas consumption on the callback by temporarily relaxing the private constraint. This enables me to invoke the function directly and measure the gas consumption. The STAKE contract provides an operator interface to enable the operator to configure the gas usage for cross contract calls. This is beyond the scope of this tutorial, but worth mentioning. Checkout the `ft_on_transfer_gas()` and `resolve_transfer_gas()` methods in the source code for details.
* The code uses what's called the high-level cross contract pattern provided by NEAR Rust SDK. It works as follows:

The remote function calls are declared as rust traits and annotated with the `#[ext_contract]` attribute. This attribute will be used to generate the low-level code to invoke the remote call on the external contract. For each external contract interface that is annotated, a rust module is generated containing functions that map to the functions defined on the trait. I explicitly specify the module name in the attribute, e.g., `ext_transfer_receiver` explicitly specifies the module name. The name is optional, and a default name will be generated based on the trait name if not specified - but I prefer to be specific.

```rust
#[ext_contract(ext_transfer_receiver)]
pub trait ExtTransferReceiver {
    fn ft_on_transfer(
        &mut self,
        sender_id: AccountId,
        amount: TokenAmount,
        msg: TransferCallMessage,
    ) -> PromiseOrValue<TokenAmount>;
}

#[ext_contract(ext_resolve_transfer_call)]
pub trait ExtResolveTransferCall {
    fn ft_resolve_transfer_call(
        &mut self,
        sender_id: AccountId,
        receiver_id: AccountId,
        amount: TokenAmount,
    ) -> PromiseOrValue<TokenAmount>;
}
```

Each module function that is generated for the external contract appends arguments required to make the remote contract function call that is required by the NEAR protocol:

* Contract account ID
* How much NEAR to attach to the function call
* How prepaid gas to supply to the function call

> Something to be aware of is that the high-level cross contract approach works well for simple cross contract calls - as in this case. However, you may need to reach down to use the lower level cross contract approach because you require more robust error handling, or your use case requires batched transactions, etc. We'll explore this topic in future tutorials.

## How Promises work on NEAR

On NEAR, think of [promises](https://docs.rs/near-sdk/2.0.1/near_sdk/struct.Promise.html) as a way to compose **actions** to run asynchronously on remote contracts. I use the term actions because promises support more than just calling remote functions. You can perform other actions, such as creating and deleting accounts, adding keys, deploying contracts, and of course transferring NEAR - see the [docs](https://docs.rs/near-sdk/2.0.1/near_sdk/struct.Promise.html) for details. For now, we'll focus the discussion on using promises for remote function calls.

Remote function calls are always scheduled to run async after the current function is committed to the blockchain. This means contract state will be persisted to blockchain storage before the remote function is executed in a future block - most likely in the next block. If the contract requires to handle the remote contract function call results or perform error handling, then the contract must schedule a callback that is triggered when the remote function call completes. We used [Promise::then](https://docs.rs/near-sdk/2.0.1/near_sdk/struct.Promise.html#method.then) to connect the two function calls and compose them into the transfer call workflow.

## NEAR Promises Considerations:

1. Promises always run async.
2. In order to handle promise results, the contract must schedule a callback.
3. There are no global transactions that spans across contracts. Think of each contract function call always being executed in its own separate transaction. If contract state needs to be rolled back because a downstream promise failed, then the contract is responsible to rollback the contract state in the form of a compensating transaction.

Let's bring it home:

```rust
#[near_bindgen]
impl ResolveTransferCall for StakeTokenContract {
    #[private]
    fn ft_resolve_transfer_call(
        &mut self,
        sender_id: ValidAccountId,
        receiver_id: ValidAccountId,
        amount: TokenAmount,
    ) -> PromiseOrValue<TokenAmount> {
        let unused_amount = self.transfer_call_receiver_unused_amount(amount);

        let refund_amount = if unused_amount.value() > 0 {
            log!("unused amount: {}", unused_amount);
            let mut sender = self.registered_account(sender_id.as_ref());
            let mut receiver = self.registered_account(receiver_id.as_ref());
            match receiver.stake.as_mut() {
                Some(balance) => {
                    let refund_amount = if balance.amount().value() < unused_amount.value() {
                        log!("ERROR: partial amount will be refunded because receiver STAKE balance is insufficient");
                        balance.amount()
                    } else {
                        unused_amount.value().into()
                    };
                    receiver.apply_stake_debit(refund_amount);
                    sender.apply_stake_credit(refund_amount);

                    self.save_registered_account(&receiver);
                    self.save_registered_account(&sender);
                    log!("sender refunded: {}", refund_amount.value());
                    refund_amount.value().into()
                }
                None => {
                    log!("ERROR: refund is not possible because receiver STAKE balance is zero");
                    0.into()
                }
            }
        } else {
            unused_amount
        };

        PromiseOrValue::Value(refund_amount)
    }
}

impl StakeTokenContract {
  /// the unused amount is retrieved from the `TransferReceiver::ft_on_transfer` promise result
  fn transfer_call_receiver_unused_amount(&self, transfer_amount: TokenAmount) -> TokenAmount {
    let unused_amount: TokenAmount = match self.promise_result(0) {
      PromiseResult::Successful(result) => {
        serde_json::from_slice(&result).expect("unused token amount")
      }
      _ => {
        log!(
          "ERROR: transfer call failed on receiver contract - full transfer amount will be refunded"
        );
        transfer_amount.clone()
      }
    };

    if unused_amount.value() > transfer_amount.value() {
      log!(
        "WARNING: unused_amount({}) > amount({}) - full transfer amount will be refunded",
        unused_amount,
        transfer_amount
      );
      transfer_amount
    } else {
      unused_amount
    }
  }
}
```

The business logic is pretty straight forward. The callback's main purpose is two-fold

1. It refunds any unused amount back from the receiver account to the sender account.
2. If the function call failed on the receiver contract, then the callback will attempt to refund the full amount.

Certain business rules and checks are executed to guard against receiver contracts that violate the contract. This ties back to our earlier discussion on promises. Before the receiver contract is invoked, the transfer has already been committed to contract storage on the blockchain. By the time the callback runs, the contract state for the STAKE contract and for the receiver contract has already been committed to the blockchain. Thus, the callback must be coded defensively to handle errors or contracts that violate the transfer call protocol.

The NEAR Rust SDK currently has no high-level support for handling promise failures in cross contract calls - but it's on the NEAR Rust SDK roadmap. The `transfer_call_receiver_unused_amount` function shows how to use the low level NEAR SDK API to handle the promise result, which requires us to manually deserialize the result: `serde_json::from_slice(&result).expect("unused token amount")`

# Show Me the Demo: Earn Some NEAR

As a bonus, you can earn some NEAR by taking the STAKE token contract for a test drive on testnet and running through the demo below using the [NEAR CLI](https://github.com/near/near-cli). I have deployed the STAKE contract to `stake-demo.oysterpack.testnet` on testnet for the demo. To earn NEAR rewards for exercising the demo, you will need to submit the NEAR requests through [DataHub](https://datahub.figment.io/) using your DataHub access key. If you have earned NEAR on previous NEAR tutorials, then you should already be set. Otherwise, follow the instructions in the following link on [how to obtain your DataHub access key](https://learn.figment.io/network-documentation/near/tutorials/intro-pathway-write-and-deploy-your-first-near-smart-contract/1.-connecting-to-a-near-node-using-datahub#configure-environment). We will use the NEAR CLI to submit the transactions. Plugin your DataHub API Key and NEAR account at the top, and then you should be all set to go.

```bash
export DATAHUB_APIKEY=<DATAHUB_APIKEY>
export NEAR_ACCOUNT=<YOUR-NEAR-ACCOUNT.testnet>

export CONTRACT=stake-demo.oysterpack.testnet
export NEAR_NODE_URL=https://near-testnet--rpc.datahub.figment.io/apikey/$DATAHUB_APIKEY
export NEAR_ENV=testnet

# register account
near call $CONTRACT register_account --node_url $NEAR_NODE_URL --accountId $NEAR_ACCOUNT --amount 1

# deposit and stake some NEAR to get some STAKE tokens
near call $CONTRACT deposit_and_stake --node_url $NEAR_NODE_URL --accountId $NEAR_ACCOUNT --amount 1 --gas 200000000000000

# check balance
near view $CONTRACT ft_balance_of --node_url $NEAR_NODE_URL --args "{\"account_id\":\"$NEAR_ACCOUNT\"}" 

# check total supply
near view $CONTRACT ft_total_supply --node_url $NEAR_NODE_URL

# check balance for receiver contract - before transfer call
near view $CONTRACT ft_balance_of --node_url $NEAR_NODE_URL --args '{"account_id":"dev-1611907846758-1343432"}'

# transfer STAKE via a simple transfer
near call $CONTRACT ft_transfer --node_url $NEAR_NODE_URL --accountId $NEAR_ACCOUNT  --args '{"receiver_id":"dev-1611907846758-1343432", "amount":"10000000"}' --amount 0.000000000000000000000001

# check balance for transfer receiver contract - before transfer call
near view $CONTRACT ft_balance_of --node_url $NEAR_NODE_URL --args "{\"account_id\":\"dev-1611907846758-1343432\"}"

# transfer STAKE via a transfer call to another contract
near call $CONTRACT ft_transfer_call --node_url $NEAR_NODE_URL --accountId $NEAR_ACCOUNT  --args '{"receiver_id":"dev-1611907846758-1343432", "amount":"1000000", "memo":"merry christmas", "msg":"{\"Accept\":{\"refund_percent\":50}}"}' --amount 0.000000000000000000000001

# check balance for transfer receiver contract - after transfer call
near view $CONTRACT ft_balance_of --node_url $NEAR_NODE_URL --args "{\"account_id\":\"dev-1611907846758-1343432\"}"
```

* **NOTE**: in case you wondering what **dev-1611907846758-1343432** is, it's a mock contract that implements the **TransferReceiver** contract interface. The `near dev-deploy` CLI command was used to deploy the contract, which automatically creates the account and auto-generated the contract account ID. For those that are even more curious about the mock contract, the code is located within the same STAKE project: [ft-transfer-receiver-mock](https://github.com/oysterpack/oysterpack-near-stake-token/tree/main/contract/ft-transfer-receiver-mock).

## Conclusion

That was longer than expected, but time flies by when you are having fun. We learned about staking on NEAR and how you can earn staking rewards as a delegator. I went over, at a high level, how the STAKE contract unlocks value in your staked NEAR by providing you with fungible tokens for your staked NEAR. We then implemented the Fungible Token Core Standard \(NEP-141\) for the STAKE contract. We did some design and coding. Along the way, we also got a little taste of how cross contract calls and promises work on NEAR using the NEAR Rust SDK. Finally, you got a chance to earn some NEAR while learning with us. I invite you to join the Figment and NEAR communities and embark on our common mission to take back the Internet together.

# Next Steps

![](../https://github.com/figment-networks/datahub-learn/raw/master/assets/oysterpack-testing-code-meme.jpg)

Just kidding folks ... I didn't forget about testing. Testing was intentionally left out of this tutorial because it is a huge topic in itself that deserves full attention. I also didn't want to turn this tutorial into a novel 😄. In the next tutorial, we'll be following up on how to test our smart contracts using the STAKE fungible token as an example.

The fun's just begun!
