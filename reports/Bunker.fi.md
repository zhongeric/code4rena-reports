# C4 bunker.finance finding: Use of ChainlinkFeed `latestAnswer` in PriceOracleImplementation is deprecated and not sufficiently validated

### **Judge decision: Med severity, duplicate of [#1](https://github.com/code-423n4/2022-05-bunker-findings/issues/1)**

C4 finding submitted: (risk = 2 (Med Risk))

May 3, 2022, 10:56 AM

# Lines of code

https://github.com/bunkerfinance/bunker-protocol/blob/752126094691e7457d08fc62a6a5006df59bd2fe/contracts/PriceOracleImplementation.sol#L29-L32

# Vulnerability details

## Impact

`PriceOracleImplementation.sol` uses the `latestAnswer()` function on the mainnet deployed ChainlinkFeed for the USDC oracle price. However, this function is deprecated as outlined in the comments of the deployed Chainlink contract (see POC below).

Notably, `latestAnswer` does not provide the other data `(roundId, answer, startedAt, updatedAt, answeredInRound)` necessary for sufficient oracle price validation as compared to the recommended `latestRoundData`. Thus in the current implementation, there is no check for round completeness and the reported price can be stale, leading to incorrect `result` return value.

## Proof of Concept

Per the comments in the Chainlink contract on mainnet:

```
/**
 * @notice Reads the current answer from aggregator delegated to.
 *
 * @dev #[deprecated] Use latestRoundData instead. This does not error if no
 * answer has been reached, it will simply return 0. Either wait to point to
 * an already answered Aggregator or use the recommended latestRoundData
 * instead which includes better verification information.
 */
function latestAnswer()
```

https://etherscan.io/address/0x986b5E1e1755e3C2440e960477f25201B0a8bbD4#code

## Tools Used

Manual Review

## Recommended Mitigation Steps

Use `latestRoundData()` and validate the values returned to check for staleness and completeness.

```solidity
  function getUnderlyingPrice(CToken cToken) external view returns (uint) {
      ...

      // For now, we only have USDC and ETH.
      (uint80 roundID, int256 usdcPrice, , uint256 timestamp, uint80 answeredInRound) = ChainlinkFeed(0x986b5E1e1755e3C2440e960477f25201B0a8bbD4).latestRoundData();
      require(answeredInRound >= roundID, "ChainLink: Stale price");
      require(timestamp != 0, "ChainLink: Round not complete");
      // Additionally, you could validate that the price is not <= 0 and revert here instead of in other places
      if (usdcPrice <= 0) {
          return 0;
      }

      ...

      return result;
  }
```

https://docs.chain.link/docs/faq/#how-can-i-check-if-the-answer-to-a-round-is-being-carried-over-from-a-previous-round
https://docs.chain.link/docs/price-feeds-api-reference/#getrounddata
