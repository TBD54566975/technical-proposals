# TP18 Message Schema for Supporting Message-Level Encryption

```yaml
TP: 18
Title: TP18 - Message Schema for Supporting Message-Level Encryption
Authors: Henry Tsai (@thehenrytsai)
Comments URI: https://github.com/TBD54566975/technical-proposals/discussions/5
Status: Draft
Created: March 29, 2023
Updated: March 29, 2023
```

## Problem Statement

The DWeb Node (DWN) message schema needs an update to support message-level encryption using key derivation scheme(s) proposed by [TP17](https://github.com/TBD54566975/technical-proposals/pull/4/files).

## Proposal
1. An "encrypted DWN message" refers to its data being encrypted, the message containing metadata itself is _not_ encrypted.

1. A randomly generated symmetric key is used for encryption per DWN message.

   This is enables data encryption at the most granular level in DWN.

1. The symmetric key used for data encryption is encrypted using an asymmetric public key derived through a scheme described in [TP17](https://github.com/TBD54566975/technical-proposals/pull/4/files).

1. The encrypted symmetric key (ciphertext) is attached to the corresponding DWN message.

1. The message schema supports attachment of multiple ciphertext of the _same_ symmetric key produced by encryption using different key derivation schemes.

1. There may be other supported key derivation schemes beyond TP17 in the future.

1. `dataCid` and `dataSize` are computed against the encrypted data.

## Proposed Schema
```json
{
  "recordId": "...",
  "descriptor": {
    "dataCid": "...", // computed against the encrypted data
    "dataSize": 123, // computed against the encrypted data
    ...
  },
  "authorization": { ... },
  "encryption": {
    "algorithm": "A256CTR | etc", // algorithm used to encrypt the data
    "initializationVector": "BASE64URL(Initialization Vector)", // used by data encryption
    // encryption of the SAME encryption key using one or more key derivation scheme
    "keyEncryption":[{
        "derivationScheme": "protocol | schema | etc",
        "algorithm": "ECIES-ES256K | etc", // algorithm used to encrypt the symmetric key
        "encryptedKey": "BASE64URL(Encrypted Data Encryption Key)"
      },
      ...
    ]
  }
}

```

## Considerations
1. The proposed schema takes inspiration from the General JWE JSON Serialization specification, however we are unable take a direct dependency on JWE because:

   a. JWE requires the encrypted data to be a part of the JWE object, the DWN deliberately decouples the data from the message that only contains the metadata.

   b. We could potentially encrypt the content encryption symmetric key in a JWE object as the `ciphertext`, but it means that we will be "double encrypting", coupled with use of ECIES, we will have a total 3 layers of symmetric key encryption, which is an unnecessary overhead.

1. The schema excludes the use of _Authentication Tag_ commonly used in symmetric encryption algorithm (e.g. `AES-GCM`) and specified in JWE, this is because the `dataCid` already serves the same purpose of message integrity verification.


## Open Questions
1. Should `encryptionCid` be added in the `authorization` (and `attestation`) property?

1. Is it okay that the signaling of whether the data is encrypted or not is implicit (by the existence of the `encryption` property)?. A negative here without an explicit `encrypted` property is that there is no quick way to tell in a `RecordsQuery` result if a message of a piece of encrypted data is missing `encryption` if `encryption` is stripped from the message.

1. Should we inherited the abbreviated naming style from JWE for properties like `enc` instead of the `algorithm` property?

1. What is the exact `algorithm` value for ECIES using SECP256K1 curve? I invented the string currently based on what is declared in the [JOSE registry](https://www.iana.org/assignments/jose/jose.xhtml).