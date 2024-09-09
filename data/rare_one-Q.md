Lack of Events for update and collect_fees functions 

The contract does not emit any events when fees are collected or positions are updated.
 Without these events, it is difficult to track important actions such as fee collection or updates to position data. 

Affected Lines of code:

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

        let owed_fees_1 = full_math::mul_div(
            fee_growth_inside_1
                .checked_sub(info.fee_growth_inside_1.get())
                .ok_or(Error::FeeGrowthSubPos)?,
            U256::from(info.liquidity.get()),
            full_math::Q128,
        )?;

        let liquidity_next = liquidity_math::add_delta(info.liquidity.get().sys(), delta)?;

        if delta != 0 {
            info.liquidity.set(U128::lib(&liquidity_next));
        }

        info.fee_growth_inside_0.set(fee_growth_inside_0);
        info.fee_growth_inside_1.set(fee_growth_inside_1);
        if !owed_fees_0.is_zero() {
            // overflow is the user's problem, they should withdraw earlier
            let new_fees_0 = info
                .token_owed_0
                .get()
                .wrapping_add(U128::wrapping_from(owed_fees_0));
            info.token_owed_0.set(new_fees_0);
        }
        if !owed_fees_1.is_zero() {
            let new_fees_1 = info
                .token_owed_1
                .get()
                .wrapping_add(U128::wrapping_from(owed_fees_1));
            info.token_owed_1.set(new_fees_1);
        }

        Ok(())
    }















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
}





Impact:
Auditing Difficulty: Users and developers will face difficulties when trying to track and verify when and how often positions are updated or fees are collected.
Potential for Disputes: In case of disputes regarding fee collection or position changes, the lack of emitted events complicates proving actions that were taken, potentially leading to trust issues.

Mitigation:
Emit events such as PositionUpdated and FeesCollected whenever a position is updated or fees are collected. This will provide better traceability and allow off-chain tools to interact more effectively with the contract. 