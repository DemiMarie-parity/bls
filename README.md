# bls [![Crates.io](https://img.shields.io/crates/v/bls-like.svg)](https://crates.io/crates/bls-like) #

Boneh-Lynn-Shacham (BLS) signatures have slow signing, very slow verification, require slow and much less secure pairing friendly curves, and tend towards dangerous malleability.  Yet, BLS permits a diverse array of signature aggregation options far beyond any other known signature scheme, which makes BLS a preferred scheme for voting in consensus algorithms and for threshold signatures. 

In this crate, we take a largely unified approach to aggregation techniques and verifier optimisations for BLS signature:  We support the [BLS12-381](https://z.cash/blog/new-snark-curve.html) (Barreto-Lynn-Scott) curves via ZCash's traits, but abstract the pairing so that developers can choose their preferred orientation for BLS signatures.  We provide aggregation techniques based on messages being distinct, on proofs-of-possession, and on delinearization, although we do not provide all known optimisations for delinearization.  

We cannot claim these abstractions provide miss-use resistance, but they at least structure the problem, provide some guidlines, and maximize the relevance of warnings present in the documentation.

## Documentation

You first bring the `bls` crate into your project just as you normally would.

```rust
use bls_like::{Keypair,ZBLS};

let keypair = Keypair::<ZBLS>::generate(::rand::thread_rng());
let message = Message::new(b"Some context",b"Some message");
let sig = keypair.sign(message);
assert!( sig.verify() );
```

In this example, `sig` is a `SignedMessage<ZBLS>` that contains the message hash, the signer's public key, and of course the signature, but one should usually detach these constituents for wire formats.

Aggregated and blind signatures are almost the only reasons anyone would consider using BLS signatures, so we focus on aggregation here.  We assume for brevity that `sigs` is an array of `SignedMessage`s, as one might construct like 

```rust
let sigs = msgs.iter().zip(keypairs.iter_mut()).map(|(m,k)| k.sign(*m)).collect::<Vec<_>>();  
```

As a rule, aggregation that requires distinct messages still requires one miller loop step per message, so aggregate signatures have rather slow verification times.  You can nevertheless achieve quite small signature sizes like

```rust
let mut dms = sigs.iter().try_fold(
    ::bls::distinct::DistinctMessages::<ZBLS>::new(), 
    |dm,sig| dm.add(sig)
).unwrap();
dms.signature()
```

Anyone who receives the already aggregated signature along with a list of messages and public keys might reconstruct this like:

```rust
let mut dms = msgs.iter().zip(publickeys).try_fold(
    ::bls::distinct::DistinctMessages::<ZBLS>::new(), 
    |dm,(message,publickey)| dm.add_message_n_publickey(message,publickey)
) ?;
dms.dms.add_signature(signature);
dms.verify()
```

We recommend distinct message aggregation like this primarily for verifying proofs-of-possession, meaning checking the self certificates for numerous keys.  

Assuming you already have proofs-of-possession, then you'll want to do aggregation with `BitPoPSignedMessage` or some variant tuned to your use case.  We recommend more care when using `BatchAssumingProofsOfPossession` because it provides no mechanism for checking a proof-of-possession table.

```rust
// TODO: Use BitPoPSignedMessage
```

If you lack proofs-of-possesion, then delinearized approaches are provided in the `delinear` module, but such schemes might require a more customised approach.

## Security Warnings

This library does not make any guarantees about constant-time operations, memory access patterns, or resistance to side-channel attacks.

## License

Licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally
submitted for inclusion in the work by you, as defined in the Apache-2.0
license, shall be dual licensed as above, without any additional terms or
conditions.

