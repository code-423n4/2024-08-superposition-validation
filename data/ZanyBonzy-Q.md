### OwnershipNFTs.sol doesn't fully match EIP-721's implementation.

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L182

#### Impact

`tokenURI` doesn't enforce that the token being checked is a valid token, which can lead to malicious users impersonating bogus NFTs as the ownership NFT. Unsuspecting users would attempt to query the token uri and would be returned with a valid uri, since the function doesn't ensure that the tokenId is a valid NFT.

Also, according to EIP-721 metadata standard, the function should revert if `_tokenId` is not a valid NFT.

> /// @notice A distinct Uniform Resource Identifier (URI) for a given asset.
>    /// @dev Throws if `_tokenId` is not a valid NFT. URIs are defined in RFC

```solidity
    function tokenURI(uint256 /* _tokenId */) external view returns (string memory) {
        return TOKEN_URI;
    }
```

#### Recommended Mitigation Steps

Add the check for NFT existence in the function by checking that the owner is not address(0)

***
### OwnershipNFTs's approve function allows approved users to also perform approvals.

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L160-L163

#### Impact

OwnershipNFTs's approve function allows approved users to also perform approvals, which can lead to approval loss.

The approve function checks if `msg.sender` is authorized. 

```solidity
    function approve(address _approved, uint256 _tokenId) external payable {
        _requireAuthorised(msg.sender, _tokenId);
        getApproved[_tokenId] = _approved;
    }
```
This is enforced by the `_requireAuthorised` which ensures that caller is either owner, owner's operator or approved to spend the token.
```solidity
    function _requireAuthorised(address _from, uint256 _tokenId) internal view {
        // revert if the sender is not authorised or the owner
        bool isAllowed =
            msg.sender == _from ||
            isApprovedForAll[_from][msg.sender] ||
            msg.sender == getApproved[_tokenId];

        require(isAllowed, "not allowed");
        require(ownerOf(_tokenId) == _from, "_from is not the owner!");
    }
```
This is an issue because any user that had be approved to spend the token risks losing their approval after calling the `approve` function.

#### Recommended Mitigation Steps

Recommend limiting the approval to the owner and operator instead.
***

### Exisitng backdoor for owner/operator in the `approve` function

* Lines of code
 
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L160-L163

#### Impact

The approve function allows operator and owner to approve themselves to spend the token. This can be used to give themselves a backdoor to the toeken. If for example the operator's status is to be revoked, he can always call the `approve` function on the token, to approve himself to spend the token. So even if his operator status is revoked, he still has access to the tokens.

```solidity
    function approve(address _approved, uint256 _tokenId) external payable {
        _requireAuthorised(msg.sender, _tokenId);
        getApproved[_tokenId] = _approved;
    }
```

#### Recommended Mitigation Steps

Ensure that the owner/operator cannot approve themselves to spend a token.

***

### Consider automatically collecting fees before transferring positons

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/lib.rs#L554-L569

#### Impact

`transfer_position_E_E_C7_A3_C_D` directly removes the positions from the sender and adds it to the receiver without making considerations for potential fees that the position has accrued. 

```rust
    pub fn transfer_position_E_E_C7_A3_C_D(
        &mut self,
        id: U256,
        from: Address,
        to: Address,
    ) -> Result<(), Revert> {
        assert_eq_or!(msg::sender(), self.nft_manager.get(), Error::NftManagerOnly);

        self.remove_position(from, id);
        self.grant_position(to, id);

        #[cfg(feature = "log-events")]
        evm::log(events::TransferPosition { from, to, id });

        Ok(())
    }
```

#### Recommended Mitigation Steps

Recommend introducting `collect_single_to_6_D_76575_F` into the `transfer_position_E_E_C7_A3_C_D` function to help users directly claim fees before transferring their positions.

***


### Functions to claim fees should be accessible even when pool is disabled

* Lines of code
 
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L562-L565

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/seawater/src/pool.rs#L538-L559

#### Impact

The `collect` and `collect_protocol` functions allow users and protocol admins to claim their fees. These functions are however not accessible when the pool is disabled. As a rule of thumb, functions to remove rewards or withdraw tokens from a protocol should always be accessible to help users claim resuce their tokens. Also a malicious admin can disable the pool and renounce his ownership, permanently locking the tokens in the protocol.

```rust
    pub fn collect(&mut self, id: U256) -> Result<(u128, u128), Revert> {
>>      assert_or!(self.enabled.get(), Error::PoolDisabled);
        Ok(self.positions.collect_fees(id))
    }
```


```rust
    pub fn collect_protocol(
        &mut self,
        amount_0: u128,
        amount_1: u128,
    ) -> Result<(u128, u128), Revert> {
>>      assert_or!(self.enabled.get(), Error::PoolDisabled);

        let owed_0 = self.protocol_fee_0.get().sys();
        let owed_1 = self.protocol_fee_1.get().sys();

        let amount_0 = u128::min(amount_0, owed_0);
        let amount_1 = u128::min(amount_1, owed_1);

        if amount_0 > 0 {
            self.protocol_fee_0.set(U128::lib(&(owed_0 - amount_0)));
        }
        if amount_1 > 0 {
            self.protocol_fee_1.set(U128::lib(&(owed_1 - amount_1)));
        }

        Ok((amount_0, amount_1))
    }
```


#### Recommended Mitigation Steps

Recommend removing the check for pool activity in the functions.

***


### Certain getter functions limited to admin executor.

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L400-L435

#### Impact

The below functions getter functions, allowing to retrieve various pool information but are limited to the executor admin. As a result, interactions with external protocols or just ordinary users trying to query these function will fail. Not sure the best severity for this as it affects external integrations.

```solidity
    function sqrtPriceX967B8F5FC5(address /* pool */) external returns (uint256) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feesOwed22F28DBD(
        address /* pool */,
        uint256 /* position */
    ) external returns (uint128, uint128) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function curTick181C6FD9(address /* pool */) external returns (int32) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function tickSpacing653FE28F(address /* pool */) external returns (uint8) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feeBB3CF608(address /* pool */) external returns (uint32) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feeGrowthGlobal038B5665B(address /* pool */) external returns (uint256) {
        directDelegate(_getExecutorAdmin());
    }

    /// @inheritdoc ISeawaterExecutorAdminExposed
    function feeGrowthGlobal1A33A5A1B(address /*pool */) external returns (uint256) {
        directDelegate(_getExecutorAdmin());
    }
```

#### Recommended Mitigation Steps
Recommend limiting to `_getExecutorPosition` instead.

***

### `enablePool579DA658` can only be queried by admin, which locks out authorized enablers and emergency council

* Lines of code

https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/SeawaterAMM.sol#L181-L187

#### Impact

The function to enable pool - `enablePool579DA658` is intended to be called by the admin, emergency council and authorized enablers. But the function in SeawaterAMM.sol, only the admin can call the function locking out the other privileged roles.

```solidity
    function enablePool579DA658(
        address /* pool */,
        bool /* enabled */
    ) external {
        directDelegate(_getExecutorAdmin());
    }
```

***