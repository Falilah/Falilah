# First Flight #6: Voting Booth - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect Reward Distribution due to Denomination Error](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #6

### Dates: Dec 15th, 2023 - Dec 22nd, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-12-Voting-Booth)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect Reward Distribution due to Denomination Error            



## Summary
 reward calculation error in VotingBooth smart contract caused by a Denomination Error
## Vulnerability Details
In the VotingBooth::_distributeRewards function, if the proposal passed, the rewardPerVoter calculation employed an invalid denominator, specifically using totalVotes to distribute rewards among the For voters. This resulted in unequal distribution, as the denominator considered users who voted Against the proposal as well. Consequently, this miscalculation has financial implications on both the smart contract and its users. The division of totalRewards by totalVotes caused the rewardPerVoter value to be inaccurately calculated.
## Impact
- Wrong distribution of rewards for the `For` Voters
-  funds get stuck forever in the smart contracts
## Tools Used
Manual Review, Foundry.

## POC

```
    function testIncorrectRewardCalculation() public {
        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(false);

        vm.prank(address(0x3));
        booth.vote(true);

        //two `For` Voters making the proposal pass means each voter will get 5 ethers each of the 10 ethers in the booth

        // shows the FOR voter did not get all the reward out of the Booth
        uint totalVotersBalance = address(0x1).balance + address(0x3).balance;
        assert(totalVotersBalance < 10 ether);

        // shows there are still funds stuck in the booth
        assert(!booth.isActive() && address(booth).balance > 0);
    }
```
Add the above function to VotingBoothTest.t.sol and run it with `forge test --mt testIncorrectRewardCalculation -vvvvv`

## Recommendations
it is advised to correct the calculation logic in the VotingBooth::_distributeRewards function to use an appropriate denominator, ensuring that only relevant votes are considered in the rewardPerVoter calculation when proposal passed.
```diff
-                uint256 rewardPerVoter = totalRewards / totalVotes;
+               uint256 rewardPerVoter = totalRewards / totalVotesFor;

-                rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
+                rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotesFor, Math.Rounding.Ceil);

```

    





