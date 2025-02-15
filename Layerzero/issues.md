# Solidity issues

## Category : Fee Management Issues

### Issue 1: Missing Receive Function for Fee Refunds

When users overpay for cross-chain messages, LayerZero attempts to refund excess native gas tokens (like ETH). If the designated `_refundAddress` is a contract, it **must** implement a `receive()` or `fallback()` function to be able to receive the refunded amount. Otherwise, the transaction will revert, and the refund will be lost.

You can pay for the `_lzSend` function in LayerZero using either the native gas token or the ZRO token. When sending a message through LayerZero, fees are incurred for gas on the source chain, fees paid to DVNs (Decentralized Verification Networks) for validating the message, and gas on the destination chain. LayerZero conveniently bundles all these fees into a single fee.

To get an estimate of how much gas a message will cost, you can implement a `quote()` function within the OApp contract to return an estimate from the Endpoint contract to use as a recommended `msg.value`. The `_quote` can be returned in either the native gas token or ZRO token. It is important that the arguments passed into the `quote()` function identically match the parameters used in the `lzSend()` function to ensure accurate pricing.

The parameters for `lzSend` are:

- `_dstEid`: The destination Endpoint ID.
- `_message`: The message to be sent.
- `_options`: Message execution options for protocol handling.
- `MessagingFee`: Specifies what token will be used to pay for the transaction.
  - `nativeFee`: Fee amount in native gas token.
  - `lzTokenFee`: Fee amount in ZRO token.
- `_refundAddress`: Specifies the address to which any excess fees should be refunded.

> 🐰 The `_refundAddress` if it is a contract, it should always implement a `receive()` function, and the refund address should be designated.

### References

https://docs.layerzero.network/v2/developers/evm/oapp/overview

### [Case Study : Nexus Protocol](https://solodit.cyfrin.io/issues/m-01-layerzero-fee-refunds-cannot-be-processed-pashov-audit-group-none-nexus_2024-11-29-markdown)

Affected Contracts: DepositUSD, DepositETHBera, DepositUSDBera

In the Nexus protocol, the `DepositUSD`, `DepositETHBera`, and `DepositUSDBera` contracts were intended to serve as the refund address for LayerZero. However, these contracts lacked a `receive()` or `fallback()` function, making them unable to receive refunds, which caused transactions to revert when excess fees were sent.

Root Cause: Missing receive/fallback functions

Impact: Unable to process LayerZero fee refunds

### Issue 2: Incorrect Refund Address Configuration

The refund address should go to the msg.sender who initiated the transaction.

### [Case study : Nexus protocol](https://solodit.cyfrin.io/issues/m-04-layerzero-fee-refunds-misdirected-to-deposit-contracts-pashov-audit-group-none-nexus_2024-11-29-markdown)

In the case of the Nexus protocol, the refund address was incorrectly set to the addresses of `DepositUSD`, `DepositETHBera`, and `DepositUSDBera`. The refund address should be `msg.sender`, the address that initiated the transaction.

Impact: This misdirection of refunds results in the sender bearing the full cost of initiating the cross-chain transaction without receiving any compensation for the excess fee.

### [Case Study: StationX Protocol](https://solodit.cyfrin.io/issues/c-01-the-layerzero-implementation-contract-is-the-receiver-of-daos-fees-on-cross-chain-buy-operations-pashov-audit-group-none-stationx-markdown)

In the StationX protocol, the LayerZero implementation contract was incorrectly set as the receiver of DAO fees for cross-chain buy operations, rather than the intended DAO treasury address.

Impact: This misconfiguration resulted in DAO fees being sent to the wrong address, causing a loss of revenue for the DAO. The fees became permanently locked in the LayerZero implementation contract since it lacked the ability to withdraw or transfer them.

> Make sure the lz fees is always sent to the right address

⭕ Lookout : if msg.sender is calling program a -> and program a calls program b -> so lz fees will be returned to the program a, we want the fees to be returned to the initial msg.sender.

Another instance of the same issue : https://solodit.cyfrin.io/issues/c-01-the-layerzero-implementation-contract-is-the-receiver-of-daos-fees-on-cross-chain-buy-operations-pashov-audit-group-none-stationx-markdown

### Missing gas

### Issue : Missing check on minimum gas for cross-chain messaging

When working with cross-chain applications in LayerZero, you need to supply a specific gas amount to ensure the successful execution of transactions on the destination chain
Yes, when working with cross-chain applications in LayerZero, you need to supply a specific gas amount to ensure the successful execution of transactions on the destination chain. Here's a breakdown of what you need to know:

LayerZero provides message execution options that allow you to specify the `gas_limit` and `msg.value` used in the Executor's transaction for message delivery on EVM chains. You can use options like `lzReceiveOption`, `lzComposeOption`, and `lzNativeDropOption`.

when working with cross-chain applications in LayerZero, you need to supply a specific gas amount to ensure the successful execution of transactions on the destination chain. This gas payment is made in the form of the native currency of the destination chain (e.g., GLMR, ETH).

To get the fees in a LayerZero contract, you can use the `quote()` function. Here's how it works:

- **Purpose of the `quote()` function:** The `quote()` function estimates the gas costs for sending messages across chains. It returns an estimate from the Endpoint contract that can be used as a recommended `msg.value`.

- **How it works:** The LayerZero Endpoint provides an on-chain quote mechanism to determine the cost of sending a message to the destination chain. The `OApp.quote` function invokes an internal `OAppSender._quote` to estimate fees.

- **Return Value:** The `_quote` function can return the fee in either the native gas token or in ZRO token.

- **When to Use:** Because cross-chain gas fees are dynamic, generate the quote right before calling `_lzSend` to ensure accurate pricing.

- **Arguments:** Ensure that the arguments passed into the `quote()` function identically match the parameters used in the `lzSend()` function. Mismatched parameters can cause errors if your `msg.value` does not match the actual gas quote.

```
function quote(
    uint32 _dstEid, // Destination chain's endpoint ID.
    string memory _message, // The message to send.
    bytes calldata _options, // Message execution options
    bool _payInLzToken // boolean for which token to return fee in
) public view returns (uint256 nativeFee, uint256 lzTokenFee) {
    bytes memory _payload = abi.encode(_message);
    MessagingFee memory fee = _quote(_dstEid, _payload, _options, _payInLzToken);
    return (fee.nativeFee, fee.lzTokenFee);
}
```

[Case study](https://solodit.cyfrin.io/issues/h-02-due-to-missing-checks-on-minimum-gas-passed-through-layerzero-executions-can-fail-on-the-destination-chain-code4rena-decent-decent-git)4

In the decent protocol, there was no lower limit on the gas, layerzero makes sures that there is 2,00,000 gas sent to cover in all the chains.

The DecentEthRouter is designed to work with LayerZero’s cross-chain messaging system. Its role is to forward user-initiated ETH transfers (and transfers with payload) to a destination chain. To do this, the contract combines a fixed relay gas amount (hardcoded as 100,000 gas) with a user-supplied parameter (`dstGasForCall`) to set the total gas that will be forwarded to the destination chain call. The expectation is that users will first call an estimation function (estimateSendAndCallFee) so they know how much gas is needed. However, the contract does not enforce a minimum value for `dstGasForCall`. This lack of validation means that if a user (or an attacker) supplies an arbitrarily low gas value, the destination call may not receive enough gas and will run out-of-gas.

Issue Description
When the destination chain’s function call is made via LayerZero, the call must have a sufficient gas limit to complete successfully. The DecentEthRouter adds the fixed GAS_FOR_RELAY (100,000 gas) to the user-specified `dstGasForCall`, then passes that sum as adapter parameters in the call. There is no check ensuring that `dstGasForCall` is at least as high as the estimated fees (typically nativeFee + zroFee). Consequently, a user can pass a low value—either inadvertently or maliciously—which results in:

Insufficient Gas for Execution: The destination function call does not have enough gas to complete its logic and reverts with an out-of-gas error.
State Transition to STORED: Even though the call fails, LayerZero’s design treats the message as “delivered” and moves it into a STORED state. A STORED message is not cleared automatically; instead, it blocks further messages on that channel until the message is retried or manually cleared.
This behavior is problematic because it means that one failed message can effectively freeze the channel between the source and destination chains, leading to a denial-of-service (DoS) scenario on cross-chain communications

**Instances**

- https://solodit.cyfrin.io/issues/m-07-redemptionsender-should-estimate-fees-to-prevent-failed-transactions-code4rena-velodrome-finance-velodrome-finance-git
- https://solodit.cyfrin.io/issues/trst-m-10-mozbridge-underestimates-gas-for-sending-of-moz-messages-trust-security-none-mozaic-archimedes-markdown_

## 2. State Management Issues

### Issue 3: Global State Synchronization Problems

**Problem**: Cross-chain state synchronization can lead to state overwrites without proper locking mechanisms.

**Example State Structure**:

```rust
struct OmniChainData {
        uint256 normalizedAmount;
        uint256 vaultValue;
        uint256 cdsPoolValue;
        uint256 totalCDSPool;
        uint256 collateralRemainingInWithdraw;
        uint256 collateralValueRemainingInWithdraw;
        uint128 noOfLiquidations;
        uint64 nonce;
        uint64 cdsCount;
        uint256 totalCdsDepositedAmount;
        uint256 totalCdsDepositedAmountWithOptionFees;
        uint256 totalAvailableLiquidationAmount;
        uint256 usdtAmountDepositedTillNow;
        uint256 burnedUSDaInRedeem;
        uint128 lastCumulativeRate;
        uint256 totalVolumeOfBorrowersAmountinWei;
        uint256 totalVolumeOfBorrowersAmountinUSD;
        uint128 noOfBorrowers;
        uint256 totalInterest;
        uint256 abondUSDaPool;
        uint256 collateralProfitsOfLiquidators;
        uint256 usdaGainedFromLiquidation;
        uint256 totalInterestFromLiquidation;
        uint256 interestFromExternalProtocolDuringLiquidation;
        uint256 totalNoOfDepositIndices;
        uint256 totalVolumeOfBorrowersAmountLiquidatedInWei;
        uint128 cumulativeValue; // cumulative value
        bool cumulativeValueSign; // cumulative value sign whether its positive or not negative
        uint256 downsideProtected;

    }
```

The design of this protocol ensures that any changes to omniChainData on one chain are synchronized to other chains via LayerZero.

However, due to the lack of a locking mechanism and the non-atomic nature of cross-chain operations, this could lead to overwriting of global states. The following is a simple explanation using a borrow example:

1. Alice deposits 1 ETH on Optimism, and almost simultaneously, Bob deposits 100 ETH on the Mode chain.
2. The totalVolumeOfBorrowersAmountinWei on Optimism becomes 1e18, while on the Mode chain, it becomes 100 \* 1e18.
3. Both Optimism and Mode chains send globalVariables.send to synchronize their local data to the other chain.
4. As a result, totalVolumeOfBorrowersAmountinWei on Optimism becomes 100 \* 1e18, while on the Mode chain, it becomes 1e18.

Both chains now hold incorrect totalVolumeOfBorrowersAmountinWei values. The correct value should be 101 \* 1e18.

The fundamental issue is that when Alice deposits on Optimism, the cross-chain synchronization to the Mode chain cannot prevent users from performing nearly simultaneous actions on the Mode chain, leading to overwriting of global states.

Furthermore, there could also be scenarios where a message is successfully sent via LayerZero on Optimism but encounters an error when executing receive() on the Mode chain, leading to additional inconsistencies.

> ⭕ This is issue which can happen in any cross-chain messaging app.

## 3. Ordered Execution of Messages

### Issue 4: Missing Ordered Execution of Messages

In LayerZero V2, the Ordered Execution Option ensures that cross-chain messages are delivered in a specific order, overriding the default behavior of Unordered Message Delivery
🟢 Ensures that the cross-chain messages are executed in a specific order.

References

- https://docs.layerzero.network/v2/developers/evm/protocol-gas-settings/options#orderedexecution-option

By adding this option, the Executor will utilize Ordered Message Delivery. This overrides the default behavior of Unordered Message Delivery.

### [Case study : Orderly Solana](https://solodit.cyfrin.io/issues/m-1-missing-layerzero-ordered-execution-option-for-orderly-chain-messages-sherlock-orderly-solana-vault-contract-git)

In the orderly contest, The Solana vault contract enforced ordered message processing through nonce validation, like this

```solidity
   if (orderDelivery) {
       require(_origin.nonce == inboundNonce + 1, "Invalid inbound nonce");
   }
   inboundNonce = _origin.nonce;
```

This meant protocol heavily relied on the order in which the transcactions took place.

But in layerzero, this restriction should be enforeced using the layerzero orderexeuction flag to ensure which was missing up here.

```solidity
   bytes memory withdrawOptions = OptionsBuilder.newOptions().addExecutorLzReceiveOption(
       msgOptions[uint8(MsgCodec.MsgType.Withdraw)].gas,
       msgOptions[uint8(MsgCodec.MsgType.Withdraw)].value
   );
```

**Impact**:

- Messages use **unordered delivery** (Default in LayerZero V2)
- Nonces may arrive out-of-sequence
- Messages can be processed in any order

If two withdrawal messages (A and B) are sent in quick succession and arrive at the Solana vault in reverse order (B before A):

Message B's nonce will not match inboundNonce + 1, causing rejection.

The Orderly Chain's ledger has already reduced the user's balance for both withdrawals.

The user loses funds equivalent to withdrawal B, as the Solana vault never processes it.

```text

   [Orderly Chain]          [LayerZero]          [Solana Vault]
       Send Withdraw A (nonce 1) --> (delayed)
       Send Withdraw B (nonce 2) --> (arrives first)
                               │
                               └──> Vault processes nonce 2 first
                                    Reverts: "Invalid inbound nonce"
```

Remediation

```solidity
bytes memory withdrawOptions = OptionsBuilder.newOptions()
    .addExecutorLzReceiveOption(
        msgOptions[uint8(MsgCodec.MsgType.Withdraw)].gas,
        msgOptions[uint8(MsgCodec.MsgType.Withdraw)].value
    )
    .addOrderedExecutionOption(); // Critical fix
```

⭕ How to lookout these issues?

> Whenever the execution of order is crucial, then orderedExecutionOption should be enforced.

### Issue 6: Vault does not have a way to withdraw native tokens

If a contract is set as refund address, then it must also have a function to withdraw those naitve tokens.

[Case study: Mozaic Archimedes](https://solodit.cyfrin.io/issues/trst-m-9-vault-does-not-have-a-way-to-withdraw-native-tokens-trust-security-none-mozaic-archimedes-markdown_)

While using the layerzero cross-chain messaging, A contract address was used as refund address , but there was no function to withdraw those native tokens.

## Layerzero v1 Blocking issue

### Issue 5: Default Blocking Behavior in LayerZero v1

- LayerZero v1's `LzApp` has a default blocking mechanism. If a message is delivered to the receiving chain and its execution fails (e.g., due to an "Out of Gas" error or logical error), the entire cross-chain message exchange channel for that application becomes blocked. This means no subsequent messages can be processed until the failed message issue is resolved.
- This blocking behavior is by design in some cases. It allows for analysis and potential retry or discard of transactions deemed incorrect or malicious. However, it can lead to increased downtime if the analysis of failed transactions cannot keep up with the message throughput.

⭕ How to lookout for issues realted to this : Where layerzero v1 is used.
Attack vector: You have to make sure the transaction passes in the sender chain and fails on another chain, this will lead to failure in the whole tx.

[Case study : Station X ](https://solodit.cyfrin.io/issues/m-11-layerzero-channel-for-deployments-can-be-blocked-due-to-reverts-pashov-audit-group-none-stationx-markdown)

In the stationX, `LayerZeroDeployer` contract used LayerZero v1, which has a blocking nature by default, and its receiver function doesn't implement a non-blocking pattern. Any revert during the cross-chain execution will block the whole channel, preventing future DAO deployments, until the protocol admin unlocks it via `forceResume()`.

The attacker in the function could send a arbitary number of entries in the arrays, and send a fixed gas to make the tx fail in the reciever's end.

In the source chain, the adversary can send any amount of gas they want, so the transaction can succeed even with a reasonably big number of `_admins`. The problem is in the destination chain as the gas is limited to `2_000_000`:

```solidity
    endpoint.send{value: msg.value}(
        uint16(_dstChainId),
        abi.encodePacked(dstCommLayer[uint16(_dstChainId)], address(this)),
        _payload,
        payable(_refundAd),
        address(0x0),
@>      abi.encodePacked(uint16(1), uint256(2_000_000))
    );
```

So, it is possible to make a transaction run successfully on the source chain and revert to the destination chain, blocking all subsequent cross-chain transactions given LayerZero v1 default behavior.

[Case study : Station X](https://solodit.cyfrin.io/issues/m-12-non-blocking-layerzero-cross-chain-buy-operations-can-be-blocked-pashov-audit-group-none-stationx-markdown)

When a cross-chain buy operation is performed, a LayerZero message is sent with a fixed `600_000` gas limit to execute in the destination chain:

```Solidity
    endpoint.send{value: msg.value}(
        _dstChainId,
        abi.encodePacked(dstCommLayer[uint16(_dstChainId)], address(this)),
        _payload,
        payable(refundAd),
        address(0x0),
@>      abi.encodePacked(uint16(1), uint256(600_000))
    );
```

LayerZero v1 has a blocking behavior by default, and `LayerZeroImpl` attempts to implement the non-blocking pattern via code:

```Solidity
    function _blockingLzReceive(uint16 _srcChainId, bytes memory _srcAddress, uint64 _nonce, bytes memory _payload) internal {
@>      (bool success, bytes memory reason) = address(this).excessivelySafeCall(
            gasleft(),
            150,
            abi.encodeWithSelector(this.nonblockingLzReceive.selector, _srcChainId, _srcAddress, _nonce, _payload)
        );

        if (!success) {
@>          _storeFailedMessage(_srcChainId, _srcAddress, _nonce, _payload, reason);
        }
    }

    function _storeFailedMessage(
        uint16 _srcChainId,
        bytes memory _srcAddress,
        uint64 _nonce,
        bytes memory _payload,
        bytes memory _reason
    ) internal virtual {
@>      failedMessages[_srcChainId][_srcAddress][_nonce] = keccak256(_payload);
@>      emit MessageFailed(_srcChainId, _srcAddress, _nonce, _payload, _reason);
    }
```

In theory, this should work as expected by trying to execute the cross-chain transaction and saving any failed message for later processing in case of failure.

But if for some reason the transaction reverts on the `failedMessages[_srcChainId][_srcAddress][_nonce] = keccak256(_payload);` statement, LayerZero will consider the whole cross-chain transaction as failed, and it will block the channel for future messages.

Setting a value from zero to a non-zero value costs at least 20,000 gas. So if the remaining gas up to that point is less than that, the transaction will revert, and LayerZero will block the channel.

Buy messages are executed with a fixed 600,000 gas limit, as previously mentioned. According to [EIP-150](https://eips.ethereum.org/EIPS/eip-150) 63/64 or \~590k would be forwarded to the external call. If all gas is used in `excessivelySafeCall()`, that would leave \~9,000 gas for the rest of the execution. This isn't enough to cover storing `failedMessages` + emitting the `MessageFailed` event and the transaction will revert.

`excessivelySafeCall()` calls `crossChainMint()`, and it should not be possible to make that function spend 590k gas with normal usage, but an adversary can provide for example a big enough `_merkleProof` array that would consume the gas when hashing over and over in a big loop. They can also send a big enough `_tokenURI` that would be too expensive to store when minting the token.

### Some instances covering the same issue

- https://github.com/pashov/audits/blob/master/team/md/StationX-security-review.md

References :

- https://reports.zellic.io/publications/orderly-network/findings/high-crosschainrelayupgradeable-layerzero-has-default-blocking-behavior
- https://github.com/code-423n4/2022-05-velodrome-findings/issues/83

## Missing Quote for gas

### Issue 5: No function to get the layerzero fees in stargate

In the stargate function, there is a function called `quoteLayerZero` which is reponsible for getting the fees required to pay for a cross-chain message in layerzero implementation,

No function of presence of `quoteLayerZero` before sending the function will be considered a low finding

[Case study : TapiaDao ](https://solodit.cyfrin.io/issues/m-08-missing-implementation-of-the-quotelayerzerofee-in-stargatelbphelpersol-pashov-none-tapiocadao-markdown)

When making cross-chain message calls that require sending ETH as msg.value, it's critical to first query the exact fee amount needed using the quoteLayerZeroFee() function. Without implementing this quote functionality, the contract has no way to determine the correct fee amount to send. This can result in:

1. Sending insufficient fees - causing the cross-chain transaction to fail
2. Sending excess fees - which get locked in the contract
3. Inconsistent gas costs across different chains

The proper implementation should:

- Call quoteLayerZeroFee() before the cross-chain transaction
- Pass the quoted amount as msg.value
- Handle cases where the quote may change between calls

## Recommendation issues

## V1

### Issue : No presence of forceResumeReceive function

The `forceResumeReceive` function in LayerZero is used to clear stalled message queues between chains, allowing user applications (UAs) to manually remove stuck payloads and resume message processing.

Key Functions:

- **Queue Unblocking**: Clears blocked messages between contracts on different chains.
- **Malicious Message Removal**: Eliminates harmful messages received by a contract.

Functionality:

- Checks for cached messages and verifies the existence of a payload for the specified source chain ID and address.
- Ensures only authorized callers, specifically the intended recipient contract, can clear the message.
- Resets `payloadLength`, `dstAddress`, and `payloadHash` to zero, effectively clearing the message queue.
- Emits a `UaForceResumeReceive` event to log the action.

Conditions for execution:

- The function will revert if no messages are cached. This safeguards against malicious UA behavior.
- The destination address must match the message sender.

Important considerations:

- **Security**: User Applications should implement the `ILayerZeroUserApplicationConfig` interface, including the `forceResumeReceive` function. This provides a way for the owner/multisig to unblock the message queue if something unexpected occurs.

[Case Study : Tradable](https://solodit.cyfrin.io/issues/layerzero-recommendations-not-followed-zokyo-none-tradable-markdown)

The TradableSettingsMessageAdapter contract has some potential issues related to its LayerZero implementation that could lead to blocked message queues and potential denial-of-service vulnerabilities. Here's a breakdown:

- **Missing ILayerZeroApplicationConfig Implementation:** The contract doesn't implement the `ILayerZeroApplicationConfig` interface. Implementing this interface provides the `forceResumeReceive` function, which allows the owner or a multi-signature wallet to manually unblock the queue of messages if an unexpected issue arises[1].

- **Missing Non-Blocking Receiver Implementation:** The contract lacks a non-blocking receiver, which LayerZero recommends. Non-blocking receivers prevent failed transactions from blocking future messages[1][2]. Without it, if a message encounters an error (like an out-of-gas exception) on the destination chain, subsequent messages can get stuck in the queue[1].

- **Missing `setTrustedRemote()` call:** The `setTrustedRemote()` function, which establishes trusted endpoints for cross-chain communication, might be missing. If inbound communication isn't enabled using `setTrustedRemote()`, it could lead to a denial-of-service vulnerability[2].

### Issue :

to address very much https://solodit.cyfrin.io/issues/h-2-malicious-user-can-use-an-excessively-large-_toaddress-in-oftcoresendfrom-to-break-layerzero-communication-sherlock-uxd-uxd-protocol-git

Oft standard breaks the precision https://solodit.cyfrin.io/issues/m-20-dust-amounts-will-be-removed-in-oft-tokens-sherlock-autonomint-colored-dollar-v1-git

Usage of \_debug force receive function https://solodit.cyfrin.io/issues/debug_forcereceivemessage-is-dangerous-and-could-be-abused-cantina-none-sweep-n-flip-pdf

Solana ATA frozen issue https://solodit.cyfrin.io/issues/denial-of-service-due-to-authority-change-ottersec-none-orderly-network-pdf

If the ata
