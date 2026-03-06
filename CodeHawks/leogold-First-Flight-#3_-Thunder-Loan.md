# First Flight #3: Thunder Loan - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Flashloan repayment can be exploited by attacker through deposit](#H-01)
    - ### [H-02. Deposit function updating exchange rate will cause a malicious depositors to keep redeeming](#H-02)
    - ### [H-03. Storage Collision during upgrade](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Redeeming underlying assets linked to a deleted AssetToken will not be possible for users.](#M-01)
    - ### [M-02. executeOperation return status not checked,](#M-02)
- ## Low Risk Findings
    - ### [L-01. CEI not followed in deposit](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #3

### Dates: Nov 1st, 2023 - Nov 8th, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-11-Thunder-Loan)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 2
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Flashloan repayment can be exploited by attacker through deposit            



## Summary
the ThunderLoan contract and the upgradeThunderLoan contract  is vulnerable to a flashloan deposit attack

## Vulnerability Details
An attacker can take out a flashloan and deposit it into the contract without actually paying it back. The attacker can then withdraw the flashloan deposit.

## Impact
An attacker can use this to drain all funds in the contract by taking as many flashloans as possible.

## POC
https://gist.github.com/Falilah/470de5a2c297f8f775b7b24a51a28cbb

## Tools Used
manual review, foundry

## Recommendations
To fix this, the team should update the deposit function to check that the token is not currently flashloaning by using a modifier or use the `s_currentlyFlashLoaning[token]` variable to check the status of the token user want to deposit at every point in time.
suggested code to fix is as follows:
   Add another error message     `error ThunderLoan__CurrentlyFlashLoaning();` and check in the deposit function to be sure token is not currently flashloaning before user can deposit i.e 
```
`if (s_currentlyFlashLoaning[token]) {revert ThunderLoan__CurrentlyFlashLoaning();}
```.
Add the above recommendation to the deposit function and see the poc failing with `ThunderLoan__CurrentlyFlashLoaning` error message
## <a id='H-02'></a>H-02. Deposit function updating exchange rate will cause a malicious depositors to keep redeeming            



## Summary
Deposit function updating exchange rate will cause a malicious depositors to keep redeeming with interest rate with or without any flashloan occuring within the period
## Vulnerability Details
  getCalculatedFee function is for knowing the current fee for user coming to take flashloan in the contract, which means the fee is only for users who flashloaned to know how much fee they will pay based on the borrowed amount, the fee paid is eventually used to update the exchange rate for liquidity provider to earn interest but unfortunately the deposit function called getCalculateFee and updateExchange rate on the amount users deposited which will affect the earnings of all users who deposited into the thunderloan contract negatively. 

## Impact
liquidity providers lose part or they can lose all of their funds to other liquidity providers with or without flashloan occuring, 
it can cause DOS for other liquidity provider to redeem their asset as the contract does not have the expected payout amount for liquidity provider.
## Tools Used
manual review, foundry
## Recommendations
getCalculateFee and updateExchange rate should be removed from the deposit function as it do not depict the intention of the developer. As stated in the docs that liquidity provider earns interest overtime depending on how much flashloan occurred, which means no user should be able to earn interest within the period of deposit if no flashloan occured
NOTE: IT IS FIXED in the thunderloanUpgraded version, the team should not waste time upgrading the contract as attackers are watching every seconds to attack any honey pot unchain.
## <a id='H-03'></a>H-03. Storage Collision during upgrade            



## Summary
The thunderloanupgrade.sol storage layout is not compatible with the storage layout of thunderloan.sol which will cause storage collision and mismatch of variable to different data.
## Vulnerability Details
Thunderloan.sol at slot 1,2 and 3 holds s_feePrecision, s_flashLoanFee and s_currentlyFlashLoaning, respectively, but the ThunderLoanUpgraded at slot 1 and 2 holds s_flashLoanFee, s_currentlyFlashLoaning respectively. the s_feePrecision from the thunderloan.sol was changed to a constant variable which will no longer be assessed from the state variable. This will cause the location at which the upgraded version will be pointing to for some significant state variables like s_flashLoanFee  to be wrong because s_flashLoanFee is now pointing to the slot of the s_feePrecision in the thunderloan.sol and when this fee is used to compute the fee for flashloan it will return a fee amount greater than the intention of the developer.
 s_currentlyFlashLoaning might not really be affected as it is back to default when a flashloan is completed but still to be noted that the value at that slot can be cleared to be on a safer side. 
## Impact
1) Fee is miscalculated for flashloan
1)  users pay same amount of what they borrowed as fee
## Tools Used
foundry
##POC 
```
import {ThunderLoanUpgraded} from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";

 function upgradeThunderloan() internal {
        thunderLoanUpgraded = new ThunderLoanUpgraded();
        thunderLoan.upgradeTo(address(thunderLoanUpgraded));
        thunderLoanUpgraded = ThunderLoanUpgraded(address(proxy));
    }

 function testSlotValuesBeforeAndAfterUpgrade() public setAllowedToken {
        AssetToken asset = thunderLoan.getAssetFromToken(tokenA);
        uint precision = thunderLoan.getFeePrecision();
        uint fee = thunderLoan.getFee();
        bool isflanshloaning = thunderLoan.isCurrentlyFlashLoaning(tokenA);
        /// 4 slots before upgrade
        console.log("????SLOTS VALUE BEFORE UPGRADE????");
        console.log("slot 0 for s_tokenToAssetToken =>", address(asset));
        console.log("slot 1 for s_feePrecision =>", precision);
        console.log("slot 2 for s_flashLoanFee =>", fee);
        console.log("slot 3 for s_currentlyFlashLoaning =>", isflanshloaning);
        //upgrade function
        upgradeThunderloan();

        //// after upgrade they are only 3 valid slot left because precision is now set to constant
        AssetToken assetUpgrade = thunderLoan.getAssetFromToken(tokenA);
        uint feeUpgrade = thunderLoan.getFee();
        bool isflanshloaningUpgrade = thunderLoan.isCurrentlyFlashLoaning(
            tokenA
        );

        console.log("????SLOTS VALUE After UPGRADE????");
        console.log("slot 0 for s_tokenToAssetToken =>", address(assetUpgrade));
        console.log("slot 1 for s_flashLoanFee =>", feeUpgrade);
        console.log(
            "slot 2 for s_currentlyFlashLoaning =>",
            isflanshloaningUpgrade
        );
        assertEq(address(asset), address(assetUpgrade));
        //asserting precision value before upgrade to be what fee takes after upgrades
        assertEq(precision, feeUpgrade); // #POC
        assertEq(isflanshloaning, isflanshloaningUpgrade);
    }

```
Add the code  above to thunderloantest.t.sol and run with `forge test --mt testSlotValuesBeforeAndAfterUpgrade -vv`
## POC 2
```  
function testFlashLoanAfterUpgrade() public setAllowedToken hasDeposits {
        //upgrade thunderloan
        upgradeThunderloan();

        uint256 amountToBorrow = AMOUNT * 10;
        console.log("amount flashloaned", amountToBorrow);
        uint256 calculatedFee = thunderLoan.getCalculatedFee(
            tokenA,
            amountToBorrow
        );
        AssetToken assetToken = thunderLoan.getAssetFromToken(tokenA);

        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver), amountToBorrow);
        thunderLoan.flashloan(
            address(mockFlashLoanReceiver),
            tokenA,
            amountToBorrow,
            ""
        );
        vm.stopPrank();

        console.log("feepaid", calculatedFee);
        assertEq(amountToBorrow, calculatedFee);
    }

```
Add the code above to thunderloantest.t.sol and run `forge test --mt testFlashLoanAfterUpgrade -vv` to test for the second poc 
## Recommendations
The team should should make sure the the fee is pointing to the correct location as intended by the developer: a suggestion recommendation is for the team to get the feeValue from the previous implementation, clear the values that will not be needed again and after upgrade reset the fee back to its previous value from the implementation.
##POC for recommendation
```
//
 function upgradeThunderloanFixed() internal {
        thunderLoanUpgraded = new ThunderLoanUpgraded();
        //getting the current fee;
        uint fee = thunderLoan.getFee();
        // clear the fee as
        thunderLoan.updateFlashLoanFee(0);
        // upgrade to the new implementation
        thunderLoan.upgradeTo(address(thunderLoanUpgraded));
        //wrapped the abi
        thunderLoanUpgraded = ThunderLoanUpgraded(address(proxy));
        // set the fee back to the correct value
        thunderLoanUpgraded.updateFlashLoanFee(fee);
    }



function testSlotValuesFixedfterUpgrade() public setAllowedToken {
        AssetToken asset = thunderLoan.getAssetFromToken(tokenA);
        uint precision = thunderLoan.getFeePrecision();
        uint fee = thunderLoan.getFee();
        bool isflanshloaning = thunderLoan.isCurrentlyFlashLoaning(tokenA);
        /// 4 slots before upgrade
        console.log("????SLOTS VALUE BEFORE UPGRADE????");
        console.log("slot 0 for s_tokenToAssetToken =>", address(asset));
        console.log("slot 1 for s_feePrecision =>", precision);
        console.log("slot 2 for s_flashLoanFee =>", fee);
        console.log("slot 3 for s_currentlyFlashLoaning =>", isflanshloaning);
        //upgrade function
        upgradeThunderloanFixed();

        //// after upgrade they are only 3 valid slot left because precision is now set to constant
        AssetToken assetUpgrade = thunderLoan.getAssetFromToken(tokenA);
        uint feeUpgrade = thunderLoan.getFee();
        bool isflanshloaningUpgrade = thunderLoan.isCurrentlyFlashLoaning(
            tokenA
        );

        console.log("????SLOTS VALUE After UPGRADE????");
        console.log("slot 0 for s_tokenToAssetToken =>", address(assetUpgrade));
        console.log("slot 1 for s_flashLoanFee =>", feeUpgrade);
        console.log(
            "slot 2 for s_currentlyFlashLoaning =>",
            isflanshloaningUpgrade
        );
        assertEq(address(asset), address(assetUpgrade));
        //asserting precision value before upgrade to be what fee takes after upgrades
        assertEq(fee, feeUpgrade); // #POC
        assertEq(isflanshloaning, isflanshloaningUpgrade);
    }

```
Add the code above to thunderloantest.t.sol and run with `forge test --mt testSlotValuesFixedfterUpgrade -vv`.
it can also be tested with `testFlashLoanAfterUpgrade function` and see the fee properly calculated for flashloan

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Redeeming underlying assets linked to a deleted AssetToken will not be possible for users.            



## Summary
Impossible Redeeming an underlying asset for token that was previously allowed to be deposited but is not allowed again.

## Vulnerability Details
`setAllowedToken function` set the status of a token to either true or false. setting a token in which a liquidity provider have deposited into the thunderloan contract back to false, deleting all information associated to that token including the AssetToken Liquidity provider hold as a receipt to redeem their asset will render the AssetToken useless as the underlying Token associated with the AssetToken is no longer a valid token in the contract and in turn will get liquidity provider funds stuck in the contract forever. 
## Impact
1) Liquidity Provider funds get stucked in the contract forever
## POC
Add the function below to thunderloan.t.sol
 ```solidity
 function testFailsetAllowedTokenToFalse()
        public
        setAllowedToken
        hasDeposits
    {
        AssetToken assetToken = thunderLoan.getAssetFromToken(tokenA);

        vm.startPrank(user);
        tokenA.mint(address(user), AMOUNT);
        tokenA.approve(address(thunderLoan), AMOUNT);

        thunderLoan.deposit(tokenA, AMOUNT);
        vm.stopPrank();

        // set deposited Token back to false
        vm.prank(thunderLoan.owner());
        thunderLoan.setAllowedToken(tokenA, false);

        // user try to Redeem UnderlyingToken and got denied
        vm.prank(user);
        thunderLoan.redeem(tokenA, assetToken.balanceOf(user));
    }
```
run with `forge test --mt testFailsetAllowedTokenToFalse -vvvv`

## Tools Used
foundry
## Recommendations
The team should make sure Liquidity Provider who have deposited into the contract when it accept the token should still be able to withdraw their token when it does not accept it again to avoid stucking funds in the contract.
They can also check the AssetToken contract attached to that underlying token is not holding any amount of the underlying Token before removing it from allowedToken. 
## <a id='M-02'></a>M-02. executeOperation return status not checked,            



## Summary

## Vulnerability Details
The execueOperation function called in the flashloanreceiver was not checked to confirm transaction actually went successful before proceeding.
## Impact
It is possible for the Operation done with flashloan taken from thunderloan to fail without having a knowledge until the end of transaction which could lead to fundLoss.
## Tools Used
Manual review.
## Recommendations
functionCall returned bytes which can be decoded to get the status of the transaction, it recommended for the team to look into the return data and confirm the status of the transaction

# Low Risk Findings

## <a id='L-01'></a>L-01. CEI not followed in deposit            



## Summary

## Vulnerability Details
The check-effect interaction was not properly followed in the deposit function as state variable were updated before getting the fund from users which is a bad practice as it can be dangerous to the protocol and users.
## Impact
Liquidity Provider tends to waste gas before realizing they don't have enough or haven't approved the protocol to spend the funds
## Tools Used
Manual review
## Recommendations
The team should confirm a transferfrom function was successful from the liquidity provider before updating the state of the contract.


