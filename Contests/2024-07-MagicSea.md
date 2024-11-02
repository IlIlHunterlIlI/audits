## Summary


| ID                                                                                                                                                                       | Title                                                                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [H-01](2024-07-MagicSea.md#h-01-inbriberewarder_modifythe-check-of-the-ownership-of-thetokenidwithmsgsenderas-the-passed-parameter-will-makevotervotealways-revert)      | in `BribeRewarder::_modify()` the check of the ownership of the `tokenId` with `msg.sender` as the passed parameter will make `voter::vote()` always revert |
| [H-02](2024-07-MagicSea.md#h-02-unchecked-finishedlockdurationofmlumstakingpositions-duringvoteopening-up-the-ability-to-double-vote-with-same-funds-in-the-same-period) | Unchecked (finished `lockDuration` of `mlumStaking` positions) during `vote()` opening up the ability to double vote with same funds in the same period.    |
| [M-02](2024-07-MagicSea.md#m-01-pools-should-be-compatible-with-weirderc20behavioursmasterchefv2is-incompatibile-with-rebasing-tokens-leading-to-bad-consequences)       | Pools should be compatible with weird `ERC20` behaviours, `MasterChefV2` is incompatibile with Rebasing tokens leading to bad consequences.                 |


## [H-01] in `BribeRewarder::_modify()` the check of the ownership of the `tokenId` with `msg.sender` as the passed parameter will make `voter::vote()` always revert

### Summary

in `BribeRewarder::_modify()` the check of the ownership of the `tokenId` with `msg.sender` as the passed parameter will make `voter::vote()` always revert

### Vulnerability Detail

in function `vote()`

```solidity
File: Voter.sol
153:     function vote(uint256 tokenId, address[] calldata pools, uint256[] calldata deltaAmounts) external {
///////////////.............. skip unnecessary code
210: 
211:             _notifyBribes(_currentVotingPeriodId, pool, tokenId, deltaAmount); // msg.sender, deltaAmount);
///////////////.............. skip unnecessary code
219:     }
```

we call `_notifyBribes` in Line [#211](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/211)

```solidity
File: Voter.sol
221:     function _notifyBribes(uint256 periodId, address pool, uint256 tokenId, uint256 deltaAmount) private {
222:         IBribeRewarder[] storage rewarders = _bribesPerPriod[periodId][pool];
223:         for (uint256 i = 0; i < rewarders.length; ++i) {
224:             if (address(rewarders[i]) != address(0)) {
225:                 rewarders[i].deposit(periodId, tokenId, deltaAmount);
226:                 _userBribesPerPeriod[periodId][tokenId].push(rewarders[i]);
227:             }
228:         }
229:     }
```

this function calls `rewarders` contract (BribeRewarder) so that it notifies them that the user voted for their Pool, so that the user becomes eleigble for claiming rewards

in Line [#225](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/225) we call `deposit()` in (BribeRewarder) contract

```solidity
File: BribeRewarder.sol
143:     function deposit(uint256 periodId, uint256 tokenId, uint256 deltaAmount) public onlyVoter {
144:         _modify(periodId, tokenId, deltaAmount.toInt256(), false);
145: 
146:         emit Deposited(periodId, tokenId, _pool(), deltaAmount);
147:     }
```

the function `deposit()` has `onlyVoter` modifier so that only the `voter` contract is able to call this function which is a good thing, but we need to keep in mind that in the current instance of of `BribeRewarder` the `msg.sender` is the `voter` contract

Now we call `_modify` in Line [#144](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/144)

```solidity
File: BribeRewarder.sol
260:     function _modify(uint256 periodId, uint256 tokenId, int256 deltaAmount, bool isPayOutReward)
261:         private
262:         returns (uint256 rewardAmount)
263:     {
264:         if (!IVoter(_caller).ownerOf(tokenId, msg.sender)) {
265:             revert BribeRewarder__NotOwner();
266:         }
267: 
///////////////............... Skip unnecessary code
298:     }
```

we see in line [#264](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/264) we check if `msg.sender` is the owner of this `tokenId` passed by calling the `voter` contract

the problem here is that since `msg.sender` is the `voter` contract, this check will always fail (since `voter` contract is not the owner of the `tokenId`)

the confusion arised from the fact that `_modify()` function is called inside `claim()` which is called by the actuall user not the `voter` contract and this check will pass then on `claim()` by users

but this check is wrong during `deposit()` as described above

### Impact

the `vote()` in `voter` contract is corrupted and will always fail and revert for any `Pool` having `BribeRewarder` which will almost always be the case.

MediumL breaking core contract functionality

### Tool used

Manual Review

### Recommendation

an easy solution would be changing the check `_modify()` to `tx.origin` here

```diff
-       if (!IVoter(_caller).ownerOf(tokenId, msg.sender))
+       if (!IVoter(_caller).ownerOf(tokenId, tx.origin)) {
            revert BribeRewarder__NotOwner();
        }
```

with taking into considerations the tradeOff using `tx.origin` (if users of the protocol are phished, this will give the ability to attacker to claim the rewards in their behalf)

but i recommend it cause it is simpler and would require less logic changes

## [H-02] Unchecked (finished `lockDuration` of `mlumStaking` positions) during `vote()` opening up the ability to double vote with same funds in the same period.

### Summary

unchecked (finished `lockDuration` of `mlumStaking` positions) during `vote()` in `voter`contract opening up the ability to double vote with same funds in the same period.

### Vulnerability Detail

in `vote()`

```solidity
File: Voter.sol
153:     function vote(uint256 tokenId, address[] calldata pools, uint256[] calldata deltaAmounts) external {
154:         if (pools.length != deltaAmounts.length) revert IVoter__InvalidLength();
155: 
///////////////...............skip
165:         uint256 currentPeriodId = _currentVotingPeriodId;
166:         // check if alreay voted
167:         if (_hasVotedInPeriod[currentPeriodId][tokenId]) {
168:             revert IVoter__AlreadyVoted();
169:         }
170: 
171:         // check if _minimumLockTime >= initialLockDuration and it is locked
172:         if (_mlumStaking.getStakingPosition(tokenId).initialLockDuration < _minimumLockTime) {
173:             revert IVoter__InsufficientLockTime();
174:         }
175:         if (_mlumStaking.getStakingPosition(tokenId).lockDuration < _periodDuration) {
176:             revert IVoter__InsufficientLockTime();
177:         }
///////////////...............skip
179:         uint256 votingPower = _mlumStaking.getStakingPosition(tokenId).amountWithMultiplier;
///////////////...............skip
216:         _hasVotedInPeriod[currentPeriodId][tokenId] = true;
217: 
218:         emit Voted(tokenId, currentPeriodId, pools, deltaAmounts);
219:     }
```

we see that in Line [#167](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/167) we revert if the user has voted before in this Period

in Lines [#172](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/172),[#175](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/175) we check that `lockDuration` and `initialLockDuration` are larger than `_periodDuration`(14 days) and `_minimumLockTime` respectively

To give more context about `lockDuration` from `_mlumStaking`

- those are duration that the user chose to lock his `mlum` and is capped at 365 Days ( to prevent very high multipliers)
- those multipliers are used to get `amountWithMultiplier` that is used for rewards calculations - and voting power retreivals as seen in Line [tedox - Users can double their voting power inside `Voter.sol` #179](https://github.com/sherlock-audit/2024-06-magicsea-judging/issues/179)

The problem here comes from the fact that there is no check for finished `lockDuration` of `mlumStaking` positions and i will describe with an example why

- Alice stakes for 365 Days and gets max multiplier for his money
- Alice lock duration ends but he didn't withdraw his `mlumStaking` position nor renewed it
- Alice sees a Pool that needs votes and wants to vote for it (taking into considerations any intentions for doing so)
- Alice Votes for it
- Alice withdraws his position `fully` and destroy his position (burning the `tokenId` that he voted with before)
- In the same time Alice Deposit his full balance again and gets minted a new `tokenId` that is not registerd in `_hasVotedInPeriod` mapping
- Alice votes again for the same Pool

### Impact

now this has alot of impacts

- breaking invariant of being able to vote two times in the same period with the same funds
- stealing extra rewards from `BribeRewarder` contract connected to that Pool
- funds loss to other Pools in the voting period that should have had more weight than this Pool and would have gotten larger `mlum` emmissions
- voting manipulation and unfair results over all

### Code Snippet

[https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/Voter.sol#L153-L219](https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/Voter.sol#L153-L219)

### Tool used

Manual Review

### Recommendation

check if `position.startLockTime + position.lockDuration) <= _currentBlockTimestamp()` revert the vote() with error `renew your lock`

or you need to make sure that `position.startLockTime + position.lockDuration)` will be less than `_currentBlockTimestamp()` and will be less than the end time of voting period

## [M-01] Pools should be compatible with weird `ERC20` behaviours, `MasterChefV2` is incompatibile with Rebasing tokens leading to bad consequences.

### Summary

Due to the fact that Pools should be compatible with weird `ERC20` behaviours, `MasterChefV2` is incompatibile with Rebasing tokens leading to bad consequences.

### Vulnerability Detail

The problem arises from the fact that `MasterChefV2` should be compatible with any weird `ERC20` Tokens according to the readMe Quoted here

> Any type of ERC20 token. Pools are permissionless. So users can open pools even with weird tokens. Issues regarding any weird token will be valid if they have Med/High impact.

To describe the vulnerability i will give an example

- A new Pool is added in `MasterChefV2`
- the Pool has rebasing token that rebase positively and negatively according to external situation
- 10 Users `deposit` 10e18 each and the Pool now has 100e18 in `totalAmounts` variable in struct `Parameter`
- the token can negatively or positvely rebase
- - in case of negative rebase for example it rebse negatively from 100e18 to 80e18
- - - now the first eight users withdrawing will get their 10e18 back and two people will not have any thing
- - in case of positive rebase the following would happen
- - - the token rebase from 100e18 to 150e18 for example
- - - 10 users withdraw their token normally but 50e18 would stay stuck at the contract, in addition the users would in theory have lost the positive rebase
- - - - to iterate through their loss during positive rebase (a token valued at 100$ and they hold 10 (1k$) each, when positive rebase happens, alot of selling pressure happens making the price go to 75$) now if they could have gotten their positive rebase then this wouldn't be a problem, but they lost it and is stuck in the contract

This always the problem of interacting with rebasing tokens, we can't interact with rebasing token during the normal increment of storage variables during `deposit` or `withdraws`

### Impact

High - Loss of funds for the stakers of that specific Pool

### Code Snippet

[https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/MasterchefV2.sol#L295-L299](https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/MasterchefV2.sol#L295-L299)

[https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/libraries/Amounts.sol#L68-L85](https://github.com/sherlock-audit/2024-06-magicsea/blob/7fd1a65b76d50f1bf2555c699ef06cde2b646674/magicsea-staking/src/libraries/Amounts.sol#L68-L85)

### Tool used

Manual Review

### Recommendation

Either prevent (blacklists) the ability of creating Pools with Rebasing tokens, or implement a logic around those specific Pools (Vault Tokens for example) `ERC4626` that gives the users `shares` to the Pool so that they have a % from depsoits in the Pool whether a rebasing happens or not