# NUT-10 Test Vectors

These vectors cover [nutroot secrets](../10.md#nutroot-secrets-v3-keysets) (v3 keysets). All signatures are BIP-340 with the auxiliary randomness fixed to 32 zero bytes, which makes them reproducible; verifiers **MUST** accept any valid signature.

## Conventions

Tag hashes ([NUT-10](../10.md)); the NUMS point is `0250929b74c1a04954b78b4b6035e97a5e078a5a0f28ec96d547bfee9ace803ac0`:

| Tag                   | `SHA256(tag)`                                                      |
| --------------------- | ------------------------------------------------------------------ |
| `Cashu_NutrootLeaf`   | `e19ba80c5d6798399efd68b1d3b0e7ad57e79a2777310a9e6334934ee7a0b52b` |
| `Cashu_NutrootBranch` | `f54194fd19dabbcca1474f329fe5ec065fc54d94063d14ae820788ba5bd8e55e` |
| `Cashu_NutrootTweak`  | `cc14d6872e6d0bc79a3dadb4c43f9362916bb6df512126dda139e65b812facd1` |

The keys throughout are the well-known small test keys, written `key N` for the private scalar `N`:

| Key | Compressed public key                                                |
| --- | -------------------------------------------------------------------- |
| `3` | `02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9` |
| `4` | `02e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd13` |
| `5` | `022f8bde4d1a07209355b4a7250a5c5128e88b84bddc619ab7cba8d569b240efe4` |
| `6` | `03fff97bd5755eeea420453a14355235d382f6472f8568a18b2f057a1460297556` |
| `7` | `025cbdf0646e5db4eaa398f365f2ea7a0e3d419b7e0330e39ce92bddedcac4f9bc` |
| `9` | `03acd484e2f0c7f65309ad178a9f559abde09796974c57e714c35f110dfc27ccbe` |

## Serialized leaves

The `after` leaf used throughout, spelled out (`n = 1`, `keys = [key 4]`, `time = 1755561600`):

```
00 02 | 02 0001 01 | 04 0021 02e493...c4cd13 | 06 0004 68a3be80
```

```json
{
  "threshold_1of1_key3": "00010200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9",
  "threshold_2of2_keys3_4": "00010200010204004202f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f902e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd13",
  "hashlock_hash": "a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1",
  "hashlock_1of1_key3": "00030200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9080020a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1",
  "after_1of1_key4": "00020200010104002102e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd1306000468a3be80"
}
```

## The tree fold

A three-leaf tree (`threshold_1of1_key3`, `after_1of1_key4`, `hashlock_1of1_key3`, in transmitted order). Sorted by leaf hash the order is `h0, h2, h1`: `h0` and `h2` pair and `h1` is promoted, so the path for leaf 2 is `[h0, h1]`.

```json
{
  "three_leaf_tree": [
    "00010200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9",
    "00020200010104002102e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd1306000468a3be80",
    "00030200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9080020a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1a1"
  ],
  "root": "3d4fbecf46f5c716d7cebd48863f3c4ab89e675e3beb4b3a28f3bd13b49d43ad",
  "path_for_index_2": [
    "23e8ff1693496ecad495b7ed3cdd7f8595c52a3adc0b92475835b0fb839116cb",
    "9ed9c0b8907f7af4fce51cbeac218907bbf80ba40f3342df2406bce30616589a"
  ],
  "internal_key": "03a3e12cc077e5605f36441046f50c114fcc883b079a34028bed66732e3a419e51",
  "secret": "022d17fddb224e53e12b40c58ab3e8828d08931640105c52fc4eaf765ed51b9999"
}
```

The commitment also verifies through `path_for_index_2` from leaf 2 alone.

Two copies of `threshold_1of1_key3` under internal key `6`: the root is `tagged_hash("Cashu_NutrootBranch", h || h)`, not the single-leaf `h`, and either copy spends with `path = [h]`:

```json
{
  "leaves": [
    "00010200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9",
    "00010200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9"
  ],
  "leaf_hash": "23e8ff1693496ecad495b7ed3cdd7f8595c52a3adc0b92475835b0fb839116cb",
  "merkle_root": "1eaf291448e2f3c3a4fc00bfd591917bbb807e63af0fb905d054002bddd2cbc6",
  "internal_key": "03fff97bd5755eeea420453a14355235d382f6472f8568a18b2f057a1460297556",
  "secret": "03dd2f11ab23b670222ada50325b5d49cd07e1d7a721d9b52fa2039df1f1b0dbfd"
}
```

## Worked example: receiver-keyed proof with a refund leaf

Alice pays Carol, refundable to Alice after `time`. Carol's static key is key `3`, Alice's refund key is key `4`, Alice's ephemeral is key `5`. The internal key is Carol's static key blinded at slot 0 ([NUT-28](../28.md#nutroot-secrets-v3-keysets)); the tree is the single `after` leaf.

```json
{
  "carol_static": "02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9",
  "ephemeral_E": "022f8bde4d1a07209355b4a7250a5c5128e88b84bddc619ab7cba8d569b240efe4",
  "slot0_r": "7dfb649b0edda814f7cf0feb889e5657eb2083a528aa60a3a943fe0cea066181",
  "internal_key": "03a3e12cc077e5605f36441046f50c114fcc883b079a34028bed66732e3a419e51",
  "leaf": "00020200010104002102e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd1306000468a3be80",
  "merkle_root": "9ed9c0b8907f7af4fce51cbeac218907bbf80ba40f3342df2406bce30616589a",
  "tweak": "b3b7846b14be0650bb03272d179931221637744d1997df80bf27d2df5effe8a4",
  "secret": "02d310a4d661e3158e7d360617e739d6bacbf015431b24a43168db0ab99ef8f828",
  "keypath_priv": "31b2e906239bae65b2d23718a037877b46a91b0b92f99fe8a899725f78d008e7"
}
```

`keypath_priv` is the key Carol signs with.

The witnesses below sign an **illustrative** input digest, `SHA256("illustrative transaction transcript")` = `e1d7170b89a2b6eedec90453e32b6c320dfadd590e6a6454bddec95a0e3834cd`. Carol's key-path witness:

```json
{
  "signatures": [
    "619e0726595b5adff06cc3e6ea1c409f10f7b064cf8888f0eed0efbac854eabf4632642930bdc7c4d7d983379301a4f263991dbd19d96e5ebcfab9e8583bd510"
  ]
}
```

Alice's script-path witness after the locktime:

```json
{
  "leaf": "00020200010104002102e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd1306000468a3be80",
  "control": {
    "K": "03a3e12cc077e5605f36441046f50c114fcc883b079a34028bed66732e3a419e51",
    "path": []
  },
  "signatures": [
    "0b2ea247bfca1264db86907aef4cb19935ed9a5b2043a757259dcdb5c599372c230602f2cfc0cd8b11aa98ff17fbed91f38f07db817263fcc7c50c846715873e"
  ]
}
```

For contrast, a bearer proof with no conditions: private key `7` travels as spend info `k`, and the secret is its bare public key `025cbdf0646e5db4eaa398f365f2ea7a0e3d419b7e0330e39ce92bddedcac4f9bc`.

## Worked example: a template covenant

A proof that key `3` can only spend into fixed txout quotes locked to the two lock keys of the [proof to two fixed txout quotes](00-tests.md#transaction-transcripts) transaction in the NUT-00 vectors. The `template` leaf lists them in `dests`; the internal key is the NUMS offset `u = 7`, as in the [NUT-18 vectors](18-tests.md), so no key path exists. The spend is the swap's 8-sat input (same amount, keyset id and `C`) under this secret, into that transaction's two fixed txout quotes.

```json
{
  "dests": [
    "03db67a892bc5c2c37c1a7979ba28706732deef4b92d69a5f68ad0ab448126d38b",
    "02ccdef4b1ff9e02ee1c142505aa501f2ae3ef30ad9837234d501c26e974f18ae2"
  ],
  "leaf": "00050200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f90c004203db67a892bc5c2c37c1a7979ba28706732deef4b92d69a5f68ad0ab448126d38b02ccdef4b1ff9e02ee1c142505aa501f2ae3ef30ad9837234d501c26e974f18ae2",
  "K": "028edfebd6fdea3e1d89359af20868a2e76315b36cdb1a79de497a1757ca7bd407",
  "merkle_root": "8d41634c46c8ee248a0b8bf65c2a4cd883ece04925e366205e6385cb3e41413d",
  "tweak": "41e5456a9808ef698ad6613fe86647c48d9b75ca0f8abe91a3d01debbbff0115",
  "secret": "03a07a07169b3dbad5ea5e6f7771acb326e788e42e38948d9d2d5f5d3f29b38cf1",
  "transaction_digest": "7431ff2a0c9cda33cd86db4473041cb2577c137e4ce122c862829f623774d3de",
  "input_id": "c7f25fbb091291c9c8e26f68c4b3d9a27a37449ad1923be3b3aa276a25636c29",
  "input_digest": "7dc8094f0a3ca7dd1df79a8289fcbd43f71e9b4202b8a62fc860df7a50dc9f4e"
}
```

The template leaf, spelled out (`n = 1`, `keys = [key 3]`, then `dests`):

```
00 05 | 02 0001 01 | 04 0021 02f930...e036f9 | 0c 0042 03db67...26d38b 02ccde...f18ae2
```

Key `3`'s script-path witness over that `input_digest`:

```json
{
  "leaf": "00050200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f90c004203db67a892bc5c2c37c1a7979ba28706732deef4b92d69a5f68ad0ab448126d38b02ccdef4b1ff9e02ee1c142505aa501f2ae3ef30ad9837234d501c26e974f18ae2",
  "control": {
    "K": "028edfebd6fdea3e1d89359af20868a2e76315b36cdb1a79de497a1757ca7bd407",
    "path": []
  },
  "signatures": [
    "9eafc4d00d4017c0946531e79c772de23234b7682a1839d46e1f933ba0b97854b1b95e9202f8fd7b1054a33b86690cab6eb352bc8a5a785635c7ddaeb267d8be"
  ]
}
```

The same proof spent with its second fixed txout quote locked to key `4` (`02e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd13`) instead, which `dests` does not list, **MUST** be rejected whatever it signs.

## Worked example: an arbiter escrow

Three leaves under the NUMS offset `u = 7`, so no key path exists. The arbiter (key `4`) can pay only the buyer (key `3`) and the seller (key `5`), in any split, through one `template` leaf; the buyer can take everything back alone from `1755561600` through an `after` leaf; buyer and seller together can spend through a 2-of-2 `threshold` leaf.

```json
{
  "leaf_arbiter": "00050200010104002102e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd130c004202f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9022f8bde4d1a07209355b4a7250a5c5128e88b84bddc619ab7cba8d569b240efe4",
  "leaf_refund": "00020200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f906000468a3be80",
  "leaf_coop": "00010200010204004202f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9022f8bde4d1a07209355b4a7250a5c5128e88b84bddc619ab7cba8d569b240efe4",
  "leaf_hash_arbiter": "6a250690c3dacdbb0f3262ed8e7b6cf635cae6084f13213715cbb75c0019d9b0",
  "leaf_hash_refund": "c63d3a44d8775d6fbbe6c92137749831f2e4af57747defbc29ce0933cb77e1d3",
  "leaf_hash_coop": "dfa245f0b825b4ea746ee82f6f41de614a7ceb00e54760327218e5e4487b5c4b",
  "K": "028edfebd6fdea3e1d89359af20868a2e76315b36cdb1a79de497a1757ca7bd407",
  "merkle_root": "5624fbc52d2041cf6bf0b72dc1712a6dc6499a228845718a52848eebb891d14b",
  "tweak": "1c2eff2c35b24cda8445026bfd459011ea82092fc13fbe9cf0db6b1af9ae5f5a",
  "secret": "022d32ec64d123475fac4593830f2e26d6ce1dc3768f6211d60f14d1eba6e06292"
}
```

The arbiter and refund leaves pair and coop is promoted, so `merkle_root = branch(branch(leaf_hash_arbiter, leaf_hash_refund), leaf_hash_coop)`. The seller builds a split and the arbiter signs it: the swap's 8-sat input under this secret into 5 sat to the seller (`id_preimage` `55` repeated 32 times) and 3 sat to the buyer (`33` repeated 32 times) (`transaction_digest` `191f1ee87fe691172ee533a18975538fdcf90a4119d997d422e9137b59e2ed73`, `input_id` `e34b281fd3ade8e3c78cfd147011cb8e00899913e12b637d59b9ad53885b2dd7`, `input_digest` `f0f868802d92c310291c5b831d1bfb598f21a47c3dfc7ac72b5f7e03b7ca272b`). The arbiter's witness reveals its leaf with `path = [leaf_hash_refund, leaf_hash_coop]`:

```json
{
  "leaf": "00050200010104002102e493dbf1c10d80f3581e4904930b1404cc6c13900ee0758474fa94abe8c4cd130c004202f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9022f8bde4d1a07209355b4a7250a5c5128e88b84bddc619ab7cba8d569b240efe4",
  "control": {
    "K": "028edfebd6fdea3e1d89359af20868a2e76315b36cdb1a79de497a1757ca7bd407",
    "path": [
      "c63d3a44d8775d6fbbe6c92137749831f2e4af57747defbc29ce0933cb77e1d3",
      "dfa245f0b825b4ea746ee82f6f41de614a7ceb00e54760327218e5e4487b5c4b"
    ]
  },
  "signatures": [
    "e6bdbdcea86847c3e974e0f1d327adf3fd85fa07361385e365b372c6e82eec5227d2c022d5b7bd5d21f042b7ae60fe48d6c418d0850d720ff510c4b27f56111e"
  ]
}
```

A txout quote to the arbiter's own key is outside `dests`, so no arbiter signature can satisfy the leaf with it.

## Worked example: a commit leaf

A NUMS offset `u = 7` (the offset of the [NUT-18 vectors](18-tests.md)) over a `threshold` leaf `n = 1` for key `3` and a `commit` leaf whose `hash` is `SHA256("external data")`. Key `3` spends through the `threshold` leaf with `path = [leaf_hash_commit]`; a witness revealing `leaf_commit` **MUST** be rejected.

```json
{
  "K": "028edfebd6fdea3e1d89359af20868a2e76315b36cdb1a79de497a1757ca7bd407",
  "hash": "64451ff981aa92887b070762e97302df00a0ab97a853c323394b0a9cfaf46fae",
  "leaf_threshold": "00010200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9",
  "leaf_commit": "000408002064451ff981aa92887b070762e97302df00a0ab97a853c323394b0a9cfaf46fae",
  "leaf_hash_threshold": "23e8ff1693496ecad495b7ed3cdd7f8595c52a3adc0b92475835b0fb839116cb",
  "leaf_hash_commit": "20cccc22a45ddfdbec0647032487870dbeadb03471164eb6767b1234f9914f27",
  "merkle_root": "a23cd934664be4637fd2d9a16febca5ed06e7e3d0adef2231eeb7bcacaa156cd",
  "tweak": "26992654d7ef6a6422bc84867ce87e7c641a59dacf86e2b339e434a92af374cf",
  "secret": "028f86ae14b1a813b7093473faf7b1024979dd6da4db0790126daa2922331afce0"
}
```

## Rejection vectors

An unknown field rejects, and odd type numbers are reserved with none allocated, so this leaf (`threshold_1of1_key3` with a four-byte field `0x09` appended) is malformed. An unallocated leaf type is unsatisfiable: the same leaf's bytes under type `0xff` parse to nothing, so a tree containing it still commits (its hash is one more sibling) but a witness revealing it **MUST** be rejected regardless of its signature:

```json
{
  "leaf_unknown_field": "00010200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9090004deadbeef",
  "leaf_unknown_type": "00ff0200010104002102f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9"
}
```

A `commit` leaf with any other field is malformed, eg with `n = 1`:

```json
{
  "leaf_commit_with_n": "00040200010108002064451ff981aa92887b070762e97302df00a0ab97a853c323394b0a9cfaf46fae"
}
```

Listing Carol's key-path signature twice, or Alice's script-path signature twice (her leaf lists one key), makes each witness above invalid.

## Empty tweak

The empty tweak, `t = tagged_hash("Cashu_NutrootTweak", K)` with no merkle root, for `K` = public key of `3`:

```json
{
  "internal_key": "02f9308a019258c31049344f85f89d5229b531c845836f99b08601f113bce036f9",
  "tweak": "764c0e0da0d17acb5cc863fbe939211869e04e522c017171a0b71e91d5b69908",
  "secret": "03b2bb251c006ae42d9c19f3157d02b3c347b1fb6512885225a8699a06d9233aee",
  "keypath_priv": "764c0e0da0d17acb5cc863fbe939211869e04e522c017171a0b71e91d5b6990b"
}
```
