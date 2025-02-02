# Ethereum Vault Connector (EVC)

Mick de Graaf, Kasper Pawlowski, Dariusz Glowinski, Michael Bentley, Doug Hoyte

<!-- TOC FOLLOWS -->
<!-- START OF TOC -->

* [Introduction](#introduction)
* [Controller](#controller)
* [Account Status Checks](#account-status-checks)
    * [Collateral Validity](#collateral-validity)
    * [Execution Flow](#execution-flow)
    * [Single Controller](#single-controller)
    * [Require Immediate](#require-immediate)
    * [Forgiveness](#forgiveness)
* [Vault Status Checks](#vault-status-checks)
* [Execution](#execution)
    * [Checks-deferrable Call](#checks-deferrable-call)
    * [call](#call)
    * [batch](#batch)
    * [controlCollateral](#controlcollateral)
    * [permit](#permit)
        * [Nonce Namespaces](#nonce-namespaces)
    * [Authorisation](#authorisation)
        * [Sub-Accounts](#sub-accounts)
        * [Operators](#operators)
    * [Execution Contexts](#execution-contexts)
        * [Nested Execution Contexts](#nested-execution-contexts)
        * [checksInProgress](#checksinprogress)
        * [controlCollateralInProgress](#controlcollateralinprogress)
        * [Extra Information](#extra-information)
    * [Simulations](#simulations)
* [Transient Storage](#transient-storage)
* [Security Considerations](#security-considerations)
    * [Authentication by Vaults](#authentication-by-vaults)
    * [EVC Contract Privileges](#evc-contract-privileges)
    * [Read-only Re-entrancy](#read-only-re-entrancy)

<!-- END OF TOC -->

## Introduction

The Ethereum Vault Connector (EVC) is a foundational layer designed to facilitate the core functionality required for a lending market. It serves as a base building block for various protocols, providing a robust and flexible framework for developers to build upon. The EVC primarily mediates between vaults, contracts that implement the [ERC-4626](https://ethereum.org/en/developers/docs/standards/tokens/erc-4626/) interface and contain additional logic for interfacing with other vaults. The EVC not only provides a common base ecosystem but also reduces complexity in the core lending/borrowing contracts, allowing them to focus on their differentiating factors.

To illustrate the process of vault mediation, let's consider a straightforward example. When a user wishes to borrow, they must link their accounts and collateral vaults to the borrowed-from vault via the EVC. The liability vault, also known as the "controller", is then consulted whenever a user wants to perform an action potentially impacting account's solvency, such as withdrawing collateral. The EVC is responsible for calling the controller which determines whether the action is allowed or if it should be blocked to prevent account insolvency.

In addition to vault mediation, the EVC contains the functionality required to build flexible products, both for EOAs and smart contracts. Here are some of the benefits of building on the EVC:

* Network effect thanks to unified liquidity and interoperability: Participating protocols can accept deposits in other vaults as collateral suitable for their vaults, providing convenience for users who no longer need to move their collateral assets from one protocol to another.
* Flexibility in asset properties: The EVC does not enforce specific properties about the assets used as collateral or liabilities, allowing users to create vaults backed by irregular asset classes, such as NFTs, Real World Assets (RWAs), uncollateralised IOUs, or synthetics.
* Standardized approach to account liquidity checks and vault global constraints enforcement: The EVC allows for deferral of the liquidity checks and vault status checks, preventing transient violations from causing a failure. The EVC exposes an interface which abstracts management of the checks away from the vault.
* Batching: Multiple operations affecting multiple vaults and external smart contracts can be performed within a single batch operation. This is more convenient for UI users, more gas efficient, and allows deferring liquidity checks until the end of the batch.
* Sub-accounts: A feature that allows users to create multiple isolated positions within their single owner account, and easily rebalance collateral/liabilities between them without the need for approvals and without requiring any special logic to be implemented by the vaults.
* Operators: Users can attach external contracts to act on behalf of a sub-account. This is a generalisation of the token approval system and will unlock powerful functionality, even for EOAs. For example, intents support, stop-loss/take-profit/trailing-stop/etc modifiers can be added to positions, or entire layered position managers can be built on top.
* Gasless transactions (meta-transactions): They can be supported out of the box for both EOAs and contract wallets.
* Simulations: The EVC exposes the optimal interface for simulating the effects of a set of operations and pre-visualising their effects in a UI for all EVC users.
* A common language for liquidations: Vaults can implement a core liquidation interface that will allow them to rely on an existing network of liquidators to keep their depositors safe.

As already mentioned, the EVC not only provides the above features to a common base ecosystem, but also reduces complexity in the core lending/borrowing contracts, allowing them to focus on their differentiating factors such as pricing and risk management.


## Controller

The primary task of the EVC is to maintain a user's voluntary associations with vaults. Typically, a user will deposit funds into one or more collateral vaults, and call `enableCollateral` for each that is intended to be used as a collateral. This adds the vault to the given account's collateral set. Users should obviously be careful which vaults they deposit to, since a malicious vault could refuse to return their funds.

After simply depositing and enabling collaterals, users are not obligated or bound in any way by the EVC, and could freely withdraw funds from the vaults and/or call `disableCollateral` to remove the vaults from the account's collateral set.

However, suppose a user wants to take out a borrow from a separate vault. In this case, the user must call `enableController` to add this vault to the account's controller set. This is a significant action because the user is now entirely submitting the account to the rules encoded in the controller vault's code. All the funds in all the collateral vaults are now indirectly under control of the controller vault. In particular, if a user attempts to withdraw collateral or `disableCollateral` to remove a vault from the collateral set, the controller could cause the transaction to fail. Moreover, the controller can allow the collateral to be seized in order to repay the debt, using [`controlCollateral`](#controlcollateral).

* When requested to perform an action such as borrow, a liability vault must call into the EVC's `isControllerEnabled` function to ensure that the account has in fact enabled the vault as a controller.
* Only the controller itself can call `disableController` on the EVC. This should typically happen upon an account repaying its debt in full. Vaults must be coded carefully to not have edge cases such as unrepayable dust, otherwise accounts could become permanently associated with a controller.
* The order of an account's collateral set can be changed with `reorderCollaterals`. Because some controller vaults will loop over an account's collateral in sequence and return early if sufficient value is found, this can save considerable gas. This feature was inspired by Gearbox's `collateralHints`.

Given that enabling a controller subjects the specified account to the rules encoded in the controller's code, users must only enable trusted, audited controllers. If the controller is malicious or incorrectly coded, it may result in the loss of the user's funds or even render the account unusable.

## Account Status Checks

Account status checks are implemented by vaults to enforce account solvency. Vaults must expose an external `checkAccountStatus` function that will receive an account's address and this account's list of enabled collateral vaults. If the account has not borrowed anything from this vault then the function should return a special magic success value (the function selector for the `checkAccountStatus` method). Otherwise, the vault should evaluate application-specific logic to determine whether or not the account is in an acceptable state. If so, it should return the special magic success value, otherwise throw an exception.

### Collateral Validity

Within the `checkAccountStatus` callback, vaults should inspect the provided list of collaterals and determine whether or not they are acceptable. Vaults can limit themselves to a small set of collaterals, or can be more general-purpose and allow borrowing using any assets they can get prices for. Alternately, a vault could always fail, if it is only intended to be a collateral vault.

Vaults have the freedom to price all the assets according to their preference (both liability and accepted collaterals), without depending on potentially untrustworthy oracles.

While it might be tempting for the controller to allow a broad range of collateral vaults to encourage borrowing, the controller vault creators must exercise caution when deciding which vaults to accept as collateral. A malicious or incorrectly coded vault could, among other things, misrepresent the amount of assets it holds, reject liquidations when a user is in violation, or fail to require account status checks when necessary. Therefore, vaults should limit allowed collaterals to a set of audited addresses known to be reliable, or verify the addresses in a registry or factory contract to ensure they were created by trustworthy, audited contracts.

### Execution Flow

Although the vaults themselves implement `checkAccountStatus`, there is no need for them to invoke this function directly. It will be called by the EVC when necessary. Instead, after performing any operation that could affect an account's liquidity, a vault should invoke `requireAccountStatusCheck` on the EVC to schedule a future callback. Additionally, operations that can affect the liquidity of a *separate* account will need their own `requireAccountStatusCheck` calls.

Upon a `requireAccountStatusCheck` call, the EVC will determine whether the current execution context is in a checks-deferrable call and if so, it will defer checking the status for this account until the end of the execution context. Otherwise, the account status check will be performed immediately.

There is a subtle complication that vault implementations should consider if they use re-entrancy guards (which is recommended). When a vault is invoked *without* account status checks being deferred (ie, directly, not via the EVC), if it calls `requireAccountStatusCheck` on the EVC, the EVC will immediately call back into the vault's `checkAccountStatus` function. A normal re-entrancy guard would fail upon re-entering at this point. To avoid this, vaults may wish to use the [call](#call) EVC function.

### Single Controller

At the time of the account status check, an account can have at most one controller. This is how single-liability-per-account is enforced. Multiple controllers are disallowed because it is unlikely two independent controllers would be able to behave consistently in the presence of "shared" accounts. If this is ever required, a multi-controller controller could be created with the desired sharing logic.

Although having more than one controller is disallowed when the account status check is performed, multiple controllers can be transiently attached while these checks are deferred. As long as all or all but one controllers release themselves during the execution of the [checks-deferrable call](#checks-deferrable-call), the account status check will succeed.

### Require Immediate

Inside a checks-deferrable call, account status checks are deferred and only checked at the end. However, in some cases it is desirable to immediately check an account's status using `requireAccountStatusCheckNow`. If the specified account has a controller, this vault's `checkAccountStatus` is immediately invoked to determine if the account has a valid status.

If valid, the account is removed from the account status deferral set (if present), so it will not be checked again at the end of the checks-deferrable call. However, any future operations in the same checks-deferrable call that require a (non-immediate) status check will re-add it to the set, and the status will be checked at the end of the call.

Some use-cases are:

* If a vault wants to prevent providing flash loans for whatever reason, it can require the account be healthy immediately following a borrow.
* If a batch creator believes there is a possibility that an account will be unhealthy after an operation (perhaps because of changes to chain state between transaction creation and inclusion times), it may make sense to check this account's health immediately, before performing other operations. This will save gas by failing the transaction early.

### Forgiveness

If a controller wants to waive the liquidity check for an account it is controlling, it can "forgive" an account. This removes it from the set of accounts that will be checked at the end of the call. Controllers can only forgive accounts that they are the sole controller of.

Needless to say, this functionality should be used with care. It should only be necessary in certain advanced liquidation flows where the collateral is seized from an unhealthy account but the seizure of funds is still not enough to bring the account to a sufficiently healthy level to pass the account status check.

When doing so, it is important that vaults verify that no *other* collaterals have unexpectedly been withdrawn during the seizure, in the event that a vault makes any unexpected external calls in its transfer/withdraw/etc method.


## Vault Status Checks

Some vaults may have constraints that should be enforced globally. For example, supply and/or borrow caps that restrict the maximum amount of assets that can be supplied or borrowed, as a risk minimisation.

It does not necessarily make sense to enforce these checks when checking account status. First of all, if many accounts are affected within a batch, checking these global constraints each time would be redundant.

Secondly, some types of checks require an initial snapshot of the vault state before any operations have been performed. In the case of a borrow cap, it could be that the borrow cap has been exceeded for some reason (perhaps due to a price movement, or the borrow cap itself was reduced). The vault would still want to permit repaying debts, even if the repay was insufficient to bring the total borrows below the borrow cap.

Vaults must expose an external `checkVaultStatus` function. The vault should evaluate application-specific logic to determine whether or not the vault is in an acceptable state. If so, it should return a special magic success value (the function selector for the `checkVaultStatus` method), otherwise throw an exception.

Although the vaults themselves implement `checkVaultStatus`, there is no need for them to invoke this function directly. It will be called by the EVC when necessary. Instead, after performing any operation that could affect the vault's status, a vault should invoke `requireVaultStatusCheck` on the EVC to schedule a future callback.

Upon receiving a `requireVaultStatusCheck` call, the EVC will determine whether the current execution context defers the checks and if so, it will defer checking the status for this vault until the end of the execution context. Otherwise, the vault status check will be performed immediately.

In order to evaluate the vault status, `checkVaultStatus` may need access to a snapshot of the initial vault state. If so, the recommended pattern as implemented in the reference vaults is as follows:

* Before performing any actions, each operation that requires a vault status check should first make an appropriate snapshot and store the data in transient storage (if a snapshot has not already been made)
* The operations should be performed
* The vault should then call `requireVaultStatusCheck`
* When the `checkVaultStatus` callback is invoked, it should evaluate the vault status by unpacking the snapshot data stored in transient storage and compare it against the current state of the vault, and return a special magic success value, or revert if there is a violation.

As with the account status check, there is a subtle complication that vault implementations should consider if they use re-entrancy guards (which is recommended). When a vault is invoked *without* vault status checks being deferred (ie, directly, not via the EVC), if it calls `requireVaultStatusCheck` on the EVC, the EVC will immediately call back into the vault's `checkVaultStatus` function. A normal re-entrancy guard would fail upon re-entering at this point. To avoid this, vaults may wish to use the [call](#call) EVC function.


## Execution

### Checks-deferrable Call

The EVC exposes multiple functions, each with its own characteristics, that allow for the [Account Status Checks](#account-status-checks) and [Vault Status Checks](#vault-status-checks) to be deferred. [call](#call), [batch](#batch) and [controlCollateral](#controlcollateral) so called checks-deferrable call functions, can be nested and allow for the checks to be deferred until the end of the execution of the outermost function call.

### call

The `call` function on the EVC allows users to invoke functions on vaults and other target smart contracts, including the EVC itself. Unless the `msg.sender` is the same as the `onBehalfOfAccount`, users *must* go through this function rather than calling the vaults directly. This is because vaults themselves don't understand sub-accounts or operators, and defer their authorisation logic to the EVC (see the [Authentication By Vaults](#authentication-by-vaults) section).

`call` also allows users to invoke arbitrary contracts, with arbitrary calldata. These other contracts will see the EVC as `msg.sender`. For this reason, it is critical that the EVC itself never be given any special privileges, or hold any token or native currency balances (except for a few corner cases where it is temporarily safe, see the [EVC Contract Privileges](#evc-contract-privileges) section).

If the target contract *is* the EVC, in order to preserve the `msg.sender`, the EVC will be self-`call`ed using the `delegatecall`.

If the target contract *is NOT* `msg.sender`, the EVC will only allow the target contract to be called if the `msg.sender` is the owner or the operator of the `onBehalfOfAccount` provided. If that condition is met, the EVC will create a context and call into the target contract with the provided calldata and the `onBehalfOfAccount` account set in the context.

Because vaults can be called directly without going through the EVC, checks may not be deferred when they are invoked. In that case, vaults can use the `call` function so that they can assume that they are always executing within a checks deferred context. If the calling vault specifies the target contract to be their own address (target contract *is* `msg.sender`), the EVC will create a context and call back into the caller with the provided calldata and the `onBehalfOfAccount` account set to whatever was provided by the calling vault. The vault should use `msg.sender` as `onBehalfOfAccount`. In theory a vault could supply any address, but the only other contract that will see this `onBehalfOfAccount` is the vault itself: recall that the `onBehalfOfAccount` should only be trusted when `msg.sender` is the EVC itself. To use `call` in this manner, it is recommended that vaults use a special modifier [`callThroughEVC`](/docs/specs.md#typical-implementation-pattern-of-the-evc-compliant-function-for-a-vault) before its re-entrancy guard modifier. This will take care of routing the calls through the EVC, and the vault can operate under the assumption that the checks are always deferred.

The `call` function also allows to forward the provided value (or full balance of the EVC if max `uint256` was specified). Therefore it can also be used to recover any remaining value in the EVC.

### batch

At the time of this writing, public/private key pair Ethereum accounts (EOAs) cannot directly perform multiple operations within a single transaction, except by invoking a smart contract that will do so on their behalf. The EVC exposes a `batch` function that allows multiple operations to be executed together. This has several advantages for users:

* Atomicity: The user knows that either all of the operations in a batch will execute, or none of them will, so there is no risk of being left with partial or inconsistent positions.
* Gas savings: If contracts are invoked multiple times, then the cost of "cold" access can be amortised across all of the invocations.
* Status check deferrals: Sometimes multiple operations in a batch may require status checks or it is more convenient or efficient to perform some operation that would leave an account/vault in an invalid state, but fix this state in a subsequent operation in a batch. For example, you may want to perform withdrawal and borrow in one batch or borrow and swap *before* you deposit your collateral. With batches, these checks can be performed once at the end of a batch (which can also itself be more gas efficient).

Same as [`call`](#call), batches can be composed of both calls to the EVC itself and external calls. Calling the EVC is how users can enable collateral from within a batch, for example. In order to preserve `msg.sender`, EVC self-`call`s are in fact done with `delegatecall`.

Batches will often be a mixture of external calls, some of which call vaults and some of which call other unrelated contracts. For example, a user might withdraw from one vault, then perform a swap on Uniswap, and then deposit into another vault. Each batch item specifies `onBehalfOfAccount` for which the authentication rules are the same as for [`call`](#call).

### controlCollateral

The `controlCollateral` function can only be used in one specific case: when a controller vault wants to invoke a function on a collateral vault on behalf of the account under its control. The typical use-case for this is a liquidation. The controller vault would detect that an account entered violation due to a price movement, and seize some collateral assets to repay the debt.

This is accomplished by the controller vault calling `controlCollateral`. It passes in the collateral vault as the target collateral and the violator as `onBehalfOfAccount`. The controller would construct a `withdraw` call using the its own address as the `receiver`. From the collateral vault's perspective, this appears as a regular withdrawal, and it does not need to know that the funds are being withdrawn due to a liquidation.

### permit

Instead of invoking the EVC directly, signed messages called `permit`s can also be provided to the EVC. Permits can be invoked by anyone, but they will execute on behalf of the signer of the permit message. They are useful for implementing "gasless" transactions.

Permits are EIP-712 typed data messages with the following fields:

* `signer`: The address to execute the operation on behalf of.
* `nonceNamespace` and `nonce`: Values used to prevent replaying permit messages, and for sequencing (see below)
* `deadline`: A timestamp after which the permit becomes invalid.
* `value`: The value of native currency that is expected to be sent to the EVC
* `data`: Arbitrary calldata that will be used to invoke the EVC. Typically this will contain an invocation of the `batch` method.

There are two types of signature methods supported by permits: ECDSA, which is used by EOAs, and [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) which is used by smart contract wallets. In both cases, the `permit` method can be invoked by any unprivileged address, such as a keeper. If the signature is exactly 65 bytes long, an `ecrecover` is attempted. If the recovered address does not match `signer`, or for signature lengths other than 65, then an ERC-1271 verification is attempted, by staticcalling `isValidSignature` on `signer`.

After verifying `deadline`, `signature`, `nonce`, and `nonceNamespace`, the `data` will be used to invoke the EVC, forwarding the specified `value` (or the full balance of the EVC contract if max `uint256` was specified). Although other methods can be invoked, the most general purpose method to use is `batch`. Inside a batch, each batch item can specify an `onBehalfOfAccount` address. This can be any sub-account of the owner, meaning a signed batch can affect multiple sub-accounts, just as a regular non-permit invocation of `batch` can. If the `signer` is an operator of another account, then the other account can also be specified -- this could be useful for gaslessly invoking a restricted "hot wallet" operator.

Internally, `permit` works by `call`ing `address(this)`, which has the effect of setting `msg.sender` to the EVC itself, indicating to the EVC that the actually authenticated user should be taken from the execution context. It is critical that a permit is the only way for this to happen, otherwise the authentication could be bypassed. Note that the EVC can be self-invoked via `call` and `batch`, but this is done with *delegatecall*, leaving `msg.sender` unchanged.

#### Nonce Namespaces

With nonces, Ethereum transactions enforce that transactions cannot be mined multiple times, and that they are included in the same sequence they were created (with no gaps).

`permit` messages contain two `uint256` fields that can be used to enforce the same restrictions: `nonceNamespace` and `nonce`. Each account owner has a mapping that maps from `nonceNamespace` to `nonce`. In order for a permit message to be valid, the `nonce` value inside the specified `nonceNamespace` must be equal to the `nonce` field in the permit message. Using the permit increments the nonce by one.

The separation of `nonceNamespace` and `nonce` allows users to optionally relax the sequencing restrictions. There are several ways that an owner may choose to use the namespaces:

* Always set `nonceNamespace` to `0`, and sign sequentially increasing `nonce`s. These permit messages will function like Ethereum transactions, and must be mined in order, with no gaps.
* Derive the `nonceNamespace` deterministically from the permit message (for example, a hash of the message fields, excluding `nonceNamespace`), and always set the `nonce` to `0`. These permit messages can be mined in any order, and some may never be mined.
* Some combination of the two approaches. For example, a user could have "regular" and "high priority" namespaces. Normal orders would be included in the regular sequence, while high priority permits are allowed to bypass this queue.

Note that any time sequencing restrictions are relaxed, users should be aware that different orderings of their transactions can have different MEV potential, and they should prepare for their transactions executing in the least favourable order (for them).

Permit messages can be cancelled in three ways:

* Creating a new message with the same nonce and having it included before the unwanted permit (as with Ethereum transactions).
* Invoking the `setNonce` method. This allows users to increase their nonce up to a specified value, potentially cancelling many outstanding permit messages in the process. Note that there is no danger of rendering an account non-functional: Even if a nonce is set to the max `uint256`, there are an effectively unlimited number of other namespaces available.
* Implicitly, by waiting until the `deadline` timestamp expires

### Authorisation

Inside each checks-deferrable call function, an `onBehalfOfAccount` can be specified. The function will determine whether or not `msg.sender` is authorised to perform operations on this account:

* If `msg.sender` has never before interacted with the EVC, if it shares the first 19 bytes with the `onBehalfOfAccount`, then `onBehalfOfAccount` is considered to be a *sub-account* of `msg.sender` and therefore `msg.sender` is authorised. Upon that first interaction with the EVC, `msg.sender` address is stored in EVC's storage as an owner of the group of 256 accounts having the same first 19 bytes.
* If `msg.sender` has interacted with the EVC before and it shares the first 19 bytes with the `onBehalfOfAccount`, its address is supposed to match the one stored in the EVC's storage. If it does, then it is authorised.
* If `msg.sender` has previously been authorized as an [operator](#operators) for the `onBehalfOfAccount`, it is authorised.
* If the `msg.sender` is the EVC itself, then this must be from a permit and the effective sender is taken from the execution context
* In all other cases, the caller is invalid, and the entire transaction will fail.

#### Sub-Accounts

Sub-accounts allow users access to multiple (up to 256) virtual accounts that are entirely isolated from one another. Although multiple separate Ethereum addresses could be used, sub-accounts are often more efficient and convenient because their operations can be grouped together in a batch without setting approvals.

Since an account can only have one controller at a time (except for mid-transaction), sub-accounts are also the only way an Ethereum account can hold multiple Vault borrows concurrently.

The EVC also maintains a look-up mapping `ownerLookup` so sub-accounts can be easily resolved to owner addresses, on- or off-chain. This mapping is populated when an address interacts with the EVC for the first time. In order to resolve a sub-account, the `getAccountOwner` function should be called with a sub-account address. It will either return the account's primary address, or revert with an error if the account has not yet interacted with the EVC.

#### Operators

Operators are a more flexible and powerful version of approvals. While in effect, the operator contract can act on behalf of the specified account. This includes interacting with vaults (ie, withdrawing/borrowing funds), enabling vaults as collateral, etc. Because of this, it is recommended that only trusted and audited contracts, or EOAs held by a trusted individuals, be installed as operators.

Operators have many use cases. For instance, a user might want to install a modifier such as stop-loss/take-profit/trailing-stop to a position in an account. To accomplish this, a special operator contract that allows a "keeper" to close out the user's position when certain conditions are met can be selected as an operator. Multiple operators can be installed per account. Note however that the operators may implement contradictory logic, so care should be taken when installing multiple operators for a single account.

An operator is similar to a controller, in that an account gives considerable permissions to a smart contract (that presumably has been well audited). However, the important difference is that an account owner can always revoke an operator's privileges at any time, however they can not do so with a controller. Instead, the controller must release its own privileges. Another difference is that controllers can not change the account's collateral or controller sets, whereas an operator can.

### Execution Contexts

As mentioned above, when interacting with the EVC, it is often useful to defer certain checks until the end of the transaction. This allows a user to temporarily violate some of the constraints imposed by the vaults, so long as the constraints are satisfied at the end of the transaction.

In order to implement this, the EVC maintains an *execution context* which holds two sets of addresses in regular or transient storage (if supported): `accountStatusChecks` and `vaultStatusChecks`. The execution context will also contain the `onBehalfOfAccount` that has currently been authenticated, so it can be queried by a vault (see [Security Considerations](#security-considerations)).

An execution context will exist for the duration of the checks-deferrable call, and is then discarded. Only one execution context can exist at a time. However, nesting calls *is* allowed (see below).

When the execution context ends, the address sets are iterated over:

* For each address in `accountStatusChecks`, confirm that at most one controller is installed (its `accountControllers` set is of size 0 or 1). If a controller is installed, invoke `checkAccountStatus` on the controller for this account and ensure that the controller is satisfied. If no controller is installed, `checkAccountStatus` is not invoked and the account status is considered valid by default. Hence, [`disableController`](#controller) must be used with care.
* For each address in `vaultStatusChecks`, call `checkVaultStatus` on the vault address stored in the set and ensure that the vault is satisfied.

Additionally, the execution context contains some locks that protect critical regions from re-entrancy (see below).

#### Nested Execution Contexts

If a vault or other contract is invoked via the EVC, and that contract in turn re-invokes the EVC to call another vault/contract, then the execution context is considered nested. The execution context is however *not* treated as a stack. The sets of deferred account and vault status checks are added to, and only after unwinding the final execution context will they be validated.

Internally, the execution context stores a `checksDeferred` flag that is set each time a checks-deferrable call is started and cleared only when its value before the call was `false`. Once the flag is cleared, the deferred checks get performed. Nesting calls is useful because otherwise calling contracts via the EVC that themselves want to call other contracts through the EVC would be more complicated.

The previous value of `onBehalfOfAccount` is stored in a local "cache" variable and is subsequently restored after invoking the target contract. This ensures that contracts can rely on the `onBehalfOfAccount` at all times when `msg.sender` is the EVC (see [Authentication by Vaults](#authentication-by-vaults)). However, when `msg.sender` is not the EVC, vaults cannot rely on `onBehalfOfAccount` because it could have been changed by a nested context.

#### checksInProgress

The EVC invokes the `checkAccountStatus` and `checkVaultStatus` callbacks using low-level `call` instead of `staticcall` so that controllers can checkpoint state during these operations. However, because of this there is a danger that the EVC could be re-entered during these operations, either directly by a controller, or indirectly by a contract it invokes.

Because of this, the EVC's execution context maintains a `checksInProgress` mutex that is acquired before unwinding the sets of accounts and vaults that need checking. This mutex is also checked during operations that alter these sets. If it did not do this, then information cached by the higher-level unwinding function (such as the sizes of the sets) could become inconsistent with the underlying storage, which could be used to bypass these critical checks.

#### controlCollateralInProgress

The typical use-case for collateral control is for a liability vault to seize collateral assets during a liquidation flow.

However, when interacting with complicated vaults that may invoke external contracts during a withdraw/transfer, a liability vault may want to ensure that no *other* collaterals are removed during the seizure.

In order to simplify the implementation of this check, the `controlCollateralInProgress` mutex is locked while invoking a collateral vault during the `controlCollateral` flow. While locked, no accounts' collateral or controller sets can be modified.

Additionally, during collateral control, the EVC cannot be re-entered via `call`, `batch`, `controlCollateral` or `permit`.

#### Extra Information

The execution context also indicates some extra information, which could be useful if a contract wants to know extra information about the EVC's authentication. This includes information about simulation and operator authentication status.

### Simulations

The EVC also supports executing batches in a "simulation" mode. This is only intended to be invoked "off-chain", and is useful for user interfaces because they can show the user what the expected outcome of a sequence of operations will be.

Simulations work by actually performing the requested operations but then reverting, which (if called on-chain) reverts all the effects. Although simple in principle, there are a number of design elements involved:

* Intermediate read-only queries can be inserted into a batch to gather simulated data useful for display
* The results are available even if status checks would cause a failure, for example so that a user can see exactly what is causing the failure
* Although internally simulations work by reverting, the recommended interface returns it as regular return data, which causes fewer compatibility problems (sometimes error data is mangled or dropped). This is the reason for `batchRevert`: You can't do a "try/catch" without an external call, so this must be an external function, although we recommend using the `batchSimulation` entry point instead.
* Simulations don't have the side-effect of making regular batches create large return-data (which would be gas inefficient)


## Transient Storage

In order to maintain the execution context, access to the same variables must occur from different invocations of the EVC. This means that they must be held in storage, and not memory. Unfortunately, storage is expensive compared to memory. Luckily, the EVM protocol may soon specify a new type of memory lifetime: transient storage that is accessible to multiple invocations, but is inexpensive to access.

In order to take advantage of transient storage, the contracts have been structured to keep all the variables that should be stored in transient storage in a separate base class contract `TransientStorage`. By optionally overriding this at compile-time, both old and new networks can be supported.


## Security Considerations

### Authentication by Vaults

Vaults out-source their authentication to the EVC, but are responsible for authorisation themselves.

In order to support sub-accounts, operators, and being able to control collateral (ie, in liquidations), vaults can be invoked via the EVC's `call`, `batch`, or `controlCollateral` functions, which will then execute the desired operations on the vault. However, the vault will see the EVC as the `msg.sender`.

When a vault detects that `msg.sender` is the EVC, it should call back into the EVC to retrieve the current execution context using `getCurrentOnBehalfOfAccount`. This will tell the vault two things:

* The `onBehalfOfAccount` which indicates the account that has been authenticated by the EVC. The vault should consider this the "true" value of `msg.sender` for authorisation purposes.
* The `controllerEnabled` which indicates whether or not a vault is currently enabled as a controller for the `onBehalfOfAccount` account. This information is needed if the vault is performing an operation (such as a borrow) that requires it to be the controller for an account. The caller of `getCurrentOnBehalfOfAccount` itself passes the vault it is interested in via the `controllerToCheck` parameter. When `controllerToCheck` is set to the zero address, the value returned is always `false`.

### EVC Contract Privileges

Because the EVC contract can be made to invoke any arbitrary target contract with any arbitrary calldata, it should never be given any privileges, or hold any native currency or tokens.

The only exception to this is mid-transaction inside of a batch. If one batch item temporarily moves value or tokens into the EVC, but a subsequent batch item moves it out, then as long as no untrusted code runs in between, it is safe. However, moving tokens to the EVC is often unnecessary because tokens can be moved immediately to their final destination with `transferFrom` and by setting various `recipient` parameters in contracts.

One exception to this is wrapping ETH into WETH. The deposit method will always credit the caller with the WETH tokens. In this case, the user must transfer the WETH in a subsequent batch item (ideally the batch item immediately after the deposit using `call` function).

One area where the untrustable EVC address may cause problems is tokens that implement hooks/callbacks, such as ERC-777 tokens. In this case, somebody could install a hook for the EVC as a recipient, and cause inbound transfers to fail, or possibly even be redirected. The EVC doesn't attempt to solve this issue and care should be taken when interacting with contracts which implement hooks/callbacks.

### Read-only Re-entrancy

The non-transient storage maintained by the EVC *can* be read while checks are deferred. In particular, this includes the lists of collaterals and controllers registered for a given account.

This should not result in "read-only re-entrancy" problems, because each individual operation will leave these lists in a consistent state. In particular, for a controller to be released, that controller itself must invoke the release, which typically means the debt has been repaid.

If an external contract attempted to read the collateral or controller states of an account in order to enforce some policy of its own, then it is possible that a user could defer its liquidity check, repay the loan, invoke the external contract, and then re-take the loan. In this case the external contract would see the controller as being released. However, this same action could be done outside of a checks-deferrable call by simply taking a flash loan from an external system, rather than using the checks-deferrable call.
