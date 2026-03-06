# First Flight #2: Puppy Raffle - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Reentrancy: ](#H-01)
    - ### [H-02. possible Invalid Array length in selectWinner() function](#H-02)
    - ### [H-03. possible Overflow variable](#H-03)
    - ### [H-04. possible transfer of fee to address zero](#H-04)
    - ### [H-05. Insecure Randomness](#H-05)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #2

### Dates: Oct 25th, 2023 - Nov 1st, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-10-Puppy-Raffle)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 5
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Reentrancy:             



## Summary
Reenttrancy in refund can allow a malicious player to steal all fund in the contract.
## Vulnerability Details
 the refund() function is vulnerable to reentrancy attacks. This means that a malicious player can call the function multiple times before the state is updated, which allows them to steal funds from the contract
## Impact
To perform a reentrancy attack, a malicious player would first call the refund() function. This would cause the contract to send some funds to the player. The player would then immediately call the refund() function again. This process would be repeated until all of the funds in the contract have been drained.

The vulnerability is particularly dangerous because state updates are performed after the external call using the sendValue() function. This means that the attacker's balance of funds is not updated until after the contract has sent funds to them. This allows the attacker to call the refund() function multiple times before the contract realizes that they have already been refunded.
## POC
link https://gist.github.com/Falilah/14be4c04945b35d81bb038789492439c
## Tools Used
Manual review, Foundry
## Recommendations
To prevent this type of attack, The teams should always use reentrancy guards. A reentrancy guard is a piece of code that prevents a function from being called multiple times before the state is updated.
## <a id='H-02'></a>H-02. possible Invalid Array length in selectWinner() function            



## Summary
Invalid array length when users got refunded during the duration of the game.
## Vulnerability Details
The refund() function only sets the index of the user refunded back to address 0 but still retains the length of the array
## Impact
Invalid total amount collected: Calculating the total amount collected with an invalid length can lead to an invalid amount. This is because the total amount collected is calculated by multiplying the number of the length players by the price of the raffle ticket. If the number of active players is less than the length of the array, then the total amount collected will be overstated. This could lead to problems such as users being able to claim more money than they are entitled to.
## Tools Used
manual review: 
## Recommendations
The refund() function should be modified to set the index of the user refunded to address 0 and to decrement the length of the array. This will ensure that the selectWinner() function always uses the correct number of active players to determine the winning index. Additionally, the total amount collected should be calculated using the number of active players, not the length of the array.
## <a id='H-03'></a>H-03. possible Overflow variable            



## Summary
TotalFees accumulation can overflow

## Vulnerability Details
The contract has the possibility of being compiled with a version less than 0.8.0, and versions less than 0.8.0 are not overflow/underflow protected. Additionally, the variable uint64 Totalfees can only take in a maximum of uint64, and it is possible for the total fee accumulated to be more than the maximum which is 18446744073709551615
(more than 18 ether ). If this is the case, the total fee will overflow, and the owner will not be able to withdraw the valid fee that has been accumulated from all rounds of the game

## Impact
1) The owner of the contract may not be able to withdraw the valid fee that has been accumulated.
2) Users may lose money if the contract overflows due to excess withdraw by owner.

## Tools Used
manual review

resources OWASP: https://owasp.org/www-project-smart-contract-top-10/2023/en/src/SC02-integer-overflow-underflow.html
## Recommendations
1) The contract should be compiled with version 0.8.0 or higher.
2) The variable uint64 Totalfees should be changed to a larger type, such as uint128 or uint256.
## <a id='H-04'></a>H-04. possible transfer of fee to address zero            



## Summary

## Vulnerability Details
The changeFeeAddress() function lacks a check for address zero. This means that an owner could call the function to change the fee address to address zero by mistake, and all fees would then be sent to address zero. This would effectively drain the contract of all of its fees to a null address.


## Impact
The contract could be drained of all of its fee funds to address zero.

Users could lose money if they paying fees to the contract.


## POC 
https://gist.github.com/Falilah/c77222f98a8a7c656bfa974e508e7211

## Tools Used
manual review, Foundry 
## Recommendations
The changeFeeAddress() function should be modified to check if the new fee address is address zero. If it is, the function should revert.


## <a id='H-05'></a>H-05. Insecure Randomness            



## Summary

## Vulnerability Details
Generating randomness in Ethereum is challenging because every node must come to the same conclusion on the state of the blockchain. Hence, naive approaches to generate randomness can be manipulated by validators or observant attackers. This can lead to unfair advantages in the game.


## Impact
Insecure randomness can be exploited by attackers to gain an unfair advantage in PuppyRaffle draw because it rely on random number generation using block.timestamp which can be manipulated by validators.


## Tools Used
Manual Review
## Recommendations
Use external oracle services that provide random numbers.
    





