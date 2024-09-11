## [L-01] Pool remains disabled after initialization, requiring an additional step to enable it

### Vulnerability Detail

The `StoragePool` struct in `pool.rs` contains an `init` function that initializes a new pool with specified parameters such as `price`, `fee`, `tick_spacing`, and `max_liquidity_per_tick`. However, this function does not set the `enabled` flag to `true` after initialization. As a result, the pool remains disabled and unusable until explicitly enabled by an admin through a separate function call. This creates a non-obvious two-step process for pool creation, which can lead to potential errors and confusion.

The relevant part of the `init` function is as follows:

```
pub fn init(
    &mut self,
    price: U256,
    fee: u32,
    tick_spacing: u8,
    max_liquidity_per_tick: u128,
) -> Result<(), Revert> {
    assert_eq_or!(
        self.sqrt_price.get(),
        U256::ZERO,
        Error::PoolAlreadyInitialised
    );

    self.sqrt_price.set(price);
    self.cur_tick
        .set(I32::lib(&tick_math::get_tick_at_sqrt_ratio(price)?));

    self.fee.set(U32::lib(&fee));
    self.tick_spacing.set(U8::lib(&tick_spacing));
    self.max_liquidity_per_tick
        .set(U128::lib(&max_liquidity_per_tick));

    Ok(())
}
```

Functions that interact with the pool (like swapping or adding liquidity) check if the pool is enabled using:

```
assert_or!(self.enabled.get(), Error::PoolDisabled);
```

If the pool is not enabled, these functions will revert with `Error::PoolDisabled`. The pool must be explicitly enabled by calling the `enablePool579DA658` function in `SeawaterAMM.sol`, which delegates the call to the admin executor:

```
function enablePool579DA658(
    address /* pool */,
    bool /* enabled */
) external {
    directDelegate(_getExecutorAdmin());
}
```

### Impact

The pool remains disabled and unusable until explicitly enabled by an admin, creating a non-obvious two-step process for pool creation.

### Recommendation

To ensure the pool is immediately usable after creation and initialization, the `init` function should set the `enabled` flag to `true`:

```
pub fn init(
    &mut self,
    price: U256,
    fee: u32,
    tick_spacing: u8,
    max_liquidity_per_tick: u128,
) -> Result<(), Revert> {
    // ... existing initialization code ...

    self.enabled.set(true);  // Set the pool as enabled after initialization

    Ok(())
}
```




## [L-02] Potential tick manipulation in `flip` function due to improper handling of negative tick values

### Vulnerability Detail

The `flip` function in the `StorageTickBitmap` implementation is responsible for toggling a tick on the bitmap. However, there are several issues in how it handles the `tick` and `spacing` parameters that could lead to potential manipulation of the tick bitmap.

The function converts `spacing` from `u8` to `i32`, which is unnecessary and could lead to unexpected behavior if `spacing` is large. The assertion `assert!(tick % spacing == 0)` is intended to ensure the tick lies on a valid space, but it doesn't account for negative tick values. Additionally, the calculation of `spaced_tick` using integer division can lead to rounding errors for negative tick values.

Here is the relevant part of the code:

```
pub fn flip(&mut self, tick: i32, spacing: u8) {
    let spacing = spacing as i32;
    assert!(tick % spacing == 0); // ensure the tick lies on a valid space

    let spaced_tick = tick / spacing;
    let (word_pos, bit_pos) = ((spaced_tick >> 8) as i16, (spaced_tick % 256));

    let mask = U256::one().wrapping_shl(bit_pos as usize);
    let bitmap = self.bitmap.get(word_pos) ^ mask;

    #[cfg(feature = "testing-dbg")]
    dbg!((
        "inside flip",
        mask.to_string(),
        bitmap.to_string(),
        word_pos
    ));

    self.bitmap.setter(word_pos).set(bitmap);
}
```

These issues combined can potentially allow an attacker to manipulate the tick bitmap in unintended ways, especially when dealing with negative tick values. This could lead to incorrect tick bitmap representation, potentially allowing initialization of ticks that should not be valid, manipulation of the pool's tick structure, and potential exploitation in combination with other functions that rely on the tick bitmap for correct operation.

### Impact

Incorrect tick bitmap representation and potential manipulation of the pool's tick structure.

### Recommendation

To address this issue, the `flip` function should be modified to handle both positive and negative tick values correctly, and to use unsigned types where appropriate. 





## [L-03] Potential silent overflow in fee calculation due to unchecked arithmetic

### Vulnerability Detail

The `update()` function in the `StoragePositions` struct @pkg/seawater/src/position.rs is responsible for updating position details, including the fees owed to users. This function calculates new accumulated fees and updates the `token_owed_0` and `token_owed_1` values. However, the current implementation uses `wrapping_add` and `U128::wrapping_from` for these updates, which can lead to silent overflows and incorrect fee accounting.

The `update()` function performs the following key operations:
1. Calculates `owed_fees_0` and `owed_fees_1` using `full_math::mul_div()`.
2. Updates `token_owed_0` and `token_owed_1` using `wrapping_add()` and `U128::wrapping_from()`.

The core issue lies in the use of `wrapping_add()` and `U128::wrapping_from()`:

```
if !owed_fees_0.is_zero() {
    let new_fees_0 = info
        .token_owed_0
        .get()
        .wrapping_add(U128::wrapping_from(owed_fees_0));
    info.token_owed_0.set(new_fees_0);
}
```

1. `U128::wrapping_from(owed_fees_0)` converts a `U256` value to `U128`. If `owed_fees_0` exceeds `U128::MAX`, the conversion silently truncates the value, leading to a loss of fees.
2. `wrapping_add()` performs addition that wraps around on overflow. If the sum exceeds `U128::MAX`, it will wrap around to a smaller value without raising an error.

In a low-impact scenario, if `info.token_owed_0.get()` is close to `U128::MAX` and `owed_fees_0` is large, their sum could exceed `U128::MAX`. This would cause the value to wrap around, resulting in a much smaller value than expected being stored in `token_owed_0`.

### Impact
Potential loss of user funds due to silent overflow in fee calculation.

### Recommendation

To address this issue, replace the unsafe arithmetic operations with checked operations that will revert the transaction in case of an overflow. 




## [L-04] Missing check for non-existent position ID in `collect_fees` function

### Vulnerability Detail 

The `collect_fees` function in the `StoragePositions` contract does not verify if the position ID exists before attempting to collect fees. This can lead to accessing uninitialized storage, causing undefined behavior or errors. 

The `collect_fees` function is responsible for collecting fees from a position and returning the amount of each token collected. It uses `self.positions.setter(id)` to access the position data. However, this method does not check if the position ID exists and directly accesses the storage. If the position ID does not exist, this can lead to accessing uninitialized storage.

The relevant code snippet is as follows:

```
pub fn collect_fees(&mut self, id: U256) -> (u128, u128) {
    let mut position = self.positions.setter(id);

    let amount_0 = position.token_owed_0.get().sys();
    let amount_1 = position.token_owed_1.get().sys();

    if amount_0 > 0 {
        position.token_owed_0.set(U128::ZERO);
    }
    if amount_1 > 0 {
        position.token_owed_1.set(U128::ZERO);
    }

    (amount_0, amount_1)
}
```

In the above code, `self.positions.setter(id)` is used to access the position data without checking if the position ID exists. This can lead to accessing uninitialized storage, causing undefined behavior or errors.

### Impact 
Accessing uninitialized storage can lead to undefined behavior or errors.

### Recommendation

Add a check to verify the existence of the position before accessing its data. 




## [L-05] Fee calculation in swap_math::compute_swap_step may lead to price impact underestimation

### Vulnerability Detail

The `swap_math::compute_swap_step()` function in the AMM protocol is responsible for calculating the next step in the swap process, including the new price, amount in, amount out, and fee amount. However, the current implementation subtracts the fee from the input amount before calculating the price impact, which can lead to an underestimation of the price impact.

The issue occurs in the following code snippet:

```
let amount_remaining_less_fee = U256::try_from(amount_remaining)
    .map_err(|_| Error::AmountTooHigh)?
    .mul_div_ceil(
        U256::from(1_000_000 - fee_pips),
        U256::from(1_000_000),
    )
    .ok_or(Error::MulDivFailed)?;
```

This calculation subtracts the fee from the input amount before determining the price impact. As a result, the actual impact on the pool's liquidity and price might be underestimated, especially for large swaps or in pools with high fees.

For example, if a user wants to swap a large amount of token0 for token1 in a pool with a high fee, the incorrect fee calculation might lead to an underestimation of the price impact. This could result in the user receiving less token1 than expected, and potentially create arbitrage opportunities that could be exploited by traders.

### Impact

Potential underestimation of price impact in swaps, leading to suboptimal trades and possible arbitrage opportunities.

### Recommendation

Modify the `swap_math::compute_swap_step()` function to apply the fee after calculating the amount required for the swap. This ensures that the price impact is calculated based on the actual amount being swapped. Here's a suggested modification:

```
pub fn compute_swap_step(
    sqrt_ratio_current: U256,
    sqrt_ratio_target: U256,
    liquidity: u128,
    amount_remaining: I256,
    fee_pips: u32,
) -> Result<(U256, U256, U256, U256), Error> {
    // ... existing code ...

    if exact_in {
        amount_in = if zero_for_one {
            get_amount_0_delta(sqrt_ratio_target, sqrt_ratio_current, liquidity, true)?
        } else {
            get_amount_1_delta(sqrt_ratio_current, sqrt_ratio_target, liquidity, true)?
        };

        let amount_remaining_less_fee = U256::try_from(amount_remaining)
            .map_err(|_| Error::AmountTooHigh)?
            .checked_sub(amount_in)
            .ok_or(Error::AmountTooHigh)?
            .mul_div_ceil(
                U256::from(1_000_000 - fee_pips),
                U256::from(1_000_000),
            )
            .ok_or(Error::MulDivFailed)?;

        // ... rest of the function ...
    }

    // ... rest of the function ...
}
```

This modification ensures that the fee is applied after calculating the amount required for the swap, leading to more accurate price impact calculations.




## [L-06] Ineffective slippage protection in SeawaterAMM swap functions exposes users to potential losses

### Vulnerability Detail 

The `SeawaterAMM` contract implements swap functionality through its `swapIn32502CA71()` and `swapOut5E08A399()` functions. These functions are designed to facilitate token swaps with slippage protection, but they contain a critical flaw in their implementation that renders the slippage protection ineffective.

The root cause of the issue lies in how these functions set the `price_limit` parameter when calling the underlying `swap904369BE()` function. In both `swapIn32502CA71()` and `swapOut5E08A399()`, `price_limit` is set to `type(uint256).max`:

```
(bool success, bytes memory data) = _getExecutorSwap().delegatecall(abi.encodeCall(
    ISeawaterExecutorSwap.swap904369BE,
    (
        token,
        true,
        int256(amountIn),
        type(uint256).max  // This effectively disables price limit protection
    )
));
```

Setting `price_limit` to the maximum possible value effectively disables any price limit protection during the swap execution. This means that the swap can proceed regardless of how much the price moves during the transaction, leaving users vulnerable to unlimited slippage.

The functions do include a slippage check, but it's performed after the swap has already occurred:

```
require(-swapAmountOut >= int256(minOut), "min out not reached!");
```

While this post-swap check can cause the transaction to revert if the slippage exceeds the user's specified limit, it happens too late in the process. By this point, the swap has already impacted the pool's state, potentially leaving it in an inconsistent condition if the transaction reverts.

This vulnerability exposes users to several risks:
1. No real-time slippage protection during swap execution.
2. Increased vulnerability to sandwich attacks and price manipulation.
3. Potential for inconsistent pool state if transactions revert post-swap.
4. Unnecessary gas costs for users on reverted transactions.

### Impact 
Users are exposed to potentially unlimited slippage during swaps, which could lead to significant financial losses, especially during volatile market conditions or under targeted attacks.

### Recommendation

Implement real-time slippage protection by calculating an appropriate `price_limit` based on the user's `minOut` parameter before executing the swap. Modify the `swapIn32502CA71()` and `swapOut5E08A399()` functions as follows:

```
function swapIn32502CA71(address token, uint256 amountIn, uint256 minOut) external returns (int256, int256) {
    uint256 priceLimit = calculatePriceLimit(amountIn, minOut, true);
    (bool success, bytes memory data) = _getExecutorSwap().delegatecall(abi.encodeCall(
        ISeawaterExecutorSwap.swap904369BE,
        (
            token,
            true,
            int256(amountIn),
            priceLimit  // Use calculated price limit instead of type(uint256).max
        )
    ));
    require(success, string(data));

    (int256 swapAmountIn, int256 swapAmountOut) = abi.decode(data, (int256, int256));
    require(-swapAmountOut >= int256(minOut), "min out not reached!");
    return (swapAmountIn, swapAmountOut);
}
```

Implement a `calculatePriceLimit()` function that computes an appropriate price limit based on the input amount, minimum output, and swap direction. This will ensure that the swap fails early if the current market price would result in slippage beyond the user's specified tolerance, providing effective protection against price manipulation and unexpected market movements.






## [L-07] Lack of signature validation in Permit2 integration may allow unauthorized token transfers

### Vulnerability Detail 

The Seawater AMM, based on Uniswap v3 and designed for Arbitrum's Stylus environment, implements a Permit2 integration for gasless approvals of token transfers. This integration is primarily handled in the `lib.rs` file, which contains two key functions: `swap_permit_2_E_E84_A_D91()` and `swap_2_exact_in_permit_2_36_B2_F_D_D8()`.

These functions accept a signature parameter (`sig: Vec<u8>`) meant to authorize token transfers on behalf of the owner. They create a `Permit2Args` struct with this signature and other data:

```rust
let permit2_args = Permit2Args {
    max_amount,
    nonce,
    deadline,
    sig: &sig,
};
```

However, neither function validates the provided signature before proceeding with the swap operation. For example, in `swap_permit_2_E_E84_A_D91()`:

```rust
pub fn swap_permit_2_E_E84_A_D91(
    &mut self,
    pool: Address,
    zero_for_one: bool,
    amount: I256,
    price_limit_x96: U256,
    nonce: U256,
    deadline: U256,
    max_amount: U256,
    sig: Vec<u8>,
) -> Result<(I256, I256), Revert> {
    let permit2_args = Permit2Args {
        max_amount,
        nonce,
        deadline,
        sig: &sig,
    };

    Pools::swap_internal(
        self,
        pool,
        zero_for_one,
        amount,
        price_limit_x96,
        Some(permit2_args),
    )
}
```

This lack of validation could potentially allow an attacker to forge Permit2 approvals and initiate unauthorized token transfers.

### Impact 
Potential for unauthorized token transfers if an attacker can forge valid Permit2 signatures.

### Recommendation

Implement signature validation in both `swap_permit_2_E_E84_A_D91()` and `swap_2_exact_in_permit_2_36_B2_F_D_D8()` functions before proceeding with swap operations. This can be done by adding a validation function:

```rust
fn validate_permit2_signature(owner: Address, permit2_args: &Permit2Args, domain_separator: [u8; 32]) -> Result<(), Error> {
    let message = encode_permit2_message(owner, permit2_args, domain_separator);
    let message_hash = keccak256(message);
    
    let signature = Signature::from_slice(&permit2_args.sig)?;
    let recovered_address = signature.recover(message_hash)?;
    
    if recovered_address != owner {
        return Err(Error::InvalidSignature);
    }
    
    Ok(())
}
```

Then, call this validation function before the swap operations in both affected functions.





## [L-08] Tick initialization in StorageTicks::update() lacks tick spacing validation

### Vulnerability Detail 

The `StorageTicks::update()` function in `tick.rs` is responsible for updating and initializing ticks with liquidity and fee data. However, it does not validate whether the `tick` being updated aligns with the pool's tick spacing. This oversight can lead to the creation of invalid ticks and potential miscalculations in fee growth.

The function takes several parameters including `tick`, `cur_amm_tick`, and various fee and liquidity values, but lacks a `tick_spacing` parameter for validation. The critical section where this validation is missing is:

```
if liquidity_gross_before == 0 {
    // initialise ourself
    if tick <= cur_amm_tick {
        info.fee_growth_outside_0.set(*fee_growth_global_0);
        info.fee_growth_outside_1.set(*fee_growth_global_1);
    }
    info.initialised.set(true);
}
```

This code initializes a tick without checking if it's valid according to the pool's tick spacing. Additionally, the condition `if tick <= cur_amm_tick` doesn't account for `cur_amm_tick` potentially being an invalid tick, which could lead to incorrect fee growth initialization.

### Impact 
Potential creation of invalid ticks and minor inaccuracies in fee calculations.

### Recommendation

Modify the `StorageTicks::update()` function to include a `tick_spacing` parameter and add validation for tick alignment. Here's a suggested implementation:

```
pub fn update(
    &mut self,
    tick: i32,
    cur_amm_tick: i32,
    liquidity_delta: i128,
    fee_growth_global_0: &U256,
    fee_growth_global_1: &U256,
    upper: bool,
    max_liquidity: u128,
    tick_spacing: i32,
) -> Result<bool, Error> {
    if tick % tick_spacing != 0 {
        return Err(Error::InvalidTick);
    }

    // ... existing code ...

    if liquidity_gross_before == 0 {
        let nearest_valid_tick = (cur_amm_tick / tick_spacing) * tick_spacing;
        if tick <= nearest_valid_tick {
            info.fee_growth_outside_0.set(*fee_growth_global_0);
            info.fee_growth_outside_1.set(*fee_growth_global_1);
        }
        info.initialised.set(true);
    }

    // ... rest of the function ...
}
```

This change ensures that only valid ticks are initialized and fee growth is calculated correctly based on the nearest valid tick.