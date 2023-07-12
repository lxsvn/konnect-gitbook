# Compatible With ERC-4337

### What is ERC-4337

Currently, there is a lack of standardized specifications for smart contract wallets. Each wallet implements transaction assembly and gas payment mechanisms according to their own needs, making it difficult for users to migrate between smart contract wallets. Therefore, we need to establish a unified standard to regulate smart contract wallets.

[**ERC-4337**](https://eips.ethereum.org/EIPS/eip-4337) is an Ethereum Improvement Proposal proposed by Vitalik and others. Its purpose is to achieve account abstraction without changing the Ethereum consensus layer.

ERC-4337 defines a set of new specifications. For example, it transforms transactions into `UserOperation` objects. Through `Bundler`, the intentions, signatures, and other data of multiple users can be bundled and passed to the `Entry Point` for batch validation and execution of transactions. The `Paymaster` mechanism is introduced to achieve decentralized transaction fee payment.

The overall protocol of ERC-4337 is very complex. To help everyone understand the entire process, we will divide it into several parts:

#### UserOperation & Bundler

First, let's understand the entire process of sending transactions for ERC-4337.

The contract wallet user sends a transaction, which we refer to as `UserOperation`, signed by the contract wallet user. Although UserOperation is structurally different from EOA-signed transactions, we can simply treat them as equivalent.

In order to submit `UserOperation` to the chain, the contract needs to use a third-party role called `Bundler`. `Bundler` acts as an intermediary between the user and the miner, helping to batch UserOperation into transactions and submit them to the chain.

Here is the detailed process:

1. The user signs a transaction (referred to as UserOperation).
2. The signed transaction is sent to the UserOperation Mempool (also known as the Alternative Mempool in the latest ERC-4337 protocol), which is similar to the Tx Mempool.
3. The Bundler selects transactions from the Mempool and bundles them into a Bundle Transaction.
4. The Bundler then submits this transaction to the chain. (The actual execution process also involves the execution and verification of the Entry Point, which we will continue to discuss below).

#### Entry Point

<figure><img src="../../.gitbook/assets/1.png" alt=""><figcaption></figcaption></figure>

Now, let's briefly understand how transactions are validated and executed after submission to the chain. The Entry Point Smart Contract (referred to as Entry Point) is the unified entry point for validation called by `UserOperation`. `Bundler` will call the handleOps function in the `Entry Point` to validate and execute the transaction submitted to the chain.

Of course, to improve the success rate of on-chain transactions, `Bundler` will first use RPC Call to call simulateValidation() in `Entry Point` locally after bundling the Bundle Transaction. This ensures that the signatures of these `UserOperations` are correct and Gas Fees can be paid normally.

#### Paymaster

Finally, we will provide a detailed explanation of the operations executed by `Bundler` after calling the `handleOps` function in the Entry Point. Additionally, we will introduce a very important role called `Paymaster`. `Paymaster` assists users in paying gas fees and enables them to pay gas fees using any token.

Here's the detailed process:

1. Bundler calls the handleOps function in the Entry Point.
2. The EntryPoint passes UserOperation as a parameter to validateUserOp in the Wallet Contract, verifying all transactions that need to be validated. Only when the validation is successful, the subsequent operations will continue.
3. At this point, the Entry Point first confirms the status of the specified Paymaster in UserOperation, such as whether the Paymaster has enough ETH to pay for the transaction fee.
4. Then, the Entry Point calls validatePaymasterUserOp in the Paymaster Contract to confirm whether the Paymaster is willing to pay the transaction fee. If the Paymaster is willing to pay, the transaction will continue; otherwise, it will fail.
5. Next, the Entry Point calls the Wallet Contract and executes the content specified in UserOperation itself.
6. Then, the Entry Point calls the Post-Op in the Paymaster Contract, which processes custom logic such as direct sponsorship or deduction of ERC20 tokens on behalf of the user.
7. The Entry Point deducts the gas fee required for the transaction from the ETH balance pledged by the Paymaster.
8. Finally, the Entry Point collects the total gas fee required for all transactions and refunds it to the Bundler. The transaction is completed.

\
📌 THERE MAY BE SEVERAL SCENARIOS REGARDING PAYMASTER PAYING FOR THE GAS IN THIS TRANSACTION:

1. Paymaster sponsored the gas fee for this transaction.
2. Paymaster deducted the user's pre-deposited ETH/ERC20 tokens to Paymaster or deducted the ERC20 tokens authorized by the user to Paymaster.
