## Title

`Critical`: Dividend Distribution Limited to Admin Role Holders Only

## Severity

Critical

## Classification

Business Logic Error
Access Control Issue
Financial Impact

## Affected Contract : 

https://github.com/afrail-inc/afrx-security-token/blob/main/AFRXToken.sol

## Description

The current implementation of the distributeDividends function in the AFRXToken contract incorrectly distributes dividends only to addresses with `DEFAULT_ADMIN_ROLE`, instead of all token holders. This violates the stated business requirement that "every investor, regardless of their role, will receive dividends proportionally based on the number of AFRX tokens they own."

## Vulnerable Code

```solidity function distributeDividends(uint256 totalAmount) external onlyRole(ADMIN_ROLE) nonReentrant whenNotPaused {
    require(totalSupply() > 0, "No tokens in circulation");
    _mint(address(this), totalAmount);
    for (uint256 i = 0; i < getRoleMemberCount(DEFAULT_ADMIN_ROLE); i++) {  // <- Vulnerability
        address investor = getRoleMember(DEFAULT_ADMIN_ROLE, i);
        if (balanceOf(investor) > 0) {
            uint256 share = (balanceOf(investor) * totalAmount) / totalSupply();
            _transfer(address(this), investor, share);
        }
    }
    emit DividendsDistributed(totalAmount);
}
```

## Impact

The vulnerability results in:

- Regular token holders receive no dividends
- Only admin role holders receive dividend distributions
- 45% or more of dividends are incorrectly distributed
- Violation of stated dividend policy and potential regulatory requirements
- Possible breach of securities laws and investor rights

# Proof of Concept
```solidity
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/AFRXToken.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract DividendDistributionPOCTest is Test {
    AFRXToken public token;
    address public admin;
    address public regularInvestor1;
    address public regularInvestor2;
    address public adminInvestor;
    
   function setUp() public {
    // Deploy token
    token = new AFRXToken();
    token.initialize(address(this));
    
    // Setup test addresses
    admin = address(this);
    regularInvestor1 = makeAddr("investor1");
    regularInvestor2 = makeAddr("investor2");
    adminInvestor = makeAddr("adminInvestor");
    
    // Setup jurisdiction
    token.addJurisdiction("US");
    
    // Setup roles
    token.grantRole(token.MINTER_ROLE(), admin);
    token.grantRole(token.DEFAULT_ADMIN_ROLE(), adminInvestor);
    
    // Whitelist investors AND the token contract itself
    token.whitelistInvestor(regularInvestor1, "US");
    token.whitelistInvestor(regularInvestor2, "US");
    token.whitelistInvestor(adminInvestor, "US");
    token.whitelistInvestor(address(token), "US");  // Add this line
    
    // Initial token distribution
    token.issueTokens(regularInvestor1, 500 ether, "US");
    token.issueTokens(regularInvestor2, 400 ether, "US");
    token.issueTokens(adminInvestor, 100 ether, "US");
}

    function testDividendDistributionInequality() public {
        // Initial balances
        console.log("=== Initial State ===");
        console.log("Regular Investor 1 balance:", token.balanceOf(regularInvestor1) / 1e18, "tokens (50%)");
        console.log("Regular Investor 2 balance:", token.balanceOf(regularInvestor2) / 1e18, "tokens (40%)");
        console.log("Admin Investor balance:", token.balanceOf(adminInvestor) / 1e18, "tokens (10%)");
        console.log("Total Supply:", token.totalSupply() / 1e18);
        
        // Distribute 1000 tokens as dividends
        uint256 dividendAmount = 1000 ether;
        console.log("\n=== Distributing", dividendAmount / 1e18, "tokens as dividends ===");
        
        // Record balances before distribution
        uint256 reg1BalanceBefore = token.balanceOf(regularInvestor1);
        uint256 reg2BalanceBefore = token.balanceOf(regularInvestor2);
        uint256 adminBalanceBefore = token.balanceOf(adminInvestor);
        
        // Distribute dividends
        token.distributeDividends(dividendAmount);
        
        // Calculate actual distributions
        uint256 reg1Received = token.balanceOf(regularInvestor1) - reg1BalanceBefore;
        uint256 reg2Received = token.balanceOf(regularInvestor2) - reg2BalanceBefore;
        uint256 adminReceived = token.balanceOf(adminInvestor) - adminBalanceBefore;
        
        console.log("\n=== Actual Distribution ===");
        console.log("Regular Investor 1 received:", reg1Received / 1e18, "tokens (Expected: 500)");
        console.log("Regular Investor 2 received:", reg2Received / 1e18, "tokens (Expected: 400)");
        console.log("Admin Investor received:", adminReceived / 1e18, "tokens (Expected: 100)");
        
        // Calculate what they should have received
        uint256 expectedReg1 = (token.balanceOf(regularInvestor1) * dividendAmount) / token.totalSupply();
        uint256 expectedReg2 = (token.balanceOf(regularInvestor2) * dividendAmount) / token.totalSupply();
        uint256 expectedAdmin = (token.balanceOf(adminInvestor) * dividendAmount) / token.totalSupply();
        
        console.log("\n=== Expected vs Actual ===");
        console.log("Regular Investor 1: Expected", expectedReg1 / 1e18, "but got", reg1Received / 1e18);
        console.log("Regular Investor 2: Expected", expectedReg2 / 1e18, "but got", reg2Received / 1e18);
        console.log("Admin Investor: Expected", expectedAdmin / 1e18, "but got", adminReceived / 1e18);
        
        // Assert the inequality
        assertTrue(reg1Received == 0, "Regular investor 1 shouldn't receive anything but should");
        assertTrue(reg2Received == 0, "Regular investor 2 shouldn't receive anything but should");
        assertTrue(adminReceived > 0, "Admin investor shouldn't receive everything");
        
        console.log("\n=== Impact Analysis ===");
        console.log("Total tokens not distributed to eligible holders:", 
            (expectedReg1 + expectedReg2) / 1e18);
        console.log("Percentage of dividends wrongly distributed:", 
            ((expectedReg1 + expectedReg2) * 100) / dividendAmount, "%");
    }
}

```


## Test Result :

```solidity Ran 1 test for test/AFRXToken.t.sol:DividendDistributionPOCTest
[PASS] testDividendDistributionInequality() (gas: 139793)
Logs:
  === Initial State ===
  Regular Investor 1 balance: 500 tokens (50%)
  Regular Investor 2 balance: 400 tokens (40%)
  Admin Investor balance: 100 tokens (10%)
  Total Supply: 1000
  
=== Distributing 1000 tokens as dividends ===
  
=== Actual Distribution ===
  Regular Investor 1 received: 0 tokens (Expected: 500)
  Regular Investor 2 received: 0 tokens (Expected: 400)
  Admin Investor received: 50 tokens (Expected: 100)
  
=== Expected vs Actual ===
  Regular Investor 1: Expected 250 but got 0
  Regular Investor 2: Expected 200 but got 0
  Admin Investor: Expected 75 but got 50
  
=== Impact Analysis ===
  Total tokens not distributed to eligible holders: 450
  Percentage of dividends wrongly distributed: 45 %

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.70ms (2.76ms CPU time)
```


## Key Impact Metrics (from PoC):

- 90% of token holders receive zero dividends
- 45% of dividend tokens are incorrectly distributed
- Only DEFAULT_ADMIN_ROLE holders

## Recommended Fix
``` solidity
contract AFRXToken {
    struct DividendInfo {
        uint256 totalAmount;
        uint256 snapshotId;
        uint256 totalSupplyAtSnapshot;
    }
    
    mapping(uint256 => DividendInfo) public dividendInfo;
    mapping(uint256 => mapping(address => bool)) public dividendsClaimed;
    uint256 public currentDividendRound;

    function distributeDividends(uint256 totalAmount) external onlyRole(ADMIN_ROLE) nonReentrant whenNotPaused {
        require(totalSupply() > 0, "No tokens in circulation");
        
        uint256 snapshotId = _snapshot();
        uint256 snapshotSupply = totalSupply();
        
        _mint(address(this), totalAmount);
        
        dividendInfo[currentDividendRound] = DividendInfo({
            totalAmount: totalAmount,
            snapshotId: snapshotId,
            totalSupplyAtSnapshot: snapshotSupply
        });
        
        emit DividendsAvailable(currentDividendRound, totalAmount);
        currentDividendRound++;
    }

    function claimDividends(uint256 roundId) external nonReentrant whenNotPaused {
        require(!dividendsClaimed[roundId][msg.sender], "Already claimed");
        require(_investors[msg.sender].isWhitelisted, "Not whitelisted");
        
        DividendInfo storage dividend = dividendInfo[roundId];
        uint256 balance = balanceOfAt(msg.sender, dividend.snapshotId);
        require(balance > 0, "No tokens at snapshot");
        
        uint256 share = (balance * dividend.totalAmount) / dividend.totalSupplyAtSnapshot;
        require(share > 0, "No dividends to claim");
        
        dividendsClaimed[roundId][msg.sender] = true;
        _transfer(address(this), msg.sender, share);
        
        emit DividendsClaimed(roundId, msg.sender, share);
    }
}
```

## Mitigation

The fix implements a snapshot-based dividend system where:

- All token holders can claim their fair share
- Distribution is based on token holdings at snapshot time
- Maintains regulatory compliance requirements
- Preserves investor rights and fairness

## References

Project documentation stating dividend policy
OpenZeppelin's ERC20Snapshot implementation
Security token dividend distribution best practices

## Timeline

Found: [5/16/2025]
Reported: [5/16/2025]