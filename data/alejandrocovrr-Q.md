# Missing validation on word extraction

On **pkg/seawater/src/eth_serde.rs line 17**

Function take_word: 

```Rust
/// Extracts a 256 bit word from a data stream, returning the word and the remaining data.
pub fn take_word(data: &[u8]) -> (&[u8; 32], &[u8]) {
    #[allow(clippy::unwrap_used)]
    data.split_first_chunk::<32>().unwrap()
}
```

## Quality issue
Does not validate that data length is sufficient to do the split.

## Impact

Unchecked assumptions about data length can lead to logical errors or vulnerabilities.

## Remediation

Validation such as: 

```Rust
if data.len() < 32 {
    return Err(Error::InsufficientData);
}

let (word, remaining) = data.split_first_chunk::<32>().ok_or(Error::InsufficientData)?;
```

Would ensure that the length of the data to extract or split chunks is sufficient. 