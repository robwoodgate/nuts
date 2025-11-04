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
- Keyset identifier: `keyset_id` (mint-supplied), hex-decoded to raw bytes before hashing
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
- Derived private key (standard derivation): `k = (p + rᵢ) mod n`
- Derived private key (negated derivation): `k = (-p + rᵢ) mod n`
- Natural Public Key: `PNat = p·G`, allows parity check to Receiver public key (`P`)

> [!NOTE]
> The shared secret (`Zx`) is generated per receiver public key (`P`), making it the primary blinding factor. The slot index (`i`) adds additional uniqueness to ensure that if the same receiver public key appears more than once (eg: as a locking AND refund key), it is blinded uniquely. The `keyset_id` adds auxiliary uniqueness between mints and epochs.

> [!IMPORTANT]
> When the receiver publishes a BIP-340 x-only public key (eg Nostr), an even-Y lift is implied.
> **Sender MUST add the '02' prefix** (`02||x`) to convert it to Cashu's SEC1 compressed format.
> On the receiver side, the wallet’s stored secret (`p`) may produce either parity under `p·G`.
> Implementations **MUST** resolve this BIP-340 sign ambiguity so the derived `k` corresponds
> to the even-Y lift. Two interoperable strategies are allowed:
>
> 1. **Unblind, then select by parity (RECOMMENDED):** \
>    a. compute `Rᵢ = rᵢ·G` \
>    a. unblind `P = P' − Rᵢ` \
>    c. verify `x(P) == x(p·G)` \
>    d. use standard derivation if `parity(P) == parity(p·G)`, otherwise use negated derivation
> 2. **Double-derive and match:** \
>    a. derive both standard and negated candidates
>    b. select the one whose public key reconstructs `P'`
>
> The first approach avoids an extra scalar multiplication per slot, so is recommended.

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
   b. Compute `Rᵢ = rᵢ·G`, and unblind `P = P' − Rᵢ` \
   c. Verify `P` is on curve, not infinity, and that `x(P) == x(p·G)`. If it does not match, this `P'` is not for this private key, skip it. \
   d. Resolve the BIP-340 sign and derive the secret key: either \
    • select by parity as in the **Unblind, then select by parity** method, or \
    • derive both candidates and pick the one that reconstructs `P'`
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
- `keyset_id` The ASCII hex string from the mint MUST be hex-decoded to raw bytes.
- `i` is a single unsigned byte (0–10).
- If `rᵢ = 0` or `rᵢ >= n`, retry once with an extra `0xff` byte appended to the hash input; abort if still out of range.
- All receiver keys **MUST** be in compressed SEC1 format (33 bytes) before ECDH and blinding. The sender **MUST add an '02' prefix** to BIP-340 x only pubkeys (eg Nostr).

## Security Considerations

- **Mint privacy:** the mint never sees `P`, `E`, or `rᵢ`.
- **Freshness:** each proof uses a new `e`; reuse leaks linkage.
- **Replay safety:** `keyset_id` prevents cross-mint reuse, `i` slot binding ensures each `rᵢ` is unique.
- **Infinity handling:** proofs yielding a point at infinity must be discarded.
- **Input validation:** receivers **MUST** validate that `P'` decodes to an on-curve, non-infinite point before unblinding, and **SHOULD** verify the unblinding relation `P' = P + rᵢ·G`.
- **Timing behaviour:** implementations SHOULD use constant-time comparisons where practical. Scalar-side candidate computation is cheap; prefer constant-structure selection at the end.
- **Multi-key tolerance:** in multi-signature or multi-key proofs, receivers **MAY** skip non-matching `P'` for a given private key without error, and proceed to sign any slots that do match.
- **Caching:** wallets MAY precompute and cache `p·G` and its parity per receiver key to avoid repeated multiplications during spends, without affecting security.

## Compatibility

- NUT-11 protocol is unchanged.
- Mints see only pure NUT-11 Proofs. The new `p2pk_e` field in Proof is stripped before sending to the mint.
- Wallets can verify ECDH derivation locally; no mint changes are required.
- Wallets that do not yet support P2BK will not leak privacy by sending P2BK Proofs to the mint. The spend will simply fail.

## Worked Example

Alice wants to create a 64 sat Proof at Bob's Mint, keyset `009a1f293253e41e`, locked to Carol's public key:

`P: 02771fed6cb88aaac38b8b32104a942bf4b8f4696bc361171b3c7d06fa2ebddf06`

She generates an ephemeral secret, `e`, with corresponding pubkey `E` and blinds Carol's pubkey `P` for insertion into the `data` tag (slot `0`) as follows:

```
e = 1cedb9df0c6872188b560ace9e35fd55c2532d53e19ae65b46159073886482ca // as hex
E = 02a8cda4cf448bfce9a9e46e588c06ea1780fcb94e3bbdf3277f42995d403a8b0c
keyset_id = '009a1f293253e41e'
Zx = x(e·P) // shared secret
Zx = '40d6ba4430a6dfa915bb441579b0f4dee032307434e9957a092bbca73151df8b' // as hex
r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 ) // all hash inputs as raw bytes
if r0 == 0 or r0 >= n:
  r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 || 0xff )
  if r0 == 0 or r0 >= n:
    abort
r0 = 41b5f15975f787bd5bd8d91753cbbe56d0d7aface851b1063e8011f68551862d // as hex
P' = P + r0·G // blinded pubkey
P' = 03f221b62aa21ee45982d14505de2b582716ae95c265168f586dc547f0ea8f135f
```

The resulting proof:

```json
{
  "amount": 64,
  "C": "0381855ddcc434a9a90b3564f29ef78e7271f8544d0056763b418b00e88525c0ff",
  "id": "009a1f293253e41e",
  "secret": "[\"P2PK\",{\"nonce\":\"d4a17a88f5d0c09001f7b453c42c1f9d5a87363b1f6637a5a83fc31a6a3b7266\",\"data\":\"03f221b62aa21ee45982d14505de2b582716ae95c265168f586dc547f0ea8f135f\",\"tags\":[]}]",
  "dleq": {
    "s": "6178978456c42eee8eefb50830fc3146be27b05619f04e3490dc596005f0cc78",
    "e": "23f2190b18bfd043d3a526103e15f4a938d646a6bf93b017e2bb7c85e1540b32",
    "r": "d26a55aa39ca50957fdaf54036b01053b0de42048b96a6fb2a167e03f00d0a0f"
  },
  "p2pk_e": "02a8cda4cf448bfce9a9e46e588c06ea1780fcb94e3bbdf3277f42995d403a8b0c"
}
```

Carol receives the Proof and can derive the blinded signing key `k` as follows:

```
E = proof.p2pk_e
p = ad37e8abd800be3e8272b14045873f4353327eedeb702b72ddcc5c5adff5129c (Carol's private key)
slot = 0 (`data` field)
keyset_id = '009a1f293253e41e'

Zx = x(p·E) // shared secret
Zx = '40d6ba4430a6dfa915bb441579b0f4dee032307434e9957a092bbca73151df8b' // as hex
r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 ) // all hash inputs as raw bytes
if r0 == 0 or r0 >= n:
  r0 = SHA-256( "Cashu_P2BK_v1" || Zx || keyset_id || 0 || 0xff )
  if r0 == 0 or r0 >= n:
    abort
r0 = 41b5f15975f787bd5bd8d91753cbbe56d0d7aface851b1063e8011f68551862d // as hex

// Standard derivation
skStd = (p + r0) mod n
skStd = eeedda054df845fbde4b8a579952fd9a240a2e9ad3c1dc791c4c6e51654698c9
// Negated derivation
skNeg = k = (-p + r0) mod n
skNeg = 947e08ad9df6c97ed96627d70e447f1238540da5ac2a25cf208614287592b4d2

// Check parity of Natural Pubkey (p·G) vs Sender Pubkey (P)
pG = '03771fed6cb88aaac38b8b32104a942bf4b8f4696bc361171b3c7d06fa2ebddf06'
P  = '02771fed6cb88aaac38b8b32104a942bf4b8f4696bc361171b3c7d06fa2ebddf06'

// Parity is mismatched to P (is Schnorr even-Y lifted), so use negated derivation
Derived private key = 947e08ad9df6c97ed96627d70e447f1238540da5ac2a25cf208614287592b4d2
```

> [!NOTE]
> For more detailed examples of slot blinding, see the [test vectors][tests].

[11]: 11.md
[18]: 18.md
[tests]: tests/xx-tests.md
