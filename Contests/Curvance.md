# Curvance - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Wrong calculation allows early unlock without penalty](#H-01)
    - ### [H-02. Wrong decimal scalation results in user paying wrong amount](#H-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Curvance

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 0

# High Risk Findings

## <a id='#H-01'></a>H-01. Wrong calculation allows early unlock without penalty            

### Relevant GitHub Links
	
https://github.com/curvance/Curvance-CantinaCompetition/blob/c01e6bda82a60ac4842bf0d3363d63e89c870758/contracts/token/VeCVE.sol#L868

https://github.com/curvance/Curvance-CantinaCompetition/blob/c01e6bda82a60ac4842bf0d3363d63e89c870758/contracts/token/VeCVE.sol#L1095

## Vulnerability Details
In the `earlyExpireLock` and `getUnlockPenalty` functions, `_getUnlockPenalty` is called to calculate the `penaltyAmount` for early unlock.

```solidity
function _getUnlockPenalty(
        uint256 amount,
        uint256 penalty,
        uint256 unlockTime
    ) internal view returns (uint256) {
        // Penalty value = lock amount * penalty multiplier, in `WAD`,
        // linearly scaled down as `unlockTime` scales from `LOCK_DURATION`
        // down to 0.
        return
            (amount *
                ((penalty * (LOCK_DURATION - (unlockTime - block.timestamp))) /
                    LOCK_DURATION)) / WAD;
    }
```

But the penalty parameter is passed as whole number instead of percent, for example 3000 instead of 30%.

```solidity
//@audit here we pass the `penalty` as whole number (i.e 3000-9000)
        uint256 penaltyAmount = _getUnlockPenalty(
            amount,
            penalty,
            lock.unlockTime
        );
```

This leads to wrong calculation and penaltyAmount going down to 0.
## Impact
This will allow users to unlock early without any penalty, which also means the DAO will not receive any tokens as compensation for early unlock.

I have coded an executable PoC to demonstrate the problem, you can include it in the EarlyExpireLock.t.sol

```solidity
function test_earlyExpireLock_incorrect_penaltyAmount(
        uint16 penaltyMultiplier,
        bool shouldLock,
        bool isFreshLock,
        bool isFreshLockContinuous
    ) public setRewardsData(shouldLock, isFreshLock, isFreshLockContinuous) {
        //Everything is set up as the `test_earlyExpireLock_success` test, so the `penaltyAmount` MUST be > 0.
        penaltyMultiplier = uint16(bound(penaltyMultiplier, 3000, 9000));
        centralRegistry.setEarlyUnlockPenaltyMultiplier(penaltyMultiplier);

        uint256 penaltyAmount = veCVE.getUnlockPenalty(address(this), 0);

        //Penalty amount MUST NOT be 0, but due to wrong calculation, it results in 0.
        assertEq(0, penaltyAmount);

        vm.expectEmit(true, true, true, true, address(veCVE));
        emit UnlockedWithPenalty(address(this), 30e18, penaltyAmount);
        veCVE.earlyExpireLock(0, rewardsData, "", 0);
    }
```

## Tools Used
Manual Review
## Recommendations
Scale the penalty parameter properly before passing it to the `_getUnlockPenalty` function.
		
## <a id='#H-02'></a>H-02. Wrong decimal scalation results in user paying wrong amount        

### Relevant GitHub Links
	
https://cantina.xyz/code/ac757733-81a4-43c7-8f49-17c5b135cdff/contracts/token/OCVE.sol#L245

## Vulnerability Details
In the exerciseOption function, we call the `_adjustDecimals` function to adjust decimals between `paymentTokenDecimals` and default 18 decimals of `optionExerciseCost`.
```solidity
uint256 payAmount = _adjustDecimals(
                optionExerciseCost, 
                paymentTokenDecimals, 
                18
            );
```
USDC for example has 6 decimals on most of the chains. So the payAmount will most likely have less than 18 decimals since it will be calculated using the else if statement in the following code:
```solidity
function _adjustDecimals(
        uint256 amount,
        uint8 fromDecimals,
        uint8 toDecimals
    ) internal pure returns (uint256) {
        if (fromDecimals == toDecimals) {
            return amount;
        } else if (fromDecimals < toDecimals) {
            return amount * 10 ** (toDecimals - fromDecimals);
        } else {
            return amount / 10 ** (fromDecimals - toDecimals);
        }
    }
```
After that `payAmount` is calculated again using the `FixedPointMathLib.mulDivUp` function
```solidity
payAmount = FixedPointMathLib.mulDivUp(
                optionExerciseCost, 
                payAmount,
                WAD
            );
```
where the payAmount is also passed as parameter as `payAmount * 1e12` in that case, but optionExerciseCost is still in 18 decimals. This will lead to wrong calculation.
## Impact
This will lead to user paying wrong amount.

I coded an executable PoC which you can put in `ExerciseOption.t.sol`
```solidity
function test_exerciseOption_success_withERC20(
    ) public {
        //using the exactly same setup as test_exerciseOption_success_withERC20_fuzzed, but without the fuzzing.
        uint256 amount = 5e18;
        uint256 oCVEUSDCBalance = usdc.balanceOf(address(oCVE));
        uint256 optionExerciseCost = (amount * oCVE.paymentTokenPerCVE()) /
            1e18;
        
        uint256 payAmount = FixedPointMathLib.mulDivUp(
            optionExerciseCost,
            amount * 1e12,
            1e18
        );

        //Here you can see that payAmount actually results in far more than it should be.
        assertEq(payAmount, 25000000000000000000000000000000);
    }
```

## Tools Used
Manual Review
## Recommendations
Scale the `payAmount` to 18 decimals before passing it to the `FixedPointMathLib.mulDivUp` function.
