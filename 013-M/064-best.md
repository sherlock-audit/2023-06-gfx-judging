peanuts

medium

# Use of slot0 to determine tick position may not be ideal because of possible price manipulation

## Summary

slot0 returns the current tick of the price position of the pool, which can be manipulated easily in the same block if the liquidity in the pool is low or through flash loans. 

## Vulnerability Detail

Uniswap'sV3 tick in slot0 returns:
> The current tick of the pool, i.e. according to the last tick transition that was run. This value may not always be equal to SqrtTickMath getTickAtSqrtRatio(sqrtPriceX96) if the price is on a tick boundary.

```solidity
 function slot0(
  ) external view returns (uint160 sqrtPriceX96, int24 tick, uint16 observationIndex, uint16 observationCardinality, uint16 observationCardinalityNext, uint8 feeProtocol, bool unlocked)
```

https://docs.uniswap.org/contracts/v3/reference/core/interfaces/pool/IUniswapV3PoolState

Since Tick = log1.0001(Price), if price changes, the current tick will also change. If the price is manipulated through a large deposit into the pool or through a flashloan and the liquidity of the pool is low, then the tick will also change.

An example of manipulating tick to the advantage is to manipulate the current price until a desired tick is obtained so that the order becomes OTM, and then call `cancelOrder`.

```solidity
    function cancelOrder(
        UniswapV3Pool pool,
        int24 targetTick,
        bool direction,
        uint256 deadline
    ) external returns (uint128 amount0, uint128 amount1, uint128 batchId) {
        // defined here since we want to grab it out of the closure
        uint256 positionId;
        // this is in a closure to avoid namespace pollusion
        {
            // Make sure order is OTM.
            (, int24 tick, , , , , ) = pool.slot0();
```


## Impact

## Code Snippet

https://github.com/sherlock-audit/2023-06-gfx/blob/main/uniswap-v3-limit-orders/src/LimitOrderRegistry.sol#L761

https://github.com/sherlock-audit/2023-06-gfx/blob/main/uniswap-v3-limit-orders/src/LimitOrderRegistry.sol#L585

## Tool used

Manual Review

## Recommendation

Use the `observe` function in UniswapV3's Pool and get the time weighted average tick value instead of using the current tick.