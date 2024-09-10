## 1. Missing Functionality to Update fee_protocol Value
### Impact

fee_protocol value, which is critical for calculating protocol fees during swaps, is never initialized or set in the contract. This means that the fee_protocol value will always default to 0, effectively disabling the collection of protocol fees

### Proof of Concept

Protocol fee calculations and the functionality for the admin to collect rewards are implemented in the contract. However, since the fee_protocol value is never set and there is no function to update it, the variable will always remain at 0, leading to no fees being collected.

```rust
    /// Creates and initialises a new pool.
    pub fn init(
        &mut self,
        price: U256,
        fee: u32,
        tick_spacing: u8,
        max_liquidity_per_tick: u128,
    ) -> Result<(), Revert> {
       //@audit protocol fee not initialized
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
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L31

```rust
    
    pub fn swap(...){
...
          //@audit protocol fee cant be updated always 0

            if fee_protocol > 0 {
                let delta = step_fee_amount.wrapping_div(U256::from(fee_protocol));
                step_fee_amount -= delta;
                state.protocol_fee += u128::try_from(delta).or(Err(Error::FeeTooHigh))?;
            }
```




### Tools Used
Manual

### Recommended Mitigation Steps

Add a function to allow admins set protocol fee.
