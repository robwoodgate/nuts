# NUT-XX: Pay-to-Blinded-Key (P2BK)

`optional`

`depends on: NUT-11, NUT-18`

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

- Curve: secp256k1, base point `G`, order `n`
- Receiver public key: `P`
- Receiver secret key: `p`
- Sender ephemeral keypair: `(e, E = e·G)` generated **per proof**
- Shared secret: `Zx = x(e·P) or x(p·E)` (32-byte x-coordinate) **per receiver key**
- Keyset identifier: `keyset_id` (mint-supplied)
- Slot index: `i` (0–10). Represents the 11 pubkey limit in a P2PK proof, in the order: `[data, ...pubkeys, ...refund]`
- Deterministic blinding scalar, obtained by either:
  ```
  rᵢ = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || i )
  ```
  or if the first calculation is out of range (`rᵢ = 0` or `rᵢ >= n`):
  ```
  rᵢ = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || i || 0xff )
  ```
  **Note:** All hash inputs must be raw byte format. (See [Determinism](#determinism-and-canonicalisation) section).
- Blinded public key: `P' = P + rᵢ·G`
- Derived private key: `k = (p + rᵢ) mod n`

> [!NOTE]
> The shared secret (`Zx`) is generated per receiver public key (`P`), making it the primary blinding factor. The slot index (`i`) adds additional uniqueness to ensure that if the same receiver public key appears more than once (eg: as a locking AND refund key), it is blinded uniquely. The `keyset_id` adds auxillary uniqueness between mints and epochs.

> [!IMPORTANT]
> If the receiver public key (`P`) was Schnorr derived (eg: Nostr), derive both standard and negated private key candidates and choose the one that generates the expected blinded public key, `P'`
>
> - Standard derivation: `k = (p + rᵢ) mod n`
> - Negated derivation: `k = (-p + rᵢ) mod n`

## Proof Object Extension

Each proof adds a single new metadata field:

```jsonc
{
  "amount": int,
  "id": hex_str,
  "secret": str,          // still ["P2PK", {...}]
  "C": hex_str,
  "p2pk_e": hex_str       // 33-byte SEC1 compressed ephemeral public key E
}
```

- `p2pk_e` contains the sender's ephemeral pubkey (`E`) used for blinding
- All pubkeys inside the `"P2PK"` secret are the blinded forms `P'`
- The mint sees standard P2PK data and remains unaware of the blinding

## Sender Workflow

1. Generate a fresh random scalar `e` and compute `E = e·G`
2. For **each receiver key** `P`, compute: \
   a. Unique shared secret for this key: `Zx = x(e·P)` \
   b. Slot index `i` in `[data, ...pubkeys, ...refund]` \
   c. Blinding scalar: `rᵢ = H("Cashu_P2BK_v1" || Zx || keyset_id || i)`\
   d. Blinded Public Key: `P' = P + rᵢ·G`
3. Build the canonical P2PK secret with the blinded `P'` keys in their slots.
4. Interact with the mint normally; the mint never learns `P` or `rᵢ`
5. Include `p2pk_e = E` in the final proof

> [!IMPORTANT]
> Use a fresh ephemeral keypair (`e` / `E`) for each new output, so that every proof has
> unique blinded keys and a unique `E` in the `Proof.p2pk_e` field.
> In the case of `SIG_ALL` proofs, the **SAME** ephemeral keypair **MUST** be used for all
> outputs, as all proofs must have the SAME locking/refund keys.

## Receiver Workflow

1. Read `E` from `proof.p2pk_e`, `keyset_id` from `proof.id`, and the key slot order index `i` from `[data, ...pubkeys, ...refund]`
2. Calculate your unique shared secret: `Zx = x(p·E)`
3. For each slot `i`, compute: \
   a. Blinding scalar: `rᵢ = H("Cashu_P2BK_v1" || Zx || keyset_id || i)` \
   b. Derived private key: `k = (p + rᵢ) mod n` (or parity-matched variant)
4. Remove the `p2pk_e` field from the proof
5. Sign with the derived private keys and spend as an ordinary P2PK proof

> [!NOTE]
> A receiver can only calculate their OWN shared secret (`pE`), because a shared secret requires either the receiver's private key (`pE`) or the sender's ephemeral private key (`eP`).

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

Where `nut26` indicates whether the receiver supports Pay-to-Blinded-Key and requests the sender to blind the receiver's public keys specified in the `nut10.d` field, as well as any `pubkeys` and `refund` tags in the `nut10.t` field.

### Sender behavior

When a payment request includes a P2PK locking condition in the `nut10` field with `nut26: true`, the sender SHOULD:

1. Blind the receiver's public key(s) specified in the `nut10.d` field, as well as any `pubkeys` and `refund` tags in the `nut10.t` field, before minting/swapping
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
- Concatenate all hash inputs as raw bytes in the order shown.
- `keyset_id` MUST be encoded exactly as provided by the mint.
- `i` is a single unsigned byte (0–10).
- If `rᵢ = 0` or `rᵢ >= n`, retry once with an extra `0xff` byte appended to the hash input; abort if still out of range.

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

## Worked Example

Alice want's to create a 64 sat Proof at Bob's Mint, keyset `009a1f293253e41e`, locked to Carol's public key:

`P: 0295fe407d92879a4669447ac92b1e67346b02c813ad4274bab2cbae2c278cea9e`

She generates an ephemeral secret, `e`, with corresponding pubkey `E` and blinds Carol's pubkey `P` for insertion into the `data` tag (slot `0`) as follows:

```
e = f90a8c8ac22a79893624ae3dcf3a19b75e07e4208454704ebefd4656d172a049 // as hex
E = 0266a67d690652afd786d68467972aab2828f8b4f99ac487e6894b35291f4861c2
keyset_id = '009a1f293253e41e'
Zx = x(e·P) // shared secret
r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 ) // deterministic `r`
if r0 == 0 or r0 >= n:
  r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 || 0xff )
  if r0 == 0 or r0 >= n:
    abort
r0 = 9af8ad5cc63103205bd64c85b392652938292c0b0d680e94a716004d4ff65392 // as hex
P' = P + r0·G // blinded pubkey
P' = 02b7715870996f2081b6237092c055ad65e0278162541a158c5f9c1dc39ea1b63f
```

The resulting proof:

```json
{
  "amount": 64,
  "C": "0382ff1303eeae4fc31ff1c70f062100ab472dc0a1c2109b86b601d2414993ba3a",
  "id": "009a1f293253e41e",
  "secret": "[\"P2PK\",{\"nonce\":\"7ea27e133e9343f77a7ff59644ad7283b568ae165978f454e4e69349e623e86d\",\"data\":\"02b7715870996f2081b6237092c055ad65e0278162541a158c5f9c1dc39ea1b63f\",\"tags\":[]}]",
  "dleq": {
    "s": "cb1171364c499fdff68538532a7bb52fe8eca021eebd87d8e09f4d8705e743ab",
    "e": "97494ca0dc28416d036741c8b3a8ac5274230de0ca6dbd4dee6090b3510ef216",
    "r": "942b9063b4a8e9c688ce76c7074ac7fb6d5aa6a02c2710404a4a7338b7fbc752"
  },
  "p2pk_e": "0266a67d690652afd786d68467972aab2828f8b4f99ac487e6894b35291f4861c2"
}
```

Carol receives the Proof and can derive the blinded signing key `k` as follows:

```
E = proof.p2pk_e
p = 495826c59e1d9d8c2f7a652a2ed02ab71b0f5f2f5cae9bf30a395597760f12c6 (Carol's private key)
slot = 0 (`data` field)
keyset_id = '009a1f293253e41e'

Zx = x(p·E) // shared secret
r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 ) // deterministic `r`
if r0 == 0 or r0 >= n:
  r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 || 0xff )
  if r0 == 0 or r0 >= n:
    abort
r0 = 9af8ad5cc63103205bd64c85b392652938292c0b0d680e94a716004d4ff65392 // as hex
k = (p + r0) mod n
k = 241304eb045c0286af32176e153720627e3b11f4e420848193f9f80781311627

or, if her private key is a Schnorr x-only key, calculate both candidates below and choose the one that generates the blinded pubkey `P'`
standard derivation: k = (p + r0) mod n
negated derivation: k = (-p + r0) mod n
std: 241304eb045c0286af32176e153720627e3b11f4e420848193f9f80781311627
neg: 11de55ce880603ba087a819d51eda9f13768693a8766f86bfa5faa064e854fbc
```

[11]: 11.md
[18]: 18.md
