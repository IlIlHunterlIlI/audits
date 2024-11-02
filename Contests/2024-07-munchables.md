## Summary


| ID                                                                                                                          | Title                                                                                            | Severity |
| --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ | -------- |
| [H-01](Hunter/Portfolio/Contests/Team/2024-07-munchables.md#h-01-infarmplotswrong-logical-opeartor-will-makeplotidto-have-full-rewards)                    | in `farmPlots()` wrong logical opeartor will make `plotId` to have full rewards                  | High     |
| [H-02](Hunter/Portfolio/Contests/Team/2024-07-munchables.md#h-02-casting-underflow-infarmplotsmaking-user-able-to-get-huge-number-of-schnibbles)           | Casting Underflow in `farmplots()` making user able to get huge number of schnibbles             | High     |
| [H-03](Hunter/Portfolio/Contests/Team/2024-07-munchables.md#h-03-infarmplotsan-underflow-in-edge-case-leading-to-freeze-of-funds-nft)                      | in `farmPlots()` an underflow in edge case leading to freeze of funds (NFT)                      | High     |
| [L-01](Hunter/Portfolio/Contests/Team/2024-07-munchables.md#l-01-whenever-a_reconfigurehappens-tobase_schnibble_ratesome-users-will-take-advantages-of-it) | Whenever a `_reconfigure` happens to `BASE_SCHNIBBLE_RATE` some users will take advantages of it | Low      |

## [H-01] in `farmPlots()` wrong logical opeartor will make `plotId` to have full rewards

### Lines of code

[https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L258](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L258)

### Vulnerability details

### Impact

Bigger rewards for invalid `plotId`

The wrong logical operator will make that invalid `plotId` to have full rewards instead of limited rewards calculated with `plotMetadata[landlord].lastUpdated` as `timestamp`

### Proof of Concept

Before the overview of the bug we first need to know that `PlotId` = 0 is a valid Plot.

This also evidenced in `stakeMunchable` as we see here

```solidity
File: LandManager.sol
146:         if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

We make sure that `plotId != totalPlotsAvail` so that a LandLord with 0 Plots can't have users staking on PlotId = 0

This shows that always the valid `PlotId` can't be equal to the number of plots. In simpler terms, its desgined so that valid PlotId are the ones < NumPlots of that LandLord.

With that in mind,

Whenever a LandLord unlock some funds -> his `NumPlots` will dcrease due to this equation

```solidity
File: LandManager.sol
344:     function _getNumPlots(address _account) internal view returns (uint256) {
345:         return lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT;
346:     }
```

For a users who staked on that landLord before he unlocked some funds, there can be users having PlotIds >= `NumPlots` of that LandLord due to the funds unlocked.

Now when those users call functions that call `_farmPlots` we do the following check

```solidity
File: LandManager.sol
258:             if (_getNumPlots(landlord) < _toiler.plotId) {
259:                 timestamp = plotMetadata[landlord].lastUpdated;
260:                 toilerState[tokenId].dirty = true;
261:             }
```

The problem in that `if` is that it checks for `_toiler.plotId` that is > but it didn't take into consideration the `plotId` that is = to `NumPlots` cause these Plot should be now invalid as provided by the information above

Now plotId that is = to `NumPlots` will get full rewards cause its timeStamp will be assigned to `block.timestamp` in the `for` Loop, although it should have gotten its reward calculated from `plotMetadata[landlord].lastUpdated`, but due to the wrong operator (_getNumPlots(landlord) < _toiler.plotId instead of <=) that specific `plotId` will have full rewards.

### Tools Used

Manaul review

### Recommended Mitigation Steps

Put the right logical operator instead of the wrong one

```diff
-           if (_getNumPlots(landlord) < _toiler.plotId) {
+           if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
```

### Assessed type

Invalid Validation


## [H-02] Casting Underflow in `farmplots()` making user able to get huge number of schnibbles

### Lines of code

[https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L283-L286](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L283-L286)

### Vulnerability details

### Impact

the user can get millions of schnibles due to wrong casting.

### Proof of Concept

The problem arises in `_farmPlots`

```solidity
File: LandManager.sol
232:     function _farmPlots(address _sender) internal {
.////////////////////OmitCode
270:             finalBonus =
271:                 int16(
272:                     REALM_BONUSES[
273:                         (uint256(immutableAttributes.realm) * 5) +
274:                             uint256(landlordMetadata.snuggeryRealm)
275:                     ]
276:                 ) +
277:                 int16(
278:                     int8(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
279:                 );
280:             schnibblesTotal =
281:                 (timestamp - _toiler.lastToilDate) *
282:                 BASE_SCHNIBBLE_RATE;
283:             schnibblesTotal = uint256(
284:                 (int256(schnibblesTotal) +
285:                     (int256(schnibblesTotal) * finalBonus)) / 100
286:             );
.////////////////////OmitCode
```

we see in Line 270 the `finalBonus` is by adding `REALM_BONUSES` to `RARITY_BONUSES`

- RARITY_BONUSES = [0, 0, 10, 20, 30, 50]
- REALM_BONUSES =[ 10, -10, -5, 0, 5, -10, 10, -5, 5, 0, -5, 0, 10, 5, -10, 0, -5, 10, 10, -10, 5, 10, 0, -10, 10,]

the problem is that the `finalBonus` can be negative when combining those indexes here

- - REALM_BONUSES is = [1,2,5,7,10,14,16,19,23]
- - index of RARITY_BONUSES is = [0,1]

Now taking into considerations that `finalBonus` can be negative then the problem arises in the following equation

```solidity
File: LandManager.sol
283:             schnibblesTotal = uint256(
284:                 (int256(schnibblesTotal) +
285:                     (int256(schnibblesTotal) * finalBonus)) / 100
286:             );
```

Here we calculate (Schnibbles + (schnibbles * negative value)) / 100

Lets take an Example to show why the above equation is wrong:

- lets Assume that `schnibblesTotal` = 400, `finalBonus` = - 10 then the equation will be the following
- - (400 + (400 * - 10)) / 100 - >
- - (400 + (-4000)) / 100 ->
- - (-3600) / 100 = -36

Now we we cast -36 to uint256 At Line 283 this will cause underflow and will have a value of `type(uint256).max - 36`

### Tools Used

manual testing

### Recommended Mitigation Steps

this can easily prevented by dividing by 100 inside () having `(schnibblesTotal) * finalBonus)`

```diff
            schnibblesTotal = uint256(
                (int256(schnibblesTotal) +
-                   (int256(schnibblesTotal) * finalBonus)) / 100
+                   ((int256(schnibblesTotal) * finalBonus) / 100))
            );
```

### Assessed type

Under/Overflow

## [H-03] in `farmPlots()` an underflow in edge case leading to freeze of funds (NFT)

### Lines of code

[https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L232-L310](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L232-L310)  
[https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L162-L168](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L162-L168)

### Vulnerability details

### Impact

The user won't be able to farm their plots or `unstake` due to the txn always panic reverting. Leading to loss of funds and DOS

### Proof of Concept

The problem arises whenever `PRICE_PER_PLOT` gets increased through `configUpdated`

The problem is that whenever it gets increased, then `_getNumPlots` will return less Numbers for that `LandLord`  
cause its denominated by `PRICE_PER_PLOT`

```solidity
File: LandManager.sol
344:     function _getNumPlots(address _account) internal view returns (uint256) {
345:         return lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT;
346:     }
```

For example:

- a user is staked in plotId 100, and the LandLord has 200 Plots
- `PRICE_PER_PLOT` gets increased and makes the Plots of that Landlord to be 90
- now the user is staked to a Plot that is considered invalid and should be flagged as `dirty` of his `toilerState[tokenId]` struct

Now the problem here is that `NumPlots` is not decreased due to LandLord unlocking funds but instead due to `PRICE_PER_PLOT` getting increased -> meaning that `plotMetadata[landlord].lastUpdated` is too old

Now in `_farmPlots` we see

```solidity
File: LandManager.sol
232:     function _farmPlots(address _sender) internal {
//////////.............OmmitCode
258:             if (_getNumPlots(landlord) < _toiler.plotId) {
259:                 timestamp = plotMetadata[landlord].lastUpdated;
260:                 toilerState[tokenId].dirty = true;
261:             }
//////////.............OmmitCode
280:             schnibblesTotal =
281:                 (timestamp - _toiler.lastToilDate) *
282:                 BASE_SCHNIBBLE_RATE;
//////////.............OmmitCode
309:         accountManager.updatePlayer(mainAccount, renterMetadata);
310:     }
```

in Line 258 we check for `NumPlots` if its lower than `plotId` of the user we then assign timeStamp variable to `landlord.lastUpdated` which is too Old timeStamp (the last time the user locked funds)

Then in Line 281 we substract `timestamp` (which is now `landlord.lastUpdated` which is too old timeStamp) from `_toiler.lastToilDate` which is larger than `timestamp` that will lead to Panic revert

Lets see why `lastToilDate` is larger

we set this variable in `stakeMunchable` to `block.timestamp` here

```solidity
File: LandManager.sol
162:         toilerState[tokenId] = ToilerState({
163:             lastToilDate: block.timestamp,
164:             plotId: plotId,
165:             landlord: landlord,
166:             latestTaxRate: plotMetadata[landlord].currentTaxRate,
167:             dirty: false
168:         });
```

and the `landlord.lastUpdated` is only updated when he lock or unlock funds (which in our case didn't lock or unlock any funds before the user stake to him)

### Tools Used

Manual Review

### Recommended Mitigation Steps

To avoid this case from happening when `PRICE_PER_PLOT` gets increased we should check if `landlord.lastUpdated` is > than `_toiler.lastToilDate` inside the block of this case `if (_getNumPlots(landlord) < _toiler.plotId)`

So that we differentiate from cases that LandLord unlocked funds from cases where `PRICE_PER_PLOT` got increased and The LandLord didn't do any thing with his funds for a while

### Assessed type

Under/Overflow


## [L-01] Whenever a `_reconfigure` happens to `BASE_SCHNIBBLE_RATE` some users will take advantages of it

### Impact

The problem here is that Whenever a `_reconfigure` happens to `BASE_SCHNIBBLE_RATE` if its going to be lower some users will frontun the update to `farmPlots`

if its bigger then some (who didn't farm for a while) will have more schnibbles through `farmPlots` than people who just farmed leading to descripancies between rewards accumulated for users that staked for same time and will lead to loss of funds (schnibbles)

### Proof of Concept

The problem is that Whenever a `_reconfigure` happens to `BASE_SCHNIBBLE_RATE` the old one can't be used for references purposes for people who were staking for a while

**Now lets have an example:**  
Trusted entity chooses to decrease `BASE_SCHNIBBLE_RATE`

This will harm people who were staking for 1 week for the old Rate cause the will be claiming schnibbles in `_farmPlots` at the new rate for the whole 7 days duartion although the new rate has been just updated

```solidity
File: LandManager.sol
280:             schnibblesTotal =
281:                 (timestamp - _toiler.lastToilDate) *
282:                 BASE_SCHNIBBLE_RATE;
```

In addition to that this can be gamed by other users to front run the change by calling `farmPlots`

- This will cause some users to maximally catalize the rate and some people will backward claim schnibbles with the new reduced Rate

same thing with the unfairness the other way if `BASE_SCHNIBBLE_RATE` is to decrease but we flip the example and there would be no FrontRun

### Tools Used

Manual Review

### Recommended Mitigation Steps

Using FlashBots will prevent the frontRun part but won't prevent the unfairness of the fact that some users may have just claimed schnibbles one block earlier before the decrease update by luck.

so the recommendation would be to tract `BASE_SCHNIBBLE_RATE` with aan array of structs that holds in it the Last rate and its updated timeStamp

and we build a logic arround it with `for` loop to see by using `toilerState[tokenId].lastToilDate = timestamp` which `BASE_SCHNIBBLE_RATE` should th user use during claiming and use that rate
