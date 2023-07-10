MohammedRizwan

medium

# Approve 0 first for tokens like USDT

## Summary
Approve 0 first for tokens like USDT

## Vulnerability Detail
## Impact

Some tokens (like USDT) do not work when changing the allowance from an existing non-zero allowance value. For example Tether (USDT)’s approve() function will revert if the current approval is not zero, to protect against front-running changes of approvals. [Link to usdt contract reference(SLOC 199-209)](https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code)

approve is actually vulnerable to a sandwich attack as explained in the following document and this check for allowance doesn't actually avoid it.

Reference document link- https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit

In ERC20, front running attack is possible via approve() function,

Reference link for better understanding- https://blog.smartdec.net/erc20-approve-issue-in-simple-words-a41aaf47bca6

In the protocol, all functions using safeApprove() must be first approved by zero. Below functions are called to make ERC20 approvals. But it does not approve 0 first.

```Solidity
File: src/LimitOrderRegistry.sol

    function _mintPosition(
        PoolData memory data,
        int24 upper,
        int24 lower,
        uint128 amount0,
        uint128 amount1,
        bool direction,
        uint256 deadline
    ) internal returns (uint256) {
        if (direction) data.token0.safeApprove(address(POSITION_MANAGER), amount0);
        else data.token1.safeApprove(address(POSITION_MANAGER), amount1);
    
       // Some code
```
and


```Solidity
File: src/LimitOrderRegistry.sol

    function _addToPosition(
        PoolData memory data,
        uint256 positionId,
        uint128 amount0,
        uint128 amount1,
        bool direction,
        uint256 deadline
    ) internal {
        if (direction) data.token0.safeApprove(address(POSITION_MANAGER), amount0);
        else data.token1.safeApprove(address(POSITION_MANAGER), amount1);

       // Some code
```

## Code Snippet
https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/753974a4ba5b07ec076c5e5bcbe7277e5921be4b/src/LimitOrderRegistry.sol#L1168-L1169

https://github.com/crispymangoes/uniswap-v3-limit-orders/blob/753974a4ba5b07ec076c5e5bcbe7277e5921be4b/src/LimitOrderRegistry.sol#L1223-L1224

## Tool used
Manual Review

## Recommendation
Approve 0 first and Use OpenZeppelin’s SafeERC20.
