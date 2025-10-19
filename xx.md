# NUT-XX: Pay-to-Blinded-Key (P2BK)

`optional`

`depends on: NUT-11`

---

## Summary

Pay-to-Blinded-Key (P2BK) extends [NUT-11][11] (P2PK) by letting senders blind the receiver public keys with fresh randomness before any mint interaction.

Mints continue to see ordinary P2PK secrets; only wallet-to-wallet transfers carry the extra `p2pk_e` metadata that enables the receiver to construct the spend key.

## Motivation

In NUT-11, receiver pubkeys are part of the Proof secret, so the mint learns them on redemption.

P2PK transactions are therefore linkable and lack privacy if pubkeys are reused, or the proof is locked to a long-lived public key (such as a Nostr ID).

P2BK preserves privacy by blinding each receiver pubkey `P` with an ECDH-derived scalar `r`. Both sides can deterministically derive the same `r` from their own keys, but a third party cannot.

This brings _"silent payments"_ to Cashu: Proofs can be locked to a well known public key, posted in public without compromising privacy, and spent by the recipient without needing any side-channel communication.

## Definitions

- Curve: secp256k1, base point `G`, order `n`.
- Receiver public key: `P`.
- Receiver secret key: `p`.
- Sender ephemeral keypair: `(e, E = e·G)` generated **per proof**.
- Shared secret: `Zx = x(e·P) or x(p·E)` (32-byte x-coordinate).
- Keyset identifier: `keyset_id` (mint-supplied).
- Slot index: `i` (0–10). Represents the 11 pubkey limit in a P2PK proof, in the order: `[data, ...pubkeys, ...refund]`
- Deterministic blinding scalar:
  ```
  rᵢ = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || i ) mod n
  ```
- Blinded public key: `P′ = P + rᵢ·G`.
- Derived private key: `k = (p + rᵢ) mod n`

**Note:** If the receiver public key (`P`) was Schnorr derived (eg: Nostr), calculate both standard and negated candidates and choose the one that generates the expected blinded public key, `P′`

Standard derivation: `k = (p + r0) mod n`
Negated derivation: `k = (-p + r0) mod n`

## Proof Object Extension

Each proof adds a single new metadata field:

```json
{
  "amount": int,
  "id": hex_str,
  "secret": str,          // still ["P2PK", {...}]
  "C": hex_str,
  "p2pk_e": hex_str       // 33-byte SEC1 compressed ephemeral public key E
}
```

- `p2pk_e` contains the sender’s ephemeral pubkey (`E`) used for blinding.
- All pubkeys inside the `"P2PK"` secret are the blinded forms `P′`.
- The mint sees standard P2PK data and remains unaware of the blinding.

## Sender Workflow

1. Generate a fresh random scalar `e` and compute `E = e·G`.
2. For each receiver key `P`, compute:
   a. Slot index `i` in `[data, ...pubkeys, ...refund]`
   a. `Zx = x(e·P)`.
   b. `rᵢ = H("Cashu_P2BK_v1" || Zx || keyset_id || i) mod n`.
   c. `P′ = P + rᵢ·G`.
3. Build the canonical P2PK secret with the blinded `P′` keys in their slots.
4. Interact with the mint normally; the mint never learns `P` or `rᵢ`.
5. Include `p2pk_e = E` in the final proof.

## Receiver Workflow

1. Read `E` from `proof.p2pk_e`, `keyset_id` from `proof.id`, and the key slot order index `i` from `[data, ...pubkeys, ...refund]`.
2. Compute `rᵢ = H("Cashu_P2BK_v1" || x(p·E) || keyset_id || i) mod n`.
3. Derive `k = (p + rᵢ) mod n` (or parity-matched variant).
4. Remove the `p2pk_e` field from the proof.
5. Sign and spend the proof as an ordinary P2PK output.

## Payment request extension

This NUT extends the `PaymentRequest` object from [NUT-18][18] with an optional `nut26` field to signal P2BK support.

### Extended PaymentRequest schema

```json
{
  "i": str <optional>,
  "a": int <optional>,
  "u": str <optional>,
  "s": bool <optional>,
  "m": Array[str] <optional>,
  "d": str <optional>,
  "t": Array[Transport] <optional>,
  "nut10": NUT10Option <optional>,
  "nut26": bool <optional>,
}
```

Where `nut26` indicates whether the receiver supports Pay-to-Blinded-Key and requests the sender to blind the receiver's public keys specified in the `nut10.d` field.

### Sender behavior

When a payment request includes a P2PK locking condition in the `nut10` field with `nut26: true`, the sender SHOULD:

1. Blind the receiver's public key(s) specified in the `nut10.d` field (and any keys in `nut10.t` pubkeys and refund tags) before minting/swapping
2. Include the ephemeral public key `E` in the `p2pk_e` (`pe` for Token V4) field of the resulting proofs
3. Send the P2BK-enhanced proofs to the receiver

### Example payment request with P2BK

```json
{
  "i": "c5f3d829",
  "a": 21,
  "u": "sat",
  "m": ["https://mint.example.com"],
  "d": "Payment for coffee",
  "nut10": {
    "k": "P2PK",
    "d": "02a9acc1e48c25eeeb9289b5031cc57da9fe72f3fe2861d264bdc074209b107ba2"
  },
  "nut26": true,
  "t": [
    {
      "t": "post",
      "a": "https://receiver.example.com/payment"
    }
  ]
}
```

In this example:

- The receiver signals P2BK support with `"nut26": true`
- The receiver requires P2PK locking with their public key in the `nut10.d` field
- The sender will blind the public key `02a9acc1e48c25eeeb9289b5031cc57da9fe72f3fe2861d264bdc074209b107ba2` before interacting with the mint
- The sender will include their ephemeral public key `E` in the `p2pk_e` field of the resulting proofs
- The sender will use a new ephemeral keypair (`e, E`) for each proof.

## Determinism and Canonicalisation

- Hash function: SHA-256.
- Concatenate raw bytes in the order shown.
- `keyset_id` MUST be encoded exactly as provided by the mint.
- `i` is a single unsigned byte (0–10).
- If `rᵢ = 0`, retry once with an extra `0xff` byte appended to the hash input; abort if still zero.

## Security Considerations

- **Mint privacy:** the mint never sees `P`, `E`, or `rᵢ`.
- **Freshness:** each proof uses a new `e`; reuse leaks linkage.
- **Replay safety:** `keyset_id` prevents cross-mint reuse, `i` slot binding ensures each `rᵢ` is unique.
- **Infinity handling:** proofs yielding a point at infinity must be discarded.

## Compatibility

- NUT-11 protocol is unchanged.
- Mints see only pure NUT-11 Proofs. The new `p2pk_e` field in Proof is stripped before sending to the mint.
- Wallets can verify ECDH derivation locally; no mint changes are required.
- Wallets that do not yet support P2BK will not leak privacy by sending P2BK Proofs to the mint. The spend will simply fail.

## Example

Alice want's to create a 64 sat Proof at Bob's Mint, keyset `009a1f293253e41e`, locked to Carol's public key:

`P: 021405b887592231816f01ba57d12ae6a37588fccbdc3050e069ac6e9e17963abf`

She generates an ephemeral secret, `e`, with corresponding pubkey `E` and blinds Carol's pubkey `P` for insertion into the `data` tag (slot `0`) as follows:

```
keyset_id = '009a1f293253e41e'
Zx = x(e·P) // shared secret
r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 ) mod n // deterministic `r`
P′ = P + r0·G // blinded pubkey
```

The resulting proof:

```json
{
  "amount": 64,
  "C": "03b6512524f9f2fc1fc2b1c1fa68ac3c037bb7a6070dea780af39a2186b8d417b3",
  "id": "009a1f293253e41e",
  "secret": "[\"P2PK\",{\"nonce\":\"fa5f4310eacc45d8b67daf6887a1994d661127ddad35e5a19eaa2df2d9123d4f\",\"data\":\"02c736ee0610ccfa8702c83f36a829b8b79aaa54f28617aa1bb6b45fb571fa3690\",\"tags\":[]}]",
  "witness": undefined,
  "dleq": {
    "s": "e2d068d139debea907e28cdc70caf00230019e7c7c2f82c5dccf36b89622dfa9",
    "e": "c6da3bf95534789037ff2c67453027739c9fd7b5efb9b2e071758f400ba7ba78",
    "r": "d87a5db3a1e45d28cebb9cfa7fb2269f151efd82ad053a5e5979d1d70a55d76a"
  },
  "p2pk_e": "0267720bc602b2a9939ea0b00d77cac1401d8032009603497bb64afd5023ab59f8"
}
```

Carol receives the Proof and can derive the blinded signing key `k` as follows:

```
E = proof.p2pk_e
p = Carol's private key
slot = 0 (`data` field)
keyset_id = '009a1f293253e41e'

Zx = x(p·E) // shared secret
r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 ) mod n // deterministic `r`
k = (p + r0) mod n

or, if her private key is a Schnorr x-only key, calculate both candidates below and choose the one that generates the blinded pubkey `P′`
standard derivation: k = (p + r0) mod n
negated derivation: k = (-p + r0) mod n
```

## References

[11]: 11.md
[18]: 18.md
