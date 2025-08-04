## Platform : Codehawks

## Severity : High 

## Title
Hardcoded Exchange Rate Calculation Will Cause Undercollateralization, Blocking Withdrawals


## Summary
The StabilityPool contract’s `getExchangeRate()` function is hardcoded to return 1e18, resulting in deTokens being minted and redeemed at a fixed 1:1 ratio regardless of the pool’s actual collateral. This flaw will cause undercollateralization and ultimately block user withdrawals.

## Vulnerability Details
In the StabilityPool contract, the `getExchangeRate()` function is implemented as follows:
```solidity
function getExchangeRate() public view returns (uint256) {
    // uint256 totalDeCRVUSD = deToken.totalSupply();
    // uint256 totalRcrvUSD = rToken.balanceOf(address(this));
    // if (totalDeCRVUSD == 0 || totalRcrvUSD == 0) return 1e18;
    // uint256 scalingFactor = 10**(18 + deTokenDecimals - rTokenDecimals);
    // return (totalRcrvUSD * scalingFactor) / totalDeCRVUSD;
    return 1e18;
}
```
​
causing this function to always returns 1e18:

```solidity
Scenerio Flow
Deposit Flow: When a user deposits rTokens via the deposit() function, calculateDeCRVUSDAmount() uses getExchangeRate() to mint deTokens at a fixed 1:1 ratio. For example, when Bob deposits rTokens, he receives an equivalent amount of deTokens regardless of the actual pool collateral.

Liquidation Event: Later, if a liquidation event occurs—such as when Shelia is liquidated via the liquidateBorrower() function—the pool’s rToken balance is reduced significantly.

Withdrawal Flow: When Bob subsequently attempts to withdraw his rTokens using the withdraw() function, calculateRcrvUSDAmount() again uses the fixed exchange rate (1e18). Since the pool’s collateral has been depleted by Shelia’s liquidation, the calculated redeemable rToken amount exceeds the available collateral, causing Bob’s withdrawal to fail.
```

## Impact
In this example the fixed exchange rate causes the StabilityPool to become undercollateralized once rToken reserves drop due to Shelia’s liquidation.

As a result, Bob will be unable to withdraw his funds because the contract’s calculations assume full collateralization, leading to failed transactions and locked assets.

Ultimately, users are directly affected by this flaw. The inability to redeem deTokens for the correct amount of rTokens compromises the integrity of the protocol and puts user funds at risk.

## Tools Used
Manual Review

## Recommendations
Modify the `getExchangeRate()` function to calculate the exchange rate dynamically based on the actual rToken reserves and the deToken total supply. For example:

```solidity

function getExchangeRate() public view returns (uint256) {
    uint256 totalDeCRVUSD = deToken.totalSupply();
    uint256 totalRcrvUSD = rToken.balanceOf(address(this));
    if (totalDeCRVUSD == 0 || totalRcrvUSD == 0) return 1e18;
    uint256 scalingFactor = 10**(18 + deTokenDecimals - rTokenDecimals);
    return (totalRcrvUSD * scalingFactor) / totalDeCRVUSD;
}
```
​
This update ensures that:

Deposits mint deTokens in proportion to the actual collateral in the pool.

Withdrawals correctly redeem rTokens based on current pool conditions.

Under adverse scenarios such as Shelia’s liquidation, Bob’s deTokens accurately reflect the diminished collateral, preventing blocked withdrawals and safeguarding user funds.

-----------------------------------------------------------------------

## Medium 1

## Severity : Medium

## Title
EmergencyRevoke Doesn’t Update categoryUsed, Permanently Locking Vesting Allocations

## Summary
The emergencyRevoke function in `RAACReleaseOrchestrator` contract does not update categoryUsed, causing revoked tokens to remain accounted for in the category usage. This leads to incorrect allocation tracking, making it impossible to allocate new vesting schedules even when tokens should be available.

## Vulnerability Details
When a vesting schedule is revoked, the contract deletes the schedule but does not update categoryUsed. Since categoryUsed continues to count revoked tokens, new schedules cannot be created under the same category even if the tokens should be available.

## Affected Code
emergencyRevoke() 

## Initial Setup

PRIVATE_SALE allocation: 10,000,000 RAAC

Used before transactions: 8,000,000 RAAC

Remaining: 2,000,000 RAAC

Step 1: Freddie Allocates to Ummi

- Freddie (admin) vests 2,000,000 RAAC to Ummi (investor).

- PRIVATE_SALE allocation is now fully used (10,000,000 RAAC).

Step 2: Freddie Revokes Ummi’s Vesting

- One year later, Ummi has claimed 500,000 RAAC.

- Freddie revokes the remaining 1,500,000 RAAC.

- categoryUsed remains 10,000,000 instead of updating to 8,500,000.

Step 3: Dan's Vesting Fails

- Freddie attempts to vest 1,500,000 RAAC to Dan (new investor).

- The transaction fails with "CategoryAllocationExceeded" error.

- The contract incorrectly treats PRIVATE_SALE as fully used, preventing new allocations.

## Impact
- Incorrect Allocation Tracking: The total available allocation per category is misrepresented.

- Denial of Future Vesting Schedules: Future vesting schedules will be incorrectly blocked due to the overestimated usage.

- Inefficient Token Utilization: The inability to properly reclaim allocation will lead to funds being locked or unused.

- this also make `getCategoryDetails()` returns a false representation of categoryUsed, making it unreliable for tracking allocation status.

> Note : This flaw effectively reduces the total allocatable supply of RAAC tokens over time as revoked allocations become unusable, directly conflicting with the token's distribution plan (65% allocated for vesting)

## Tools Used
Manual Review

## Recommendations
1 - Store the vesting category within VestingSchedule for reference.

2 - Reduce categoryUsed upon revocation to accurately track available allocations.

```solidity
struct VestingSchedule {
    uint256 totalAmount;
    uint256 releasedAmount;
    uint256 startTime;
    uint256 duration;
    bool initialized;
+   bytes32 category; // Store category for reference
}
​
function createVestingSchedule(
    address beneficiary,
    bytes32 category,
    uint256 amount,
    uint256 startTime
) external onlyRole(ORCHESTRATOR_ROLE) whenNotPaused {
    ...
+   schedule.category = category; // Store category in the struct
}
​
function emergencyRevoke(address beneficiary) external onlyRole(EMERGENCY_ROLE) {
    VestingSchedule storage schedule = vestingSchedules[beneficiary];
    if (!schedule.initialized) revert NoVestingSchedule();
​
    uint256 unreleasedAmount = schedule.totalAmount - schedule.releasedAmount;
+   categoryUsed[schedule.category] -= unrealeasedAmount; // Adjust category usage
​
    delete vestingSchedules[beneficiary];
​
    if (unreleasedAmount > 0) {
        raacToken.transfer(address(this), unreleasedAmount);
        emit EmergencyWithdraw(beneficiary, unreleasedAmount);
    }
​
    emit VestingScheduleRevoked(beneficiary);
}
```
-----------------------------------------------------------------------

## Medium 2

## Severity : Medium

## Title
Lack of Vote Delay Check Leads to Rapid Gauge Weight Manipulation

Summary
The GaugeController contract declares a VOTE_DELAY of 10 days but does not enforce it in the vote() function. This oversight allows users to cast multiple votes in rapid succession.

## Vulnerability Details
Affected Code :

GaugeController.sol, specifically the vote() function -

https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/governance/gauges/GaugeController.sol#L184C3-L203C6

Issue:


Although `lastVoteTime[msg.sender]` and `VOTE_DELAY` are defined, there is no require statement to ensure the caller waits 10 days before voting again. Nor is `lastVoteTime[msg.sender]` updated to the current timestamp after a vote.

Exploit Scenario:


An attacker with a high veRAAC balance can call `vote()` multiple times in quick succession to rapidly shift gauge weights. This repeated manipulation distorts the reward allocation process and undermines governance.

## Impact
Rapid, repeated voting can skew gauge weights, giving an attacker undue influence over emission distributions.

Other users’ votes become less meaningful if one party can continuously re-vote to adjust weights.

## Tools Used
Manual code review of GaugeController.sol, focusing on the vote() function implementation.

## Recommendations
Enforce Vote Delay: Add a check in the vote() function


Track Vote Timestamp: Immediately after a successful vote, update the lastVoteTime for msg.sender

```solidity
@@ function vote(address gauge, uint256 weight) external override whenNotPaused {
-    if (!isGauge(gauge)) revert GaugeNotFound();
+    if (!isGauge(gauge)) revert GaugeNotFound();
+    require(
+        block.timestamp >= lastVoteTime[msg.sender] + VOTE_DELAY,
+        "Vote delay not yet passed"
+    );
 
     if (weight > WEIGHT_PRECISION) revert InvalidWeight();
 
     uint256 votingPower = veRAACToken.balanceOf(msg.sender);
     if (votingPower == 0) revert NoVotingPower();
@@
-    uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
-    userGaugeVotes[msg.sender][gauge] = weight;
+    uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
+    userGaugeVotes[msg.sender][gauge] = weight;
+    lastVoteTime[msg.sender] = block.timestamp;
​
     _updateGaugeWeight(gauge, oldWeight, weight, votingPower);
     emit WeightUpdated(gauge, oldWeight, weight);
}
```
​