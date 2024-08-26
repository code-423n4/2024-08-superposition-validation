## Title
OwnershipNFTs has Non-Unique tokenURI Functionality

## URL
```url
https://github.com/code-423n4/2024-08-superposition/blob/4528c9d2dbe1550d2660dac903a8246076044905/pkg/sol/OwnershipNFTs.sol#L181-L184
```

## Impact
Detailed description of the impact of this finding.

The tokenURI function in the OwnershipNFTs contract returns the same URI (TOKEN_URI) for all tokens. This could be a design oversight, especially for NFTs, where each token is typically expected to have a unique metadata URI. This non-unique URI behavior can limit the usability of the NFTs in contexts where individual token identification is critical (e.g., marketplaces, games, etc.).

The issue can lead to confusion or incorrect representation of NFTs in systems that rely on unique metadata for each token. This can result in misinterpretation of the token’s purpose or value, and in severe cases, may lead to loss of confidence in the platform if users expect unique URIs for each NFT.

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

The following code demonstrates how the tokenURI function currently returns the same URI for all tokens. This example uses Remix IDE to deploy and interact with the OwnershipNFTs contract:
```sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;

interface ISeawaterAMM {
    function positionOwnerD7878480(uint256 _tokenId) external view returns (address);
    function positionBalance4F32C7DB(address _spender) external view returns (uint256);
    function transferPositionEEC7A3CD(uint256 _tokenId, address _from, address _to) external;
}

contract OwnershipNFTs {
    ISeawaterAMM immutable public SEAWATER;
    string public TOKEN_URI;

    constructor(
        string memory _name,
        string memory _symbol,
        string memory _tokenURI,
        ISeawaterAMM _seawater
    ) {
        TOKEN_URI = _tokenURI;
        SEAWATER = _seawater;
    }

    function tokenURI(uint256 /* _tokenId */) external view returns (string memory) {
        return TOKEN_URI;
    }
}

// Deployment and interaction in Remix IDE:
// 1. Deploy OwnershipNFTs with a sample URI like "https://example.com/nft"
// 2. Call tokenURI with any token ID (e.g., 1, 2, 3) and observe the output is always "https://example.com/nft"
```
**Expected Behavior:**
Each token ID should return a unique URI associated with its metadata.

**Actual Behavior:**
All token IDs return the same URI, which could be misleading in many NFT use cases.

## Tools Used
Manual review & Remix IDE.

## Recommended Mitigation Steps

To resolve this issue, implement a mapping of token IDs to unique URIs and adjust the tokenURI function accordingly. Here’s how you can do this:
```sol
// SPDX-License-Identifier: MIT
pragma solidity 0.8.16;

interface ISeawaterAMM {
    function positionOwnerD7878480(uint256 _tokenId) external view returns (address);
    function positionBalance4F32C7DB(address _spender) external view returns (uint256);
    function transferPositionEEC7A3CD(uint256 _tokenId, address _from, address _to) external;
}

contract OwnershipNFTs {
    ISeawaterAMM immutable public SEAWATER;

    // New mapping for token URIs
    mapping(uint256 => string) private _tokenURIs;

    constructor(
        string memory _name,
        string memory _symbol,
        ISeawaterAMM _seawater
    ) {
        SEAWATER = _seawater;
    }

    // New function to set a token URI
    function _setTokenURI(uint256 _tokenId, string memory _uri) internal {
        _tokenURIs[_tokenId] = _uri;
    }

    function tokenURI(uint256 _tokenId) external view returns (string memory) {
        require(bytes(_tokenURIs[_tokenId]).length != 0, "URI not set for this token");
        return _tokenURIs[_tokenId];
    }
}

// How to use in Remix IDE:
// 1. Deploy OwnershipNFTs without specifying a TOKEN_URI.
// 2. Use _setTokenURI to set URIs for each token ID as needed.
// 3. Call tokenURI with different token IDs to retrieve their specific URIs.
```