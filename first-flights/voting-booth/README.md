# Findings

### [H-1] `VotingBooth::_distributeRewards` to "for" voters leaves ETH leftover and locked in `VotingBooth` contract. 

**Description:** When a proposal gets passed, the rewards are calculated using the `totalVotes` instead of `totalVotesFor`, not distributing the full reward amount to the `s_votersFor` and leaving leftover funds in the `VotingBooth` contract.

```js
    // distributes rewards to the `for` voters if the proposal has
    // passed or refunds the rewards back to the creator if the proposal
    // failed
    function _distributeRewards() private {
        uint256 totalVotesFor = s_votersFor.length;
        uint256 totalVotesAgainst = s_votersAgainst.length;
        uint256 totalVotes = totalVotesFor + totalVotesAgainst;
        uint256 totalRewards = address(this).balance;

        if (totalVotesAgainst >= totalVotesFor) {
            _sendEth(s_creator, totalRewards);
        }
        else {
@>          uint256 rewardPerVoter = totalRewards / totalVotes;

            for (uint256 i; i < totalVotesFor; ++i) {
                if (i == totalVotesFor - 1) {
                    rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            }
        }
    }
```

**Impact:** Users that voted "for" the proposal don't recieve the full-rewards. The leftover funds are locked in `VotingBooth` because there is also no `withdraw` function.

**Proof of Concept:**

1. New `booth` is created with 5 users and 10 ETH of rewards
2. 2 users vote for, 1 user votes againt, the `booth` closes and rewards are distributed
3. "For" users are rewarded `3.33` eth and `3.33` ETH is leftover in the contract
4. Leftover ETH can't be taken out of contract without `withdraw` function

Paste test in `VotingBoothTest.t.sol

<details>
<summary>Test</summary>

```js
function testIfPeopleVoteForFundsAreDistributedToEachForVoterAndContractBalanceIsZero() public {
        uint256 startingAmount = address(this).balance;
        console.log("Creator Balance BEFORE: ", startingAmount);
        console.log("- - - - - - - - - - - - - - - - - - - - - - - -");

        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(true);

        vm.prank(address(0x3));
        booth.vote(false);

        assert(!booth.isActive());

        console.log("(FOR) address(0x1).balance: ", address(0x1).balance);
        console.log("(FOR) address(0x2).balance: ", address(0x2).balance);
        console.log("(AGAINST) address(0x3).balance: ", address(0x3).balance);
        console.log("- - - - - - - - - - - - - - - - - - - - - - - -");

        console.log("address(booth).balance: ", address(booth).balance);
        console.log("- - - - - - - - - - - - - - - - - - - - - - - -");

        console.log("Creator Balance AFTER: ", address(this).balance);
}
```

</details>

<details>
<summary>Logs</summary>

```bash
Creator Balance BEFORE:  0
- - - - - - - - - - - - - - - - - - - - - - - -
(FOR) address(0x1).balance:  3333333333333333333
(FOR) address(0x2).balance:  3333333333333333334
(AGAINST) address(0x3).balance:  0
- - - - - - - - - - - - - - - - - - - - - - - -
address(booth).balance:  3333333333333333333
- - - - - - - - - - - - - - - - - - - - - - - -
Creator Balance AFTER:  0
```

</details>


**Recommended Mitigation:** 

1. Add a `withdraw` function

2. Calculate rewards using `totalVotesFor` instead of `totalVotes`:
   1. NOTE: only works with this section commented out:

```js
if (i == totalVotesFor - 1) {
     rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
 }
``` 
Implementation:
```diff
 function _distributeRewards() private {
     uint256 totalVotesFor = s_votersFor.length;
     uint256 totalVotesAgainst = s_votersAgainst.length;
     uint256 totalVotes = totalVotesFor + totalVotesAgainst;
     uint256 totalRewards = address(this).balance;

     if (totalVotesAgainst >= totalVotesFor) {
         _sendEth(s_creator, totalRewards);
     }
     else {
-         uint256 rewardPerVoter = totalRewards / totalVotes;
+         uint256 rewardPerVoter = totalRewards / totalVotesFor;

         for (uint256 i; i < totalVotesFor; ++i) {
             if (i == totalVotesFor - 1) {
                 rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
             }
             _sendEth(s_votersFor[i], rewardPerVoter);
         }
     }
 }
```

<details>
<summary>Logs</summary>

```bash
Creator Balance BEFORE:  0
- - - - - - - - - - - - - - - - - - - - - - - -
(FOR) address(0x1).balance:  5000000000000000000
(FOR) address(0x2).balance:  5000000000000000000
(AGAINST) address(0x3).balance:  0
- - - - - - - - - - - - - - - - - - - - - - - -
address(booth).balance:  0
- - - - - - - - - - - - - - - - - - - - - - - -
Creator Balance AFTER:  0
```

</details>

- - - -

### [H-1] Contract locks Ether without a withdraw function

It appears that the contract includes a payable function to accept Ether but lacks a corresponding function to withdraw it, which leads to the Ether being locked in the contract. To resolve this issue, please implement a public or external function that allows for the withdrawal of Ether from the contract.

<details><summary>1 Found Instances</summary>


- Found in src/VotingBooth.sol [Line: 36](src/VotingBooth.sol#L36)

	```solidity
	contract VotingBooth {
	```

</details>
