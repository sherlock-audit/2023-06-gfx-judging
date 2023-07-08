Avci

medium

# getGasPrice() doesn't check Arbitrum l2 chainlink feed is active

## Summary
getGasPrice() doesn't check Arbitrum l2 chainlink feed is active
## Vulnerability Detail
When utilizing Chainlink in L2 chains like Arbitrum, it's important to ensure that the prices provided are not falsely perceived as fresh, even when the sequencer is down. This vulnerability could potentially be exploited by malicious actors to gain an unfair advantage.

If the sequencer is down, messages cannot be transmitted from L1 to L2, and no L2 transactions are executed. Instead, messages are enqueued in the CanonicalTransactionChain on L1


On the L1 network:

1.A network of node operators runs the external adapter to post the latest sequencer status to the AggregatorProxy contract and relays the status to the Aggregator contract. The Aggregator contract calls the validate function in the OptimismValidator contract.

2.The OptimismValidator contract calls the sendMessage function in the L1CrossDomainMessenger contract. This message contains instructions to call the updateStatus(bool status, uint64 timestamp) function in the sequencer uptime feed deployed on the L2 network.

3.The L1CrossDomainMessenger contract calls the enqueue function to enqueue a new message to the CanonicalTransactionChain.

4.The Sequencer processes the transaction enqueued in the CanonicalTransactionChain contract to send it to the L2 contract.

On the L2 network:

1.The Sequencer posts the message to the L2CrossDomainMessenger contract.

2.The L2CrossDomainMessenger contract relays the message to the OptimismSequencerUptimeFeed contract.

3.The message relayed by the L2CrossDomainMessenger contains instructions to call updateStatus in the OptimismSequencerUptimeFeed contract.

4.Consumers can then read from the AggregatorProxy contract, which fetches the latest round data from the OptimismSequencerUptimeFeed contract.

## Impact
could potentially be exploited by malicious actors to gain an unfair advantage.
## Code Snippet
```solidity
 function getGasPrice() public view returns (uint256) {
        // If gas feed is set use it.
        if (fastGasFeed != address(0)) {
            (, int256 _answer, , uint256 _timestamp, ) = IChainlinkAggregator(fastGasFeed).latestRoundData();
            uint256 timeSinceLastUpdate = block.timestamp - _timestamp;
            // Check answer is not stale.
            if (timeSinceLastUpdate > FAST_GAS_HEARTBEAT) {
                // If answer is stale use owner set value.
                // Multiply by 1e9 to convert gas price to gwei
                return uint256(upkeepGasPrice) * 1e9;
            } else {
                // Else use the datafeed value.
                uint256 answer = uint256(_answer);
                return answer;
            }
        }
        // Else use owner set value.
        return uint256(upkeepGasPrice) * 1e9; // Multiply by 1e9 to convert gas price to gwei
    }
```
https://github.com/sherlock-audit/2023-06-gfx/blob/main/uniswap-v3-limit-orders/src/LimitOrderRegistry.sol#L1447-L1465
## Tool used

Manual Review

## Recommendation
The recommendation is to implement a check for the sequencer in the L2 version of the contract, and a code example of Chainlink can be found at https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code.

CHAINLINK DOC : https://docs.chain.link/data-feeds/l2-sequencer-feeds#example-code


Same issue in other contests

https://github.com/sherlock-audit/2023-02-bond-judging/issues/1

https://github.com/sherlock-audit/2022-11-sentiment-judging/issues/3

https://github.com/sherlock-audit/2023-01-sentiment-judging/issues/16


