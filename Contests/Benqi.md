## Platform : Codehawks

## Severity : Low

## Title
Incorrect Event Emission in `SetSlippage()` Causing Misrepresentation of Updated Slippage Value


## Summary
The `setSlippage` function in the `Staking.sol` contract emits incorrect values in the `SlippageUpdated` event. Specifically, the event logs the original `_slippage` input rather than the corrected `boundedSlippage` value stored in the slippage state variable. This inconsistency can mislead off-chain systems, such as frontends and monitoring tools, that rely on event data for tracking contract behavior.

## Vulnerability Details
The `setSlippage` function performs validation to ensure the `_slippage` parameter falls within predefined bounds (`minSlippage` and `maxSlippage`). If `_slippage` is out of bounds, it is clamped to `boundedSlippage`, which is then stored in the slippage state variable. However, the `SlippageUpdated` event incorrectly emits the original `_slippage` value instead of the actual stored `boundedSlippage`.

### Affected Code Snippet:
```solidity
function setSlippage(
    uint256 _slippage
) external onlyRole(BENQI_ADMIN_ROLE) {
    uint256 boundedSlippage = _slippage;
    if (_slippage < minSlippage) {
        boundedSlippage = minSlippage;
    } else if (_slippage > maxSlippage) {
        boundedSlippage = maxSlippage;
    }
    uint256 oldSlippage = slippage;
    slippage = boundedSlippage;
    emit SlippageUpdated(oldSlippage, _slippage); // Incorrect: Emits the wrong value
}
```
## Scenario Flow:

* **Admin:** As an admin, Alice calls `setSlippage` with `_slippage = 150` (above `maxSlippage` of 100).

* **Contract Behavior:** The contract clamps `_slippage` to `boundedSlippage = 100` and updates the `slippage` state variable to 100.

* **Event Emission:** The `SlippageUpdated` event logs `oldSlippage` and the original `_slippage = 150` instead of the actual stored value, `boundedSlippage = 100`.

* **Bob (User):** Bob, relying on event logs, sees a slippage of 150 on the frontend. This incorrect value leads him to believe the contract is configured differently than it is, creating confusion.


## Impact

Event subscribers, such as frontends or monitoring systems, will track `_slippage` instead of the corrected `boundedSlippage`, leading to discrepancies.

Users providing out-of-bound slippage values may see incorrect data reflected in frontends, creating a poor user experience.

The event logged will become unreliable for debugging and analysis as it provides incorrect value.

## Tools Used

Manual review.

## Recommendations

Update the `setSlippage` function to emit the actual stored value, `boundedSlippage`, in the `SlippageUpdated` event to ensure consistency between on-chain state and off-chain tracking systems.

#### Corrected Code:
```diff
function setSlippage(
    uint256 _slippage
) external onlyRole(BENQI_ADMIN_ROLE) {
    uint256 boundedSlippage = _slippage;
    if (_slippage < minSlippage) {
        boundedSlippage = minSlippage;
    } else if (_slippage > maxSlippage) {
        boundedSlippage = maxSlippage;
    }
    uint256 oldSlippage = slippage;
    slippage = boundedSlippage;
-   emit SlippageUpdated(oldSlippage, _slippage); // Incorrect: Emits the wrong value
+   emit SlippageUpdated(oldSlippage, boundedSlippage); // Correct: Emits the actual stored value
}
```
By addressing this issue, the contract will provide accurate event data, ensuring better synchronization with off-chain systems and improving overall maintainability and user experience
