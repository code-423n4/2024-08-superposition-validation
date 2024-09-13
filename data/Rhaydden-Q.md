# Quality Assurance

# [QA-01] Everyone has access to administrative functions in `SeawaterAMM.sol`

## Impact
Critical administrative functions in the SeawaterAMM contract are vulnerable to unrestricted access, potentially allowing malicious actors to manipulate pool creation, fee collection, pool enabling/disabling, authorssation of enablers, price setting, and updating of critical roles.

## Proof of Concept
The following functions lack proper access control:

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L160-L178

```solidity
function createPoolD650E2D0(
    address /* token */,
    uint256 /* sqrtPriceX96 */,
    uint32 /* fee */,
    uint8 /* tickSpacing */,
    uint128 /* maxLiquidityPerTick */
) external {
    directDelegate(_getExecutorAdmin());
}

function collectProtocol7540FA9F(
    address /* pool */,
    uint128 /* amount0 */,
    uint128 /* amount1 */,
    address /* recipient */
) external returns (uint128, uint128) {
    directDelegate(_getExecutorAdmin());
}

// ... (similar pattern for other functions)
```

These functions directly delegate calls to `_getExecutorAdmin()` without checking if the caller has the necessary permissions. This allows anyone to execute these critical operations.

## Recommended Mitigation Steps
Implement access control mechanisms for all administrative functions, such as adding an `onlyAdmin` modifier that checks if the caller is authorized before executing the operation.






# [QA-02] Ownership verification bypass enables unauthorized NFT transfers

## Impact
This issue allows unauthorized users to potentially manipulate tokens they don't own, which could cause theft or misuse of NFTs. The impact could be severe if exploited, resulting in loss of assets for legitimate token owners.

## Proof of Concept
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L98-L107

```solidity
function _requireAuthorised(address _from, uint256 _tokenId) internal view {
    bool isAllowed =
        msg.sender == _from ||
        isApprovedForAll[_from][msg.sender] ||
        msg.sender == getApproved[_tokenId];

    require(isAllowed, "not allowed");
    require(ownerOf(_tokenId) == _from, "_from is not the owner!");
}
```

The problem arises because the function first checks if the sender is allowed to perform the action, and only then checks if the `_from` address is the actual owner of the token. This order of operations creates a honeypot for malicious actors to bypass the ownership check.

Consider this scenario:
1. An attacker calls `transferFrom` with a `_from` address that doesn't own the token.
2. The `isAllowed` check passes because the attacker might be approved for all tokens of `_from` or specifically approved for this token.
3. The function then proceeds to call `SEAWATER.transferPositionEEC7A3CD(_tokenId, _from, _to)` without verifying if `_from` actually owns the token.

This could allow an attacker to transfer tokens they don't own, provided they have been approved by the specified `_from` address.

## Recommended Mitigation Steps
We need to reorder the checks in the `_requireAuthorised` function. 

```diff
function _requireAuthorised(address _from, uint256 _tokenId) internal view {
+    require(ownerOf(_tokenId) == _from, "_from is not the owner!");
    
    bool isAllowed =
        msg.sender == _from ||
        isApprovedForAll[_from][msg.sender] ||
        msg.sender == getApproved[_tokenId];

    require(isAllowed, "not allowed");
-    require(ownerOf(_tokenId) == _from, "_from is not the owner!");

}
```




# [QA-03] Undetected overflow in liquidity addition function

## Impact
Potential loss of funds due to undetected overflow in liquidity calculations.

## Proof of Concept
Consider the `add_delta` function in the `liquidity_math.rs` contract. Specifically, in the addition case:
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/maths/liquidity_math.rs#L16-L25

```rust
else            <----- // which means y>= 0
    let z = x.overflowing_add(y as u128);
    if z.0 < x {
        Err(Error::LiquidityAdd)
    } else {
        Ok(z.0)
    }
}
```

The implementation checks for overflow by comparing `z.0 < x`. However, this method fails to detect overflows when `x` is very large (close to the maximum value of u128). When adding a small positive value `y` to such a large `x`, an overflow might occur without changing the most significant bits, resulting in `z.0` being greater than `x` despite the overflow.

For example, consider the case where `x` is `u128::MAX - 1` (340282366920938463463374607431768211455) and `y` is 2:

```rust
let x = u128::MAX - 1;
let y = 2;

let z = x.overflowing_add(y);
assert!(z.0 > x); // This condition passes, incorrectly indicating no overflow
assert!(z.1); // This correctly indicates an overflow occurred
```

In this case, the current implementation would incorrectly return `Ok(z.0)` potentially leading to undetected loss of funds or incorrect balance updates.

## Recommended Mitigation Steps
Consider modifying the `add_delta` function to use the second element of the tuple returned by `overflowing_add`, which explicitly indicates whether an overflow occurred.

```diff
pub fn add_delta(x: u128, y: i128) -> Result<u128, Error> {
    if y < 0 {
        let z = x.overflowing_sub(-y as u128);
        if z.1 {
            Err(Error::LiquiditySub)
        } else {
            Ok(z.0)
        }
    } else {
       let z = x.overflowing_add(y as u128);
-       if z.0 < x {
+       if z.1 {
           Err(Error::LiquidityAdd)
       } else {
           Ok(z.0)
       }
    }
}
```




# [QA-04] Price limit validation in swap function could allow unbounded swaps in some cases

## Proof of Concept
In the `swap` function of the `StoragePool` struct, when a user provides `U256::MAX` as the `price_limit`, the function sets a new limit but fails to properly validate it against the current price:

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L312-L331

```rust
match zero_for_one {
    true => {
        if price_limit == U256::MAX {
            price_limit = tick_math::MIN_SQRT_RATIO + U256::one();
        }
        if price_limit >= self.sqrt_price.get() || price_limit <= tick_math::MIN_SQRT_RATIO
        {
            Err(Error::PriceLimitTooLow)?;
        }
    }
    false => {
        if price_limit == U256::MAX {
            price_limit = tick_math::MAX_SQRT_RATIO - U256::one();
        }
        if price_limit <= self.sqrt_price.get() || price_limit >= tick_math::MAX_SQRT_RATIO
        {
            Err(Error::PriceLimitTooHigh)?;
        }
    }
};
```

For `zero_for_one = true`, if a user provides `U256::MAX` as `price_limit`, it's set to `tick_math::MIN_SQRT_RATIO + U256::one()`. The subsequent check doesn't verify if this new value is still `>= self.sqrt_price.get()`. 

Similarly, for `zero_for_one = false`, if `U256::MAX` is provided, `price_limit` becomes `tick_math::MAX_SQRT_RATIO - U256::one()`, but it's not checked against `self.sqrt_price.get()`.

This allows the swap to proceed with a price limit that could be beyond the current price, effectively removing the price protection for these swaps.

## Recommended Mitigation Steps
Consider implementing something like a two-step validation process for the price limit. First, check if the provided `price_limit` is `U256::MAX` and set the default value if necessary. Then, perform a comprehensive validation of the `price_limit` against both the current price (`self.sqrt_price.get()`) and the allowable range (`MIN_SQRT_RATIO` to `MAX_SQRT_RATIO`). This is just to make sure that regardless of whether the default value is used or not, the price limit is always properly enforced.






# [QA-05] Exact output swaps in `compute_swap_step` function may raise a subtle issue

## Impact
The `compute_swap_step` function may produce incorrect/ inconsistent results when performing exact output swaps, especially in edge cases where the liquidity is slightly more than needed for the requested output amount. As a result, swap calculations will be inaccurate.

## Proof of Concept

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/maths/swap_math.rs#L139-L141
```rust
if !exact_in && amount_out > (-amount_remaining).into_raw() {
    amount_out = (-amount_remaining).into_raw();
}
```

This code attempts to cap the `amount_out` at the requested amount for exact output swaps. However, there are two issues with this approach:

It doesn't account for the fee in the comparison. The `amount_remaining` includes the fee, but `amount_out` doesn't.

It doesn't adjust the `amount_in` or `fee_amount` when capping the `amount_out`, which could lead to inconsistencies in the returned values.

As a result, the function may return incorrect values for `amount_out`, `amount_in`, and `fee_amount` in certain scenarios, particularly when the liquidity is just slightly more than needed for the requested output amount.

## Recommended Mitigation Steps
We could:
1. Calculate the maximum output amount including the fee before comparing it with the calculated `amount_out`.
2. If the calculated `amount_out` exceeds the maximum output amount, adjust both `amount_out` and `amount_in` proportionally to maintain consistency.
3. Recalculate the `fee_amount` based on the adjusted `amount_in` to ensure accurate fee calculations.
