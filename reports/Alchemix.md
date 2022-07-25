# C4 Alchemix finding: Alchemist can mint `AlTokens` above their assigned ceiling by calling `lowerHasMinted()`

### **Judge decision: Med severity, picked to be included in final report [#1](https://github.com/code-423n4/2022-05-alchemix-findings/issues/166)**

C4 finding submitted: (risk = 2 (Med Risk))

Wed, May 18, 1:03 PM

# Lines of code

https://github.com/code-423n4/2022-05-alchemix/blob/de65c34c7b6e4e94662bf508e214dcbf327984f4/contracts-full/AlchemicTokenV2Base.sol#L111-L124
https://github.com/code-423n4/2022-05-alchemix/blob/de65c34c7b6e4e94662bf508e214dcbf327984f4/contracts-full/AlchemicTokenV2Base.sol#L189-L191

# Vulnerability details

## Impact

An alchemist / user can mint more than their alloted amount of AlTokens by calling `lowerHasMinted()` before they reach their minting cap.

## Proof of Concept

Function `mint()` in `AlchemicTokenV2Base.sol`

```solidity
function mint(address recipient, uint256 amount) external onlyWhitelisted {
  if (paused[msg.sender]) {
    revert IllegalState();
  }

  uint256 total = amount + totalMinted[msg.sender];
  if (total > mintCeiling[msg.sender]) {
    revert IllegalState();
  }

  totalMinted[msg.sender] = total;

  _mint(recipient, amount);
}
```

Note the require conditional check that `total > mintCeiling[msg.sender]`.

In the same contract, there is the function `lowerHasMinted()` with the same permission level as mint and is thus callable by the same user as well.

```solidity
function lowerHasMinted(uint256 amount) external onlyWhitelisted {
  totalMinted[msg.sender] = totalMinted[msg.sender] - amount;
}
```

It is clear that a user can accumulate an infinite (within supply) amount of AlTokens by calling `lowerHasMinted()` before any action that would make them exceed their minting cap.

## Tools Used

Manual review, VScode

## Recommended Mitigation Steps

Change the permissioning on `lowerHasMinted()` to be restricted to a higher permissioned role like `onlySentinel()` , or deprecate this function as I could not find any uses of it throughout the codebase or in tests.

# QA report

Relevant portions of code are marked with `@audit` tag.

### **Low-01 Check return boolean value of call in `gALCX.sol:reApprove()`**

```solidity
  function reApprove() public {
      // @audit check success return value for approve
      bool success = alcx.approve(address(pools), type(uint256).max);
  }
```

Suggested: `require(success, "failed to approve");`

### **Low-02 Potential rug vector for malicious admins in `EthAssetManager.sol:sweepToken()`**

```solidity
  function sweepToken(address token, uint256 amount) external lock onlyAdmin {
      // @audit prevent withdraw of specific tokens like `curveToken`
      SafeERC20.safeTransfer(address(token), msg.sender, amount);
      emit SweepToken(token, amount);
  }
```

The functionality of sweepToken is used to send any amount of any ERC20 token held by the EthAssetManager to the admin. However, there is no restriction on which tokens can be sent. Thus, the admin, which can be an EOA address, can sweep all underlying `curveTokens` to themselves, stealing the rewards generated in Convex from the `rewardReceiver` (called by `claimRewards()`).

Same issue is in `ThreePoolAssetManager.sol`.

**Recommended Mitigation Steps:**
Do not allow sweeping of the `curveToken` and/or any other critical tokens.

### **QA-01: Move `gALCX.sol:reApprove()` a line under to public functions section**

```solidity
  // @audit move to section below
  function reApprove() public {
      bool success = alcx.approve(address(pools), type(uint256).max);
  }

  // PUBLIC FUNCTIONS
```

### **QA-02: Emit event in `TransmuterV2.sol:setCollateralSource()`**

```solidity
function setCollateralSource(address _newCollateralSource) external {
  _onlyAdmin();
  buffer = _newCollateralSource;
  // @audit QA - emit NewCollateralSource event here
}
```

Events are often emitted for changes to state throughout the codebase,and a changing of the collateral source should warrant one too.
