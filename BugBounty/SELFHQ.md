## Project : SELFHQ

## Severity : Low

## Title : Accounting Inconsistency in     SELFPresale Refund Mechanism

## Summary

A vulnerability exists in `selfPresale.sol` refund mechanism where per-round contribution tracking becomes permanently inconsistent with global accounting. When users claim refunds, the contract correctly updates the global totalRaised but fails to update per-round `rounds[i].raised` values, creating irreversible accounting corruption that misrepresents actual funds raised per round and potentially breaks round finalization logic as  `rounds[i].raised` becomes irreversibly inflated.

## Core Issue
When a user contributes the `rounds[i].raised` becomes incremented by the amount contributed  but the `claimRefund()` function clears user-specific `contributionsByRound` mapping without using this data to update the corresponding `rounds[i].raised` values. This creates a permanent discrepancy between:

- Global raised amount (totalRaised) - correctly decreased

- Per-round raised amounts (rounds[i].raised) - incorrectly left unchanged

- User contribution tracking (contributionsByRound) - correctly cleared


Current Implementation Flow in claimRefund():

```solidity
// Global amount decreased ✓
totalRaised -= amount;

// User contributions cleared without updating rounds[] ✗
for (uint256 i = 0; i < 5; i++) {
    contributionsByRound[msg.sender][i] = 0; // DATA LOST HERE
}

// Result: rounds[i].raised remains inflated
```

## Vulnerability Flow:


##### Initial State:

 User contributes $1,000 in Round 0

 `rounds[0].raised` = 1,000

 `contributionsByRound[user][0]` = 1,000

 `totalRaised` = 1,000



##### Refund Claim:

 User calls `claimRefund()`

 `totalRaised` = 0 (correct)

 `contributionsByRound[user][0]` = 0 (correct)

 rounds[0].raised = 1,000 (INCORRECT - should be 0)



##### Resulting State Corruption:

 Round 0 shows $1,000 raised when contract actually has $0

 Statistics permanently misleading due to inflated value

 Impossible to determine actual net contributions per round


## Proof of Concept
Step-by-Step Scenario


 ##### Initial Setup:

 - Contract initialized with 5 rounds

 - Round 0 target: $1,500,000

 ##### Step 1: Normal Contribution

 - Alice contributes $1,000,000 in Round 0

 State:

 `rounds[0].raised` = 1,000,000

 contributionsByRound[Alice][0] = 1,000,000

 `totalRaised` = 1,000,000

 Round 0 appears 66.6% complete


##### Step 2: Refund Scenario Triggers

-  Soft cap ($500k) not reached after all rounds

 - Admin enables refunds via enableRefunds()

 - Refund window opens for 30 days

 ##### Step 3: Alice Claims Refund

 - Alice calls `claimRefund()`
 

 Function executes:

 ```solidity
 totalRaised -= 1,000,000; // Now 0
 // Loop clears contributionsByRound but doesn't update rounds[0].raised
 for (uint i = 0; i < 5; i++) {
    contributionsByRound[Alice][i] = 0;
 }
```


- Resulting state:

 `totalRaised` = 0 ✓

 `contributionsByRound[Alice][0]` = 0 ✓

 `rounds[0].raised` = 1,000,000 ✗ (STILL INFLATED)


##### Step 4: Irreversible Corruption

 - New user Bob queries contract state

 - He sees: rounds[0].raised = 1,000,000

- But actual funds in contract: $0

- No way to correct this without manual intervention

- All future reports show incorrect round performance


## Recommended Fix

Immediate Correction:
- Update claimRefund() to properly synchronize all accounting layers because:


- Sum of all rounds raised  (rounds[i].raised) must always equal totalRaised

- User's total contributions must equal sum of per-round contributions

- Contract USDC balance should equal totalRaised when no refunds pending

```solidity
function claimRefund() external nonReentrant {
    // ... existing validation checks ...
    
    UserContribution storage userContrib = contributions[msg.sender];
    uint256 amount = userContrib.totalUSDC;
    
    // Decrease totalRaised
    if (totalRaised >= amount) {
        totalRaised -= amount;
    } else {
        totalRaised = 0;
    }
    
    // CRITICAL FIX: Update per-round raised amounts
    for (uint256 i = 0; i < 5; i++) {
        uint256 roundAmount = contributionsByRound[msg.sender][i];
        if (roundAmount > 0) {
            // Ensure no underflow
            require(rounds[i].raised >= roundAmount, "Round raised underflow");
            rounds[i].raised -= roundAmount;
            contributionsByRound[msg.sender][i] = 0;
        }
    }
    
    // ... rest of existing function ...
}
```

## TEAM RESPONSE
> Hi Okiki
>  
> Good catch - this is legitimate albeit low risk that we hadn't arrived at yet. It seems that:
>  
> No fund loss — Users still get their full refunds
> Timing constraint — Refunds only happen AFTER all rounds are finalized (presale failed), so the inflated rounds[i].raised won't affect contribution logic

> Impact is data integrity — External dashboards, analytics, and getCurrentRound() views show incorrect data

>  
> However - it indeed violates accounting invariants (sum of rounds[i].raised should equal totalRaised) and misleading for transparency/reporting

## BOUNTY

Team paid out 25000 SLF token for this disclosure.