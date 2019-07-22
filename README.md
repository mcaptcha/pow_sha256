# PoW-SHA256

Rust crate which generates SHA256 Proofs of Work on serializable datatypes. Any type that implements `serde::Deserialize` can be used.

This is a fork of the [`Pow` library](https://github.com/bddap/pow) by bddap with some new additions. Primary of these being:

- PoW datatype now saves the calculation result to be used for checking proof validity given input
- `is_valid_proof` method to do the above mentioned

Other small changes have also been included of various importance but mostly just stylistic/ease of use improvements.

# Examples

Prove we did work targeting a phrase.

```rust
use PoW::PoW;

// very easy mode
let difficulty = u128::max_value() - u128::max_value() / 2;

let phrase = b"Phrase to tag.".to_vec();
let pw = PoW::prove_work(&phrase, difficulty).unwrap();
assert!(pw.score(&phrase).unwrap() >= difficulty);
```

Prove more difficult work. This time targeting a time.

```rust
// more diffcult, takes around 100_000 hashes to generate proof
let difficulty = u128::max_value() - u128::max_value() / 100_000;

let now: u64 = get_unix_time_seconds();
let pw = PoW::prove_work(&now, difficulty).unwrap();
assert!(pw.score(&now).unwrap() >= difficulty);
```

Define a blockchain block.

```rust
struct Block<T> {
    prev: [u8; 32], // hash of last block
    payload: T,     // generic data
    proof_of_work: PoW<([u8; 32], T)>,
}
```

# Score scheme

To score a proof of work for a given (target, PoW) pair:
Sha256 is calculated over the concatenation SALT + target + PoW.
The first 16 bytes of the hash are interpreted as a 128 bit unsigned integer.
That integer is the score.
A constant, SALT, is used as prefix to prevent PoW reuse from other systems such as proof
of work blockchains.

In other words:

```rust
fn score<T: Serialize>(target: &T, PoW_tag: &PoW<T>) -> u128 {
    let bytes = serialize(&SALT) + serialize(target) + serialize(PoW_tag);
    let hash = sha256(&bytes);
    deserialize(&hash[..16])
}
```

# Serialization encoding.

It shouldn't matter to users of this library, but the bincode crate is used for cheap
deterministic serialization. All values are serialized using network byte order.

# Threshold scheme

Given a minimum score m. A PoW p satisfies the minimum score for target t iff score(t, p) >= m.

# Choosing a difficulty setting.

Difficulty settings are usually best adjusted dynamically a la bitcoin.

To manually select a difficulty, choose the average number of hashes required.

```rust
fn difficulty(average: u128) -> u128 {
    debug_assert_ne!(average, 0, "It is impossible to prove work in zero attempts.");
    let m = u128::max_value();
    m - m / average
}
```

Conversely, to calculate probable number of hashes required to satisfy a given minimum
difficulty.

```rust
fn average(difficulty: u128) -> u128 {
    let m = u128::max_value();
    if difficulty == m {
        return m;
    } 
    m / (m - difficulty)
}
```

# License

This project is licensed under either of Apache License, Version 2.0 or MIT license, at your option.