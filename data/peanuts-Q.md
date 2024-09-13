### [L-01] grant_position does not check that the `position_owners` mapping has an owner

Function `grant_position()` in lib.rs has a comment stating that the position must not have an owner.

```
 > /// Makes the user the owner of a position. The position must not have an owner.
    fn grant_position(&mut self, owner: Address, id: U256) {
        // set owner
        self.position_owners.setter(id).set(owner);

        // increment count
        let owned_positions_count = self.owned_positions.get(owner) + U256::one();
        self.owned_positions
            .setter(owner)
            .set(owned_positions_count);
    }
```

There is no check whether the position has an owner or not. This means that the current owner can potentially be overwritten with a new owner.

Have a check that looks something like that (not tested!):

```
// Check if the position already has an owner using the getter
    if self.position_owners.getter(id).get().is_some() {
        return Err(Error::PositionAlreadyOwned);
    }
```

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L462


### [L-02] seawater_admin can be set to zero-address in lib.rs function ctor()

The `ctor()` function checks that the `seawater_admin` is not address zero to prevent anyone from calling the function twice. However, the `seawater_admin` itself can be set to a zero address.

```
  pub fn ctor(
        &mut self,
        seawater_admin: Address,
        nft_manager: Address,
        emergency_council: Address,
    ) -> Result<(), Revert> {
        assert_eq_or!(
 >          self.seawater_admin.get(),
            Address::ZERO,
            Error::ContractAlreadyInitialised
        );

        self.seawater_admin.set(seawater_admin);
        self.nft_manager.set(nft_manager);
        self.emergency_council.set(emergency_council);

        Ok(())
    }
```

If the protocol accidentally sets `seawater_admin` to the zero address, anyone can call `ctor()` to set the admin again. Ensure that `seawater_admin` parameter cannot be the zero address.

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L968

### [L-03] Tests are not removed from code

In tick.rs, comment mentions to remove the test but it is still in the code.

```
>  #[cfg(feature = "testing-dbg")] // REMOVEME
        {
            if *fee_growth_global_0 < fee_growth_below_0 {
                dbg!((
                    "fee_growth_global_0 < fee_growth_below_0",
                    current_test!(),
                    fee_growth_global_0.to_string(),
                    fee_growth_below_0.to_string()
                ));
            }
            let fee_growth_global_0 = fee_growth_global_0.checked_sub(fee_growth_below_0).unwrap();
            if fee_growth_global_0 < fee_growth_above_0 {
                dbg!((
                    "fee_growth_global_0 < fee_growth_above_0",
                    current_test!(),
                    fee_growth_global_0.to_string(),
                    fee_growth_above_0.to_string()
                ));
            }
        }
```

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/tick.rs#L206

### [L-04] A position with zero liquidty can still be updated

In position.rs, the update function does not check that the liquidity of the position is zero:

```
 /// Updates a position, refreshing the amount of fees the position has earned and updating its
    /// liquidity.
    pub fn update(
        &mut self,
        id: U256,
        delta: i128,
        fee_growth_inside_0: U256,
        fee_growth_inside_1: U256,
    ) -> Result<(), Error> {
        let mut info = self.positions.setter(id);

        let owed_fees_0 = full_math::mul_div(
            fee_growth_inside_0
                .checked_sub(info.fee_growth_inside_0.get())
                .ok_or(Error::FeeGrowthSubPos)?,
            U256::from(info.liquidity.get()),
            full_math::Q128,
        )?;
```

[Uniswap](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Position.sol) checks that to prevent pokes for 0 liquidity positions

```
  /// @param feeGrowthInside0X128 The all-time fee growth in token0, per unit of liquidity, inside the position's tick boundaries
    /// @param feeGrowthInside1X128 The all-time fee growth in token1, per unit of liquidity, inside the position's tick boundaries
    function update(
        Info storage self,
        int128 liquidityDelta,
        uint256 feeGrowthInside0X128,
        uint256 feeGrowthInside1X128
    ) internal {
        Info memory _self = self;

        uint128 liquidityNext;
        if (liquidityDelta == 0) {
            require(_self.liquidity > 0, 'NP'); // disallow pokes for 0 liquidity positions
            liquidityNext = _self.liquidity;
        } else {
            liquidityNext = LiquidityMath.addDelta(_self.liquidity, liquidityDelta);
        }

        // calculate accumulated fees
        uint128 tokensOwed0 =
            uint128(
```

Considering checking for zero liquidity positions and disallowing the calling of `update()`.

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/position.rs#L43

### [L-05] Best practice to set all helper functions to internal 

In tick.rs, all the functions are publicly exposed, which is not supposed to be the case as it can be easily called and misused.

```
 /// Deletes a tick from the map, freeing storage slots.
    pub fn clear(&mut self, tick: i32) {
        // delete a tick
        self.ticks.delete(tick);
    }
```

In UniswapV3, the contract is a library and all helper functions are internal/private.

```
  /// @param tick The tick that will be cleared
    function clear(mapping(int24 => Tick.Info) storage self, int24 tick) internal {
        delete self[tick];
    }
```

Drop to pub in all helper functions in tick.rs and position.rs. Public functions should only be exposed in lib.rs.

https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/libraries/Tick.sol#L155

### [L-06] uint256 is used instead of uint160

[Uniswap](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/SqrtPriceMath.sol) uses uint160 for the sqrtRatio prices, like so:

```
  function getAmount0Delta(
        uint160 sqrtRatioAX96,
        uint160 sqrtRatioBX96,
        int128 liquidity
    ) internal pure returns (int256 amount0) {
        return
            liquidity < 0
```

sqrt_price_math.rs and other math contracts uses uint256:

```
pub fn _get_amount_0_delta(
    mut sqrt_ratio_a_x_96: U256,
    mut sqrt_ratio_b_x_96: U256,
    liquidity: u128,
    round_up: bool,
) -> Result<U256, Error> {
```

Use uint160 for best practice.

https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/maths/sqrt_price_math.rs#L143

