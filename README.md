vnr
============

[![Build Status]][Build Link] [![Doc Status]][Doc Link] [![Crates
Status]][Crates Link]

[Build Status]: https://github.com/AgeManning/enr/workflows/build/badge.svg?branch=master
[Build Link]: https://github.com/AgeManning/enr/actions
[Doc Status]: https://docs.rs/enr/badge.svg
[Doc Link]: https://docs.rs/enr
[Crates Status]: https://img.shields.io/crates/v/enr.svg
[Crates Link]: https://crates.io/crates/enr

[Documentation at docs.rs](https://docs.rs/enr)

This crate contains an implementation of an Vapory Node Record (VNR) as specified by
[EIP-778](https://eips.ethereum.org/EIPS/eip-778) extended to allow for the use of ed25519 keys.

An VNR is a signed, key-value record which has an associated `NodeId` (a 32-byte identifier).
Updating/modifying an VNR requires an `VnrKey` in order to re-sign the record with the
associated key-pair.

VNR's are identified by their sequence number. When updating an VNR, the sequence number is
increased.

Different identity schemes can be used to define the node id and signatures. Currently only the
"v4" identity is supported and is set by default.

## Signing Algorithms

User's wishing to implement their own singing algorithms simply need to
implement the `VnrKey` trait and apply it to an `Vnr`.

By default, `libsecp256k1::SecretKey` implement `VnrKey` and can be used to sign and
verify ENR records. This library also implements `VnrKey` for `ed25519_dalek::Keypair` via the `ed25519`
feature flag.

Furthermore, a `CombinedKey` is provided if the `ed25519` feature flag is set, which provides an
VNR type that can support both `secp256k1` and `ed25519` signed VNR records. Examples of the
use of each of these key types is given below.

## Features

This crate supports a number of features.

- `serde`: Allows for serde serialization and deserialization for ENRs.
- `ed25519`: Provides support for `ed25519_dalek` keypair types.
- `rust-secp256k1`: Uses `c-secp256k1` for secp256k1 keys.

These can be enabled via adding the feature flag in your `Cargo.toml`

```toml
vnr = { version = "*", features = ["serde", "ed25519", "rust-secp256k1"] }
```

## Examples

To build an VNR, an `VnrBuilder` is provided.

#### Building an VNR with the default `secp256k1` key type

```rust
use vnr::{EnrBuilder, secp256k1};
use std::net::Ipv4Addr;
use rand::thread_rng;

// generate a random secp256k1 key
let mut rng = thread_rng();
let key = secp256k1::SecretKey::random(&mut rng);

let ip = Ipv4Addr::new(192,168,0,1);
let vnr = VnrBuilder::new("v4").ip(ip.into()).tcp(8000).build(&key).unwrap();

assert_eq!(enr.ip(), Some("192.168.0.1".parse().unwrap()));
assert_eq!(enr.id(), Some("v4".into()));
```

#### Building an VNR with the `CombinedKey` type (support for multiple signing algorithms).

Note the `ed25519` feature flag must be set. This makes use of the
`VnrBuilder` struct.

```rust
use vnr::{VnrBuilder, CombinedKey};
use std::net::Ipv4Addr;

// create a new secp256k1 key
let key = CombinedKey::generate_secp256k1();

// or create a new ed25519 key
let key = CombinedKey::generate_ed25519();

let ip = Ipv4Addr::new(192,168,0,1);
let vnr = VnrBuilder::new("v4").ip(ip.into()).tcp(8000).build(&key).unwrap();

assert_eq!(enr.ip(), Some("192.168.0.1".parse().unwrap()));
assert_eq!(enr.id(), Some("v4".into()));
```

#### Modifying an VNR

Vnr fields can be added and modified using the getters/setters on `Vnr`. A custom field
can be added using `insert` and retrieved with `get`.

```rust
use vnr::{VnrBuilder, secp256k1::SecretKey, Vnr};
use std::net::Ipv4Addr;
use rand::thread_rng;

// generate a random secp256k1 key
let mut rng = thread_rng();
let key = SecretKey::random(&mut rng);

let ip = Ipv4Addr::new(192,168,0,1);
let mut vnr = VnrBuilder::new("v4").ip(ip.into()).tcp(8000).build(&key).unwrap();

vnr.set_tcp(8001, &key);
// set a custom key
vnr.insert("custom_key", vec![0,0,1], &key);

// encode to base64
let base_64_string = vnr.to_base64();

// decode from base64
let decoded_enr: Vnr = base_64_string.parse().unwrap();

assert_eq!(decoded_vnr.ip(), Some("192.168.0.1".parse().unwrap()));
assert_eq!(decoded_vnr.id(), Some("v4".into()));
assert_eq!(decoded_vnr.tcp(), Some(8001));
assert_eq!(decoded_vnr.get("custom_key"), Some(&vec![0,0,1]));
```

#### Encoding/Decoding VNR's of various key types

```rust
use vnr::{VnrBuilder, secp256k1::SecretKey, Vnr, ed25519_dalek::Keypair, CombinedKey};
use std::net::Ipv4Addr;
use rand::thread_rng;
use rand::Rng;

// generate a random secp256k1 key
let mut rng = thread_rng();
let key = SecretKey::random(&mut rng);
let ip = Ipv4Addr::new(192,168,0,1);
let vnr_secp256k1 = VnrBuilder::new("v4").ip(ip.into()).tcp(8000).build(&key).unwrap();

// encode to base64
let base64_string_secp256k1 = vnr_secp256k1.to_base64();

// generate a random ed25519 key
let key = Keypair::generate(&mut rng);
let vnr_ed25519 = VnrBuilder::new("v4").ip(ip.into()).tcp(8000).build(&key).unwrap();

// encode to base64
let base64_string_ed25519 = vnr_ed25519.to_base64();

// decode base64 strings of varying key types
// decode the secp256k1 with default Vnr
let decoded_vnr_secp256k1: Vnr = base64_string_secp256k1.parse().unwrap();
// decode ed25519 VNRs
let decoded_vnr_ed25519: Vnr<Keypair> = base64_string_ed25519.parse().unwrap();

// use the combined key to be able to decode either
let decoded_vnr: Vnr<CombinedKey> = base64_string_secp256k1.parse().unwrap();
let decoded_vnr: Vnr<CombinedKey> = base64_string_ed25519.parse().unwrap();
