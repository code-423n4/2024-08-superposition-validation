Vulnerability Details:

Type:Unchecked Overflow


Location: update function in StoragePositions struct

Code Snippet:

let liquidity_next = liquidity_math::add_delta(info.liquidity.get().sys(), delta)?;

Vulnerable Line:
liquidity_math::add_delta(info.liquidity.get().sys(), delta)?

My Explanation:

The update function calculates the new liquidity_next value by adding the delta value to the current liquidity value using the add_delta function from the liquidity_math module. However, this operation is not checked for overflow.

If the delta value is large enough the addition operation can exceed the maximum value representable by the u128 type, causing an overflow. This can result in an incorrect liquidity_next value being calculated.

The Impact:

- Incorrect calculation of liquidity_next value
Leads to Potential overflow error



Recommendation:

Add overflow checks using std::u128::checked_add or a similar mechanism
     AND 
Consider using a wider integer type, like u256, to reduce the likelihood of overflow

AFFECTED FILE CONTRACT:
https://github.com/code-423n4/2024-08-superposition/blob/main/pkg/seawater/src/position.rs
