# C4 Backed Protocol finding: Malicious users can frontrun borrowers trying to repay loans, causing DoS and possibly a loan default

### **Judge decision: High severity, duplicate of [#24](https://github.com/code-423n4/2022-04-backed-findings/issues/24)**

C4 finding submitted: (risk = 3 (High Risk))

Thu, Apr 7, 6:20 PM

# Lines of code

https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L129-L226
https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L230-L250


# Vulnerability details

## Impact
Attackers can listen for a borrower to call `repayAndCloseLoan` on a specific `loanId`, and frontrun their transaction with a call to `lend`, creating a new loan with an increased amount, causing the borrower's transaction to fail due to the new loanAmount being greater than initially anticipated.

This can cause many issues: a borrower may repeatedly find that they cannot repay and close their loan, and while the duration of the loan is extended on takeover, the borrower must keep retrying and/or try again later if they do not want to default on the loan.

If the underlying collateral is valuable enough (either originally in value, or it appreciated), an attacker would be motivated to continuously prevent the loan from being closed in an attempt to seize the collateral.

If the duration of the loan, `minDurationSeconds`, is low enough, the attacker can simply takeover the loan with a 10% more amount, and hope that the borrower will fail to repay and close the new loan that now has a new deadline so that they can call `seizeCollateral`.

A dedicated attacker can continue to deny the borrower from repaying their loan indefinitely. If the borrower is a regular user, they likely will not be able to keep up and end up defaulting.

## Proof of Concept
A simple test suite can be written where the borrower calls `repayAndCloseLoan` on their outstanding loan, an attacker can see that txn in the mempool and submit a higher-priority one calling `lend` with 10% more amount, causing the call to `repayAndCloseLoan` to revert due to insufficient balance. If the borrower does not repay the loan before the new deadline, the attacker can call `seizeCollateral` and receive the collateralized NFT.

## Tools Used
Manual code review, vscode

## Recommended Mitigation Steps
Front-running mitigation methods such as [decay periods](https://blog.1inch.io/how-1inch-protects-users-from-front-running-a51ec6e3c6d5) (durations after which certain txns are not allowed / conditions are changed) could be helpful.

# C4 Backed Protocol finding: NFTLoanFacilitator lend() function vulnerable to re-entrancy

### **Judge decision: Invalid due to lack of POC**

C4 finding submitted: (risk = 2 (Med Risk))

Wed, Apr 6, 7:00 PM

# Lines of code

https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L155
https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L200-L204
https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L215-L219


# Vulnerability details

## Impact
The `lend()` function is vulnerable to re-entrancy from potentially malicious ERC20 contracts.

## Proof of Concept
The team seems to be aware of this already as evidenced by the `MaliciousERC20` tests, however I think it's worth pointing out that `closeLoan` naturally prevents re-entrancy because of the `notClosed` modifier. Currently, a user can enter the `lend` function many times using the `safeTransfer` calls [here](https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L155) and [here](https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L200-L204), and is thus able to take over the same loan multiple times.

While I don't see a direct attack vector exploiting this currently, this is unusual behavior and it would be more relevant if the protocol wanted to implement some restriction on the frequency of loan takeovers and/or lending in general.

## Tools Used
Manual code review, vscode

## Recommended Mitigation Steps
Add a re-entrancy guard for future-proofing

# C4 Backed Protocol finding: A borrower can lend to themselves, receiving a borrowTicket and lendTicket for the same loanId

### **Judge decision: Invalid, circular borrowing is allowed. Failed to show adequate attack vector.**

C4 finding submitted: (risk = 2 (Med Risk))

Thu, Apr 7, 6:32 PM

# Lines of code

https://github.com/code-423n4/2022-04-backed/blob/e8015d7c4b295af131f017e646ba1b99c8f608f0/contracts/NFTLoanFacilitator.sol#L129-L227


# Vulnerability details

## Impact
A borrower of a loan can takeover the same loan and become its lender. They will then own the corresponding `borrowTicket` and `lendTicket` for that `loanId`.

If some system is established in the future that rewards lenders (holders of `lendTickets`), a malicious user could lend to themselves with an interest rate of 0 or arbitrarily small, and receive those benefits.

I'm not aware of any current way that this can be exploited for gain, but it is unexpected behavior and can lead to weird business logic being executed.

## Proof of Concept
Create a test suite where the borrower of a loan calls `lend()` using the same `loanId` of the loan that they created. Regardless of if there is an existing lender, they will be minted a `lendTicket`.

## Tools Used
Manual code review, Vscode

## Recommended Mitigation Steps
Add a require check in `lend()` that the `msg.sender` does not hold a `borrowTicket` already for that `loanId`.