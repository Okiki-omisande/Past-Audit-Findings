## High-1 

Public Access Control Vulnerability in changeOwner() Function


## Summary

The `changeOwner()` function is publicly accessible and does not check if the caller is the current owner of the contract before changing the ownership of the contract. This allows anyone to call the function and transfer ownership to themselves or another malicious address

## Vulnerability Details

The `changeOwner()` function can be called by anyone due to its public visibility. Additionally, there is no validation in place to check whether `msg.sender` is the current owner. As a result, an attacker can claim ownership of the contract and take control, leading to potential abuse, including the withdrawal of funds or unauthorized modifications.

## Impact

Any user, including a malicious actor, can take over ownership of the contract. This gives the attacker full control, allowing them to access owner-restricted functions, such as :

* setBoxPrice() : attacker can claim ownership and modify the box price 

* withdrawFunds() : attacker can claim ownership and proceed to call the withdrawFunds() method and completely drain the contract

* addReward() : after claiming ownership, an attacker can add new reward by calling the addReward() function

All users interacting with the contract, including buyers and reward eligible users, are at risk since an attacker gaining control would disrupt the intended contract functionality and compromise the trust in the system and potentially making users lose their rewards.

## Tools Used

manual review

## Recommendations

Implement an ownership check to ensure that only the current owner can call the `changeOwner()` function. Use the `require()` statement to enforce this access control.

As I have indicated below :

```diff

- function changeOwner(address _newOwner) public {
+ function changeOwner(address _newOwner) public {
+     require(msg.sender == owner, "Caller is not the owner");
      owner = _newOwner;
  }

```

-----------------------------------------------------------------------

## High-2

Reentrancy Vulnerability in claimSingleReward() Function


## Summary

The `claimSingleReward()` function is susceptible to reentrancy attacks, which could allow an attacker to call this function recursively via a fallback function, potentially draining funds or receiving more rewards than intended.

## Vulnerability Details

The implementation does not follow the check-effects-interactions pattern. An attacker could exploit this by creating a malicious contract with a fallback function that calls `claimSingleReward()`, leading to a reentrancy attack and potentially draining the contract.

* **Vulnerable Code Snippet**

```Solidity
function claimSingleReward(uint256 _index) public {
    require(_index <= rewardsOwned[msg.sender].length, "Invalid index");
    uint256 value = rewardsOwned[msg.sender][_index].value;
    require(value > 0, "No reward to claim");

    (bool success,) = payable(msg.sender).call{value: value}("");
    require(success, "Transfer failed");

    delete rewardsOwned[msg.sender][_index];
}

```

## Impact

An attacker could exploit the vulnerability in the contract's reward distribution mechanism through a **reentrancy attack**, allowing them to repeatedly call the `claimSingleReward() ` function before the original execution is complete. This would allow the attacker to drain the contract’s funds by claiming the same reward multiple times, or even claim rewards they are not entitled to, leading to a significant loss of funds allocated for legitimate users.

In this scenario, **the attacker could manipulate the contract** to withdraw more rewards than they are supposed to receive. Since the contract does not properly secure the state change (deleting the reward or marking the reward as claimed) before executing the external call (the transfer of funds), the attacker could exploit this gap. By repeatedly calling the function in a recursive manner, they could claim rewards multiple times, even if they are only entitled to claim once.

This vulnerability exposes **all users interacting with the contract** to a severe risk. Any user attempting to claim their rightful rewards could find that the funds have already been drained by the attacker. As a result, legitimate claimants could suffer **financial losses**, as there would be insufficient funds left in the contract to honor their claims. Moreover, the **contract owner is also at risk** of losing a significant amount of capital, potentially leading to the collapse of the entire reward system and a loss of trust in the platform.

## Tools Used

manual review

## Recommendations

* Implement the check-effects-interactions pattern by moving the state changes (deleting rewards) before the external call to prevent reentrancy.

       As I have shown below: 

```diff
function claimSingleReward(uint256 _index) public {
    
    //check
    require(_index < rewardsOwned[msg.sender].length, "Invalid index"); // Corrected condition
    uint256 value = rewardsOwned[msg.sender][_index].value;
    require(value > 0, "No reward to claim");
   
    //effect
+   delete rewardsOwned[msg.sender][_index]; // Move this line before the external call

    //interaction
    (bool success,) = payable(msg.sender).call{value: value}("");
    require(success, "Transfer failed");

-   delete rewardsOwned[msg.sender][_index];
}

```

 

* consider using OpenZeppelin's **`ReentrancyGuard`** modifier is essential to secure the contract against reentrancy exploits and maintain the integrity of the reward distribution process.


-----------------------------------------------------------------------

## High-3

Reentrancy Vulnerability in claimAllRewards()


## Summary

The `claimAllRewards()` function is vulnerable to a reentrancy attack. This occurs because external calls are made before the contract state is updated, allowing an attacker to repeatedly call the function in a fallback, potentially draining the contract of its funds.

## Vulnerability Details

The function transfers funds to the user by calling `payable(msg.sender).call{value: totalValue}("");` before the state variable `rewardsOwned` is updated. This gives an attacker the opportunity to exploit the contract by invoking a fallback function and calling `claimAllRewards()` again before the deletion of rewards takes place.

&#x20;An attacker can use this vulnerability to repeatedly call the `claimAllRewards()` function in their fallback function, draining all funds from the contract or claiming more rewards than they are entitled to.

&#x20;The function does not follow the **Check-Effects-Interactions** pattern, which recommends updating the contract’s state before making external calls

* **Vulnerable Code Snippet**

```Solidity
function claimAllRewards() public {
        uint256 totalValue = 0;
        for (uint256 i = 0; i < rewardsOwned[msg.sender].length; i++) {
            totalValue += rewardsOwned[msg.sender][i].value;
        }
        require(totalValue > 0, "No rewards to claim");

        (bool success,) = payable(msg.sender).call{value: totalValue}("");
        require(success, "Transfer failed");

        delete rewardsOwned[msg.sender];
    }
```

## Impact

An attacker could exploit this vulnerability to drain all funds from the contract, claim multiple rewards, or cause a severe financial loss. This could lead to significant damage to the integrity of the contract and the trust of users interacting with it.

All users of the contract are at risk since an attacker could abuse this reentrancy vulnerability and manipulate reward claims, rendering the contract unusable or depleting its funds.

## Tools Used

Manual Review

## Recommendations

Consider using the Check-Effects-Interactions pattern by moving the deletion of the `rewardsOwned[msg.sender]` state variable to occur before the external call to the `msg.sender`. This ensures the contract’s state is updated before any interactions with external addresses take place.

As I have shown here : 

```diff
function claimAllRewards() public {
    uint256 totalValue = 0;
    for (uint256 i = 0; i < rewardsOwned[msg.sender].length; i++) {
        totalValue += rewardsOwned[msg.sender][i].value;
    }
    require(totalValue > 0, "No rewards to claim");

+  // Update state before making the external call
+  delete rewardsOwned[msg.sender];

    (bool success,) = payable(msg.sender).call{value: totalValue}("");
    require(success, "Transfer failed");

-   delete rewardsOwned[msg.sender]; // Moved this line above the external call
}
```

-----------------------------------------------------------------------

## Medium-1

Modulo Bias in Random Number Generation in openBox() Function


## Summary

The `openBox()` function generates a random value to determine rewards based on a probability distribution. However, the current implementation could lead to modulo bias in the random number generation, affecting the fairness of reward distribution.

## Vulnerability Details

The random value is generated using `keccak256(abi.encodePacked(block.timestamp, msg.sender)) % 100`. This approach can introduce bias because if the output of `keccak256` is not uniformly distributed, the modulo operation will not yield an even distribution of numbers between 0 and 99.

* **Vulnerable Code Snippet** 

  ```Solidity
  uint256 randomValue = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender))) % 100;
  ```

## Impact

All users interacting with the `openBox()` function who expect a fair chance of receiving rewards could be impacted by the uneven distribution.

Users may experience an unfair distribution of rewards due to modulo bias, leading to some rewards being less likely to be awarded than others. This could decrease user trust in the reward system and reduce engagement.

## Tools Used

manual review

## Recommendations

Consider using a more robust method for generating random numbers.\
\
&#x20;A common approach is to utilize an external oracle service (like Chainlink VRF) for true randomness, which eliminates bias

-----------------------------------------------------------------------

## Medium-2

Newly Added Rewards Not Reflected for Users


## Summary

The contract has no mechanism to ensure that newly added rewards are reflected in future reward allocations when users open mystery boxes. As a result, users may not have a chance to receive any new rewards added after their interaction begins.

## Vulnerability Details

When new rewards are added to the `rewardPool` via the `addReward` function, the current logic for the `openBox` function uses a hardcoded probability system that does not account for changes in the `rewardPool`. The randomization logic and reward allocation seem to be fixed, so users opening boxes are not given an updated chance to win any newly added rewards. This could lead to a scenario where users, even after the addition of new rewards, are still only winning old rewards, making the newly added rewards effectively inaccessible to existing users.

## Impact

Users might expect new rewards after interacting with the contract, but the static logic prevents them from obtaining any of these newly added rewards.

Additionally, the contract owner would need to redeploy or modify the contract frequently to ensure new rewards are fairly reflected, increasing operational overhead

While this does not directly affect the security of the contract, it significantly impacts the fairness of reward distribution. Users may feel misled if new rewards are added but they are unable to win them, leading to a potential loss of trust in the platform.

## Tools Used

Manual Review

## Recommendations

Implement dynamic reward allocation logic that adjusts based on the current state of the `rewardPool`.

&#x20;Ensure that when new rewards are added, they are incorporated into the randomization logic used in the `openBox` function.

-----------------------------------------------------------------------

## Low-1 

Gas Limit and Denial of Service (DoS) Vulnerability

## Summary

The `claimAllRewards` function may lead to a Denial of Service (DoS) if the number of rewards owned by a user is excessively large. This can cause gas consumption to exceed the block gas limit, preventing users from successfully claiming their rewards.

## Vulnerability Details

The function iterates through `rewardsOwned[msg.sender]` to calculate the total value of rewards. If a user has a large number of rewards, the iteration may consume excessive gas, potentially causing the transaction to fail.

* **Vulnerable Code Snippet**

  ```Solidity
  function claimAllRewards() public {
          uint256 totalValue = 0;
          for (uint256 i = 0; i < rewardsOwned[msg.sender].length; i++) {
              totalValue += rewardsOwned[msg.sender][i].value;
          }
          require(totalValue > 0, "No rewards to claim");

          (bool success,) = payable(msg.sender).call{value: totalValue}("");
          require(success, "Transfer failed");

          delete rewardsOwned[msg.sender];
      }
  ```

## Impact

Users who invest in purchasing boxes for a chance to win rewards could find themselves unable to claim their earnings due to this vulnerability, leading to significant frustration and a loss of faith in the contract's reliability. \
\
Since rewards can only be obtained through these purchases and a stroke of luck, malicious actors could exploit this flaw by artificially inflating users' reward lists, effectively locking them out of their hard-earned prizes. This means that all users—especially those who have invested time and resources into buying boxes—face the risk of encountering this Denial of Service (DoS) vulnerability.\
\
&#x20;If left unaddressed, this issue could severely damage the user experience and erode confidence in the platform's integrity

## Tools Used

manual review

## Recommendations

Implement a maximum limit on the number of rewards that can be claimed in a single transaction to prevent excessive gas usage and ensure smoother interactions for all users. This change not only protects the network from potential Denial of Service (DoS) attacks but also enhances the overall user experience by making reward claims more manageable


