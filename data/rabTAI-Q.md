## Impact
The _requireAuthorised() function checks whether the sender is authorized to transfer an NFT and whether the sender is the owner of the token. However, the error messages are inconsistent and unclear. The first require statement has the message "not allowed," while the second has "_from is not the owner!" These messages should be more descriptive and consistent for easier debugging.

## Proof of Concept
The current implementation has two inconsistent error messages:
```
function _requireAuthorised(address _from, uint256 _tokenId) internal view {
    bool isAllowed =
        msg.sender == _from ||
        isApprovedForAll[_from][msg.sender] ||
        msg.sender == getApproved[_tokenId];

    require(isAllowed, "not allowed");
    require(ownerOf(_tokenId) == _from, "_from is not the owner!");
}
```
This can confuse developers or users when debugging issues.

## Tools Used
Manual

## Recommended Mitigation Steps
 Update the error messages to be more descriptive and consistent:


```
function _requireAuthorised(address _from, uint256 _tokenId) internal view {
    bool isAllowed =
        msg.sender == _from ||
        isApprovedForAll[_from][msg.sender] ||
        msg.sender == getApproved[_tokenId];

    require(isAllowed, "Unauthorized transfer attempt");
    require(ownerOf(_tokenId) == _from, "Sender is not the token owner");
}
```
This provides clearer information about the error and makes it easier to understand why the transfer failed.