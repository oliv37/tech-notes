# JOSE fundamentals

![JOSE overview](img/01/jose.jpeg)

JOSE (JSON Object Signing and Encryption) groups five related specs, each solving one piece of "how do I trust and/or hide a piece of JSON data":

- **JWS** (JSON Web Signature) — a signed JSON structure. Proves the payload wasn't tampered with and (optionally) who signed it. Content stays readable, just tamper-evident.
- **JWE** (JSON Web Encryption) — an encrypted JSON structure. Hides the payload itself, not just its integrity.
- **JWA** (JSON Web Algorithms) — the registry of algorithms usable inside JWS/JWE (e.g. `HS256`, `RS256`, `ES256`, `A256GCM`).
- **JWK** (JSON Web Key) — a JSON representation of a cryptographic key, used to describe the keys referenced by JWS/JWE. A set of keys (JWKS) is commonly published at a `/.well-known/jwks.json` endpoint so recipients can fetch the public key needed to verify a signature.
- **JWT** (JSON Web Token) — a compact, URL-safe token carrying claims (e.g. user identity, expiration) between parties. Almost always a JWS (signed claims, readable but tamper-proof); rarely a JWE (encrypted claims) when the payload itself must stay confidential.

## Structure

This section covers a JWT built as a JWS — the common case, and what "JWT" usually means unqualified. A JWT built as a JWE instead follows JWE's own 5-part structure (no `signature` field at all) — see the [JWE](#jwe) section below.

A JWS-based JWT is three base64url-encoded parts joined by dots: `header.payload.signature`.

- **Header** — identifies the algorithm (`alg`) and token type (`typ`), and often a key identifier (`kid`) pointing to which key in a JWKS was used.
- **Payload** — the claims: registered ones with a defined meaning (`iss` issuer, `sub` subject, `exp` expiration, `aud` audience, ...), plus any public or private claims the application needs.
- **Signature** — computed over `header + "." + payload` using the algorithm from `alg` and a key. This is the JWS part of a JWT.

### Example

Encoded (what actually gets sent, e.g. in an `Authorization: Bearer` header):

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Decoded:

| Part | base64url | JSON |
|---|---|---|
| Header | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9` | `{"alg":"HS256","typ":"JWT"}` |
| Payload | `eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ` | `{"sub":"1234567890","name":"John Doe","iat":1516239022}` |
| Signature | `SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c` | *(not JSON — raw HMAC-SHA256 bytes over `header + "." + payload`)* |

## Signing and verification

![Signing and verification](img/01/signing-verification.jpg)

How the signature is produced and checked depends on whether `alg` is symmetric or asymmetric.

### Symmetric (e.g. `HS256`)

The same secret both signs and verifies:

- **Sign**: `signature = HMAC-SHA256(secret, header + "." + payload)`.
- **Verify**: the recipient, who must already hold that same secret, recomputes `HMAC-SHA256(secret, header + "." + payload)` and compares it byte-for-byte with the received signature. A match proves the token was signed with that secret and wasn't altered.

Since verifying requires the signing secret itself, every verifier is also capable of signing — fine between two parties that trust each other, but unsuitable for a token an issuer hands out to many independent verifiers.

### Asymmetric (e.g. `RS256`, `ES256`)

Signing and verifying use different keys, and exactly how "verify" works depends on the algorithm — it's not the same operation for every asymmetric scheme.

**RSA (`RS256`, RSASSA-PKCS1-v1.5)** — signing and verifying are inverse operations of the same RSA trapdoor function, so verification really does mean "recover the signed hash and compare it":

![RSA sign and verify](img/01/rsa-sign-verify.jpeg)

- **Sign** (issuer, **private** key): hash the signing input (`header + "." + payload`) with SHA-256, pad it, then raise it to the private exponent (`s = padded_hash^d mod n`) — this is the same modular exponentiation RSA uses for decryption, just applied to a hash instead of a message.
- **Verify** (recipient, **public** key): raise the received signature to the public exponent (`padded_hash' = signature^e mod n` — the same operation RSA uses for encryption), strip the padding to get back `padded_hash'`, then independently compute `SHA-256(header + "." + payload)` and compare the two byte-for-byte. A match is only possible if the signature was produced with the matching private key, since only `d` can undo what `e` will later "undo" again.

**ECDSA (`ES256`)** — verification is *not* a decrypt-and-compare; it's a different construction, an algebraic check that doesn't recover or reconstruct a hash at all. The signature is a pair of numbers `(r, s)`, and verifying combines them with the public key and a freshly computed hash through elliptic-curve point arithmetic, checking that the result lands on a specific point derived from `r` — mathematically unrelated to RSA's "recover then compare," even though it gives the same guarantee (only the private key could have produced a signature that passes).

This is what makes asymmetric algorithms suited to JWTs handed to many verifiers (e.g. any API that accepts the token): the issuer keeps its private key secret, and freely publishes the public key (via JWKS) to anyone who needs to verify.

### Verification flow (recipient side)

1. Decode the header to read `alg` and `kid`.
2. Fetch the matching key: for asymmetric algorithms, the public key from the issuer's JWKS (`kid` selects which entry); for symmetric algorithms, the shared secret agreed with the issuer out of band (it must never appear in a JWKS).
3. Run the verification operation above for that key/algorithm — failure means the token was altered or wasn't signed by the expected issuer.
4. Check the non-cryptographic claims too, like `exp` (token expired?) and `aud` (token meant for this service?).

Because the payload is only base64url-encoded (not encrypted), anyone can read a JWT's claims — a JWS token proves *authenticity*, not *confidentiality*. Sensitive data in claims needs a JWE instead.

## JWK / JWKS

![JWK / JWKS](img/01/jwk-jwks.jpeg)

A JWK describes a single key as JSON. `kty` (key type) determines which other fields are present.

Symmetric key (used to both sign and verify with algorithms like `HS256`):

```json
{
  "kty": "oct",
  "kid": "hmac-key-1",
  "alg": "HS256",
  "k": "AyM1SysPpbyDfgZld3umj1qzKObwVMkoqQ-EstJQLr_T-1qS0gZH75aKtMN3Yj0iPS4hcgUuTwjAzZr1Z9CAow"
}
```

Asymmetric public key (used to verify signatures made with the matching private key, e.g. `RS256`):

```json
{
  "kty": "RSA",
  "kid": "1b94c",
  "use": "sig",
  "alg": "RS256",
  "n": "uZ8sgiVztTawfDtJ7HloDbLw5bDMR2boEAU12TAeGjzr05E_9vHv3fDW_yrahBhX9S_0xBmCl0Tr87qAzZpefarOasxjreAMJn_4kYQgSEluxyBSPKuEnyhhybDzu8ISgNvDPZ7cU0Yhn-UsBo3K3yi8k1c9wnXWqlICsSHGnAYJML-fXIrgeSLLYhXar_lxsp7UTcvVN1pEeJQITSs_xr-gBW_TsYSlTJwcGFHB74q9upmPJn0N-H7P39ewaCdV9kkbZv74shIe2MxMu_zrv1ClRrO1Y2VeSFrbp-d5olujGDGJ8wRwP3FHRt_jBKMme1Pl1TLMap69UJibk1w0LQ",
  "e": "AQAB"
}
```

`n` and `e` are the RSA modulus and public exponent, base64url-encoded — enough to reconstruct the public key, never the private one.

`use` declares what the key is for: `"sig"` here (verifying signatures, i.e. JWS/JWT). A key meant for JWE key-wrapping instead uses `"use": "enc"` — same JWK shape otherwise, just a different intended purpose, so a client picking a key from a JWKS knows not to use a signature-verification key to encrypt.

### Private key

A JWK can also represent a private key: same `kty`, plus the extra parameters needed to reconstruct it. For RSA, that's the private exponent `d` and the CRT (Chinese Remainder Theorem) values used to speed up signing/decryption. This is the private counterpart of the exact public key above (`n`/`e` match, and `d`/`p`/`q`/`dp`/`dq`/`qi` were generated together as one real key pair, verified to sign/verify correctly):

```json
{
  "kty": "RSA",
  "kid": "1b94c",
  "n": "uZ8sgiVztTawfDtJ7HloDbLw5bDMR2boEAU12TAeGjzr05E_9vHv3fDW_yrahBhX9S_0xBmCl0Tr87qAzZpefarOasxjreAMJn_4kYQgSEluxyBSPKuEnyhhybDzu8ISgNvDPZ7cU0Yhn-UsBo3K3yi8k1c9wnXWqlICsSHGnAYJML-fXIrgeSLLYhXar_lxsp7UTcvVN1pEeJQITSs_xr-gBW_TsYSlTJwcGFHB74q9upmPJn0N-H7P39ewaCdV9kkbZv74shIe2MxMu_zrv1ClRrO1Y2VeSFrbp-d5olujGDGJ8wRwP3FHRt_jBKMme1Pl1TLMap69UJibk1w0LQ",
  "e": "AQAB",
  "d": "mgSOfxlpqN3NZ_IfTy0YPvBQckWrOpO9z40t4R3L3HmFiUbm4ZWzMgoPBYAepac3_ySLiStMRbfVEFGauP7IPf2Rc4xO0-HTiHPRYZdVh1ARsFK2lHYSKlLs0NhtZJ2OPN1xuETYYUBlJgdB6y475xl7XxuZuIDjsu8uzKBPQPqpd6oVoagA6cXaPa-4DvTjLFi6Z6aGNwWrcmkPNtMzeOWpEZkzB6ZyE_InQaAeDurYDEuruEldJqyFjS8wwzGIl7hvnzU5yvQui6ZdrXiwE5gmbLqfyquDT3UYGrtlzGGyZeun2sHG-Mw2KUDCN0lAPtz_dk3WNEd6sYCe714J",
  "p": "-Ucwjt8O9x2Os2QCCu6miqZhT85a8Wc-OMyBhc1PTkt50yOquSPbncrRIE26-J_QFJfP6fPAF-bqL2GvIRGNft5Z_E1sKQfP2jLRODSBg0WFrS7IAYSdyf68gZLqS7RbCXkiN_TkHIgTc8r94UFlU_hkOmqM2eGq_M3yrRPKKLU",
  "q": "vqCNsjrot494QVqHJ71LvJwT216LlvGyu0OG5vtUD6eLim_4Zwcpti8RTufhW7hWPx80cQ47gSQdaEfgwtLGEzVbDQLnQcEkRQ0GiSMlZmjH-_y_wU2aUbmir1_RsKeqPBVtddEJkjXR556sxj6Edy5V6rKyCm1-up4wP7juYJk",
  "dp": "BZVupQSiFS74T1RiJ1SJLnnjnCIHL3kcj7ElBZY_vIU8SVp3LIZ9GN--J3cWRRNvWca8xYTMPLzTahRKE_eMr9bTNS-WyKSAJ2cUAKpMbGBWSjbODBqoXinylwRvcxqrH18zQQWfRnRV2zYk0bLhTiw2MojIdujzJ46FbRMQaDE",
  "dq": "NFeLcLPiMHzRbUh5Mf01LY3BmguE0qZ9CBRLl2LNOxsrIpEKKOf7mQ2leAANjgy4ic58rMpWpmgPNxH9ciFDyARvgswYSaLJ3bfs4nznCQ6eUt-cm2w-glAxjmwF_Lac9LCBhMcoFKB4iVu1Xx3Na5gqeS4l2t0j3gQPhX-23Ik",
  "qi": "CRMMA8w7WujBdZnUUpRlyncM_07Twuje2KBozqPHx8R2YmOOUSC5PLAQwGlJaGd4aojlkd4KUozkMLRA-MfvH3DCx5zacBp_BouD-cgSu2B4YUaMh93zOxwYRdNGWlzAhoA6BNZCQg0OXPN4a70OwVg8hZ9vnzLEaDo5-Msyc7w"
}
```

`n` and `e` stay the same as the public key — they're derivable from it, so publishing them is harmless. `d`, `p`, `q`, `dp`, `dq`, `qi` are what turn this into a *private* key: anyone holding them can sign or decrypt. That's why a private-key JWK must stay on the server that owns it and never be published in a JWKS — only public keys (used to verify signatures or encrypt) belong there.

For symmetric keys (`kty: "oct"`), there is no public/private split — the `k` value shown earlier already *is* the secret, so a symmetric JWK is sensitive by nature and must be handled like any other shared secret (not published, not logged).

A JWKS just wraps a list of such keys, letting a client pick the right one via `kid`:

```json
{
  "keys": [
    { "kty": "RSA", "kid": "1b94c", "use": "sig", "alg": "RS256", "n": "uZ8s...", "e": "AQAB" },
    { "kty": "RSA", "kid": "2h9e0", "use": "sig", "alg": "RS256", "n": "sXch...", "e": "AQAB" }
  ]
}
```

Publishing this at a well-known URL (`/.well-known/jwks.json`) lets any relying party fetch current public keys without the issuer sharing its private key — and lets the issuer rotate keys by adding a new one to the set before removing the old.

## JWE

![JWE hybrid encryption](img/01/jwe.jpeg)

A JWS only proves a payload wasn't tampered with — it's still readable by anyone. A JWE encrypts the payload instead, so only the intended recipient can read it. It uses **hybrid encryption**: asymmetric crypto to protect a key, symmetric crypto to protect the actual data (asymmetric algorithms are too slow/limited to encrypt arbitrary-size payloads directly).

1. **Content Encryption Key (CEK)** — a random symmetric key generated fresh for this one message.
2. The payload is encrypted with the CEK using a fast AEAD cipher (`enc`, e.g. `A256GCM` — AES-256-GCM), which produces both a **ciphertext** and an **authentication tag** proving the ciphertext (and header) weren't altered.
3. The CEK itself is then encrypted ("wrapped") with the recipient's **public** key (`alg`, e.g. `RSA-OAEP-256`), so only the recipient's **private** key can recover it.

This is the mirror image of JWS key ownership: in JWS the *signer* holds the private key and everyone else gets the public key to verify; in JWE the *recipient* holds the private key and gives out the public key so anyone can encrypt a message only that recipient can open.

### Structure

Compact serialization has five base64url parts (JWS/JWT only has three):

```
header.encrypted_key.iv.ciphertext.tag
```

- **Header** — `alg` (how the CEK is protected, e.g. `RSA-OAEP-256`) and `enc` (how the payload is encrypted, e.g. `A256GCM`).
- **Encrypted key** — the CEK, encrypted with the recipient's public key. Empty when `alg` is `dir` (a pre-shared symmetric key is used directly as the CEK, skipping key wrapping entirely).
- **IV** — the nonce for the content cipher, random per message (GCM needs 12 bytes).
- **Ciphertext** — the payload, encrypted with the CEK and IV.
- **Tag** — the AEAD authentication tag, covering both the ciphertext and the header (used as "additional authenticated data") — so neither can be swapped without detection.

### Why an IV, and what the tag actually is

GCM turns AES into a stream cipher: it generates a pseudo-random keystream from the CEK plus a counter, and XORs it with the plaintext. The **IV** seeds where that counter starts. Reusing the same IV with the same CEK would produce the same keystream twice, letting an attacker recover plaintext by comparing two ciphertexts — so it just needs to never repeat for a given key, not be secret (it's sent in the clear, as the 3rd part of the JWE).

The **tag** is not encrypted data and can't be decrypted back into anything — it's a keyed integrity check (a MAC-like value) computed from the CEK, IV, ciphertext, and header together:

- **Encrypting**: `tag = MAC(CEK, IV, ciphertext, header)`.
- **Decrypting**: the recipient recomputes that same value from what it received and compares it byte-for-byte with the received tag *before* trusting the plaintext.
- A mismatch means the ciphertext or the header was altered — decryption must be aborted outright, without ever returning partial plaintext (returning anything on a bad tag is what enables oracle-style attacks).

Because the header is included in that computation as "additional authenticated data," the tag also stops an attacker from swapping `alg`/`enc` to a weaker algorithm — not just from touching the ciphertext.

### Example

Generated for `{"sub":"1234567890","role":"admin"}`, encrypted with `RSA-OAEP-256` + `A256GCM` against a throwaway RSA key pair, and round-trip decrypted back to the original JSON to confirm the shape is correct:

```
eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJlbmMiOiJBMjU2R0NNIn0.Gm_N0EE6hiw_yvohcLyt30bMreeE6AG2_7BOVA8KTS6yxqyfFCVosQEAANbc1TCYhelKcDZDBV3FCMitGEINwt8-_V2tGmTKb39mVupRe0zSDV3gMJH3hNARBH-qk1oyMdTVt9PONI2ryUS3roPPLE_p7j4X6FCQikoZZ4SIn-3SlM2H6iFRD5JjsAKPMfao_dMMu9PbG21LTzB6XbS4AnR9j-xNgkOxBiHUXSH2xqVwywknSscQ9-wFYxFOcJmKUqSL0Uz8Qw32f7aHDC4y2l33x7HM2DY9epDXbv26xPDCDE7Flewf_eYsff667HQUBiBjN6wxxFtzpY9I9AfnLw.F-aN9hzeV8zIMgIE.TWeFd3jdCmM1n8ignr8pByRQoPkxaPz-JVuaBSjn5beYTSQ.bp4ypQEqOzTUYrzJocoDVA
```

| Part | base64url | Meaning |
|---|---|---|
| Header | `eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJlbmMiOiJBMjU2R0NNIn0` | `{"alg":"RSA-OAEP-256","enc":"A256GCM"}` |
| Encrypted key | `Gm_N0EE6...AfnLw` | the CEK, encrypted with the recipient's RSA public key |
| IV | `F-aN9hzeV8zIMgIE` | 12-byte GCM nonce |
| Ciphertext | `TWeFd3jdCmM1n8ignr8pByRQoPkxaPz-JVuaBSjn5beYTSQ` | `{"sub":"1234567890","role":"admin"}`, encrypted |
| Tag | `bp4ypQEqOzTUYrzJocoDVA` | GCM authentication tag over ciphertext + header |

### Decryption flow (recipient side)

1. Split the five parts and decode the header to read `alg`, `enc`, and `kid`.
2. Look up its own **private** key matching `kid` (this key never leaves the recipient — unlike JWS verification keys, it's not published anywhere).
3. Decrypt the encrypted-key part with that private key to recover the CEK.
4. Use the CEK, IV, and header (as additional authenticated data) to decrypt the ciphertext and check the tag with the `enc` algorithm — a failed tag check means the ciphertext or header was tampered with, and decryption must be rejected.

A JWT built as a JWE (rather than the far more common JWS) follows this same shape, just with claims as the plaintext payload — used when the claims themselves must stay confidential from anyone who isn't the intended recipient, not just tamper-evident.
