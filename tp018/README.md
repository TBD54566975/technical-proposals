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
  "authorization": { must contain encryptionCid },
  "encryption": {
    "algorithm": "A256CTR | etc", // algorithm used to encrypt the data
    "initializationVector": "BASE64URL(Unique Initialization Vector)", // used in data encryption
    // encryption of the SAME encryption key using one or more key derivation scheme
    "keyEncryption":[{
        "rootKeyId": "ID of the root asymmetric key used to derive the public key used for this key encryption"
        "derivationScheme": "protocol | schema | etc",
        "algorithm": "ECIES-ES256K | etc", // algorithm used to encrypt the symmetric key
        "encryptedKey": "BASE64URL(Encrypted Data Encryption Key)",
        // operational properties dependent on derivation scheme and/or key encryption algorithm:
        "initializationVector": "BASE64URL(Unique Initialization Vector)", // used in ECIES
        "messageAuthenticationCode": "BASE64URL(Message Authentication Code)", // used in ECIES
        "ephemeralPublicKey": "Key in JWK format" // used in ECIES
      },
      ...
    ]
  }
}

```

## Design Considerations
1. The proposed schema takes inspiration from the General JWE JSON Serialization specification, however we are unable take a direct dependency on JWE because:

   a. JWE requires the encrypted data to be a part of the JWE object, the DWN deliberately decouples the data from the message which only contains the metadata.

   b. We could potentially encrypt the data encryption symmetric key in a JWE object as the `ciphertext`, but it means that we will be "double encrypting", coupled with use of ECIES, we will have a total 3 layers of symmetric key, which is an unnecessary overhead.


1. The `algorithm` property adopts values registered in the [JOSE registry](https://www.iana.org/assignments/jose/jose.xhtml) when there is a direct match.

1. The schema excludes the use of _Authentication Tag_ commonly used in symmetric encryption algorithm (e.g. `AES-GCM`) and specified in JWE, this is because the `dataCid` already serves the same purpose of message integrity verification. This is why `AES-CTR` is used instead.

1. The message handler will determine whether a records is encrypted or not implicitly by checking the existence of `encryption` property. 

1. The `authorization` property needs to contain the `encryptionCid` property so that a DWN cannot strip away the `encryption` property and still have the message considered to be valid, resulting in potentially irreversible corrupt state of a record if such message is replicated to all DWN instances of a DID.