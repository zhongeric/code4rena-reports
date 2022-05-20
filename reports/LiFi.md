# C4 LI.FI finding: Users can swap for no cost when the Facet contract calling LibSwap has a balance of ERC20 or native tokens

### **Judge decision: Downgraded to med risk and confirmed as valid**
https://github.com/code-423n4/2022-03-lifinance-findings/issues/10

C4 finding submitted: (risk = 3 (High Risk))

Mar 24, 2022, 11:44 PM

# Lines of code

https://github.com/code-423n4/2022-03-lifinance/blob/699c2305fcfb6fe8862b75b26d1d8a2f46a551e6/src/Libraries/LibSwap.sol#L29-L42


# Vulnerability details

## Impact
When the Facet contract calling `Swagger->_executeSwaps->LibSwap.swap` contains a balance of ERC20 or native tokens, the origin tokens are not transferred out of the user's account to preform the swap but rather taken from the contract's balance, and the user is sent the swapped amount in recipient tokens.

If a swap is initiated on any of the Facet contracts (Anyswap, Generic, etc.) with balance of the source token for the swap, an attacker can easily exploit the contract by sending a swap transaction with a `fromAmount` of just less than the balance held by the contract to bypass the conditional check on `LibSwap.sol:34`. Then, the external call on [`LibSwap.sol : line 42`](https://github.com/code-423n4/2022-03-lifinance/blob/699c2305fcfb6fe8862b75b26d1d8a2f46a551e6/src/Libraries/LibSwap.sol#L42) will be executed, sending the calldata for the swap, and resulting in the contract's own funds being used. The `_swapData` fed into the swap function on the facet is also not verified and can be manually constructed by an attacker.

## Proof of Concept
1. Copy and paste the `Generic Swap Facet` test suite in `GenericSwapFacet.test.ts` into a new file.
2. Find the first test which simulates standard behavior `(it('performs a swap then starts bridge transaction on the sending chain')`
3. Before initiating the swap, have **bob** seed another account, **alice** for example, with 1000 USDC. These are not the actual values in the accounts but this is an example.
4. Now, have `alice` send the 1000 USDC to the contract `GenericSwapFacet`.
5. The rest of the code can be left untouched, but be sure to change the amounts being swapped `(amountIn and amountOut)` to be < 1000 USDC, (i.e. 500 & 490).
6. Observe that Bob's USDC balance will be unchanged after the swap, but he will have gained 500 hUSDC. The contract's balance of USDC will have decreased from 1000 to 500.

This scenario shows that any user can swap using the funds in the contract rather than their own funds if there is a balance. I'm not sure if there are intended scenarios where the swap Facet contracts might hold ERC20 tokens and/or native (ETH, matic), but in this case, the funds would be vulnerable.

## Tools Used
Hardhat, ethers, normal testing suite

## Recommended Mitigation Steps
Either always call `transferFromERC20` before swapping in `LibSwap.sol:35` or, implement the logic from `AnyswapFacet.sol:43` elsewhere - permalink [here](https://github.com/code-423n4/2022-03-lifinance/blob/699c2305fcfb6fe8862b75b26d1d8a2f46a551e6/src/Facets/AnyswapFacet.sol#L43)