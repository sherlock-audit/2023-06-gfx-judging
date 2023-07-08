xiaoming90

medium

# Owner unable to collect fulfillment fee from certain users due to revert error

## Summary

Certain users might not be able to call the `claimOrder` function under certain conditions, resulting in the owner being unable to collect fulfillment fees from the users.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-06-gfx/blob/main/uniswap-v3-limit-orders/src/LimitOrderRegistry.sol#L721

```solidity
File: LimitOrderRegistry.sol
696:     function claimOrder(uint128 batchId, address user) external payable returns (ERC20, uint256) {
697:         Claim storage userClaim = claim[batchId];
698:         if (!userClaim.isReadyForClaim) revert LimitOrderRegistry__OrderNotReadyToClaim(batchId);
699:         uint256 depositAmount = batchIdToUserDepositAmount[batchId][user];
700:         if (depositAmount == 0) revert LimitOrderRegistry__UserNotFound(user, batchId);
701: 
702:         // Zero out user balance.
703:         delete batchIdToUserDepositAmount[batchId][user];
704: 
705:         // Calculate owed amount.
706:         uint256 totalTokenDeposited;
707:         uint256 totalTokenOut;
708:         ERC20 tokenOut;
709: 
710:         // again, remembering that direction == true means that the input token is token0.
711:         if (userClaim.direction) {
712:             totalTokenDeposited = userClaim.token0Amount;
713:             totalTokenOut = userClaim.token1Amount;
714:             tokenOut = poolToData[userClaim.pool].token1;
715:         } else {
716:             totalTokenDeposited = userClaim.token1Amount;
717:             totalTokenOut = userClaim.token0Amount;
718:             tokenOut = poolToData[userClaim.pool].token0;
719:         }
720: 
721:         uint256 owed = (totalTokenOut * depositAmount) / totalTokenDeposited;
722: 
723:         // Transfer tokens owed to user.
724:         tokenOut.safeTransfer(user, owed);
```

Assume the following:

- SHIB has 18 decimals of precision, while USDC has 6.
- Alice (Small Trader) deposited 10 SHIB while Bob (Big Whale) deposited 100000000 SHIB. 
- The batch order was fulfilled, and it claimed 9 USDC (`totalTokenOut`)

The following formula and code compute the number of swapped/claimed USDC tokens a user is entitled to. 

```solidity
owed = (totalTokenOut * depositAmount) / totalTokenDeposited
owed = (9 USDC * 10 SHIB) / 100000000 SHIB
owed = (9 * 10^6 * 10 * 10^18) / (100000000 * 10^18)
owed = (9 * 10^6 * 10) / (100000000)
owed = 90000000 / 100000000
owed = 0 USDC (Round down)
```

Based on the above assumptions and computation, Alice will receive zero tokens in return due to a rounding error in Solidity.

The issue will be aggravated under the following conditions:

- If the difference in the precision between `token0` and `token1` in the pool is larger
- The token is a stablecoin, which will attract a lot of liquidity within a small price range (e.g. `$0.95 ~ $1.05`)

Note: Some tokens have a low decimal of 2 (e.g., [Gemini USD](https://etherscan.io/token/0x056Fd409E1d7A124BD7017459dFEa2F387b6d5Cd?a=0x5f65f7b609678448494De4C87521CdF6cEf1e932)), while others have a high decimal of 24 (e.g., `YAM-V2` has 24). Refer to https://github.com/d-xo/weird-erc20#low-decimals

The rounding down to zero is unavoidable in this scenario due to how values are represented. It is not possible to send Alice 0.9 WEI of USDC. The smallest possible amount is 1 WEI.

In this case, it will attempt to transfer a zero amount of `tokenOut,` which might result in a revert as some tokens disallow the transfer of zero value. As a result, when users call the `claimOrder` function, it will revert, and the owner will not be able to collect the fulfillment fee from the users.

https://github.com/sherlock-audit/2023-06-gfx/blob/main/uniswap-v3-limit-orders/src/LimitOrderRegistry.sol#L724

```solidity
723:         // Transfer tokens owed to user.
724:         tokenOut.safeTransfer(user, owed);
```

## Impact

When a user cannot call the `claimOrder` function due to the revert error, the owner will not be able to collect the fulfillment fee from the user, resulting in a loss of fee for the owner.

## Code Snippet

https://github.com/sherlock-audit/2023-06-gfx/blob/main/uniswap-v3-limit-orders/src/LimitOrderRegistry.sol#L724

## Tool used

Manual Review

## Recommendation

Consider only transferring the assets if the amount is more than zero.

```diff
uint256 owed = (totalTokenOut * depositAmount) / totalTokenDeposited;

// Transfer tokens owed to user.
- tokenOut.safeTransfer(user, owed);
+ if (owed > 0) tokenOut.safeTransfer(user, owed);
```