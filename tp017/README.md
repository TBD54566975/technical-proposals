# TP17 Encryption for DWeb Nodes

```yaml
TP: 17
Title: Encryption for DWeb Nodes
Author(s): Daniel Buchner (@csuwildcat), Henry Tsai (@thehenrytsai)
Comments URI: https://github.com/TBD54566975/technical-proposals/discussions/3
Status: Accepted
Created: March 15, 2023
Updated: May 24, 2023
```

## Problem Statement

DWeb Nodes need a mechanism for the user and permitted external entities to create and read encrypted Records. The following proposal lays out options for doing so that span across all Record types (Protocols and flat object space).


## Proposal

1. Every DID would have an ECC public encryption key in its DID Document.
2. That key will be used as the HD (Hierarchical Deterministic) derivation master key to derive descendent keys for data encryption.
3. Instead of derivations using integer indices for the child key path segments, we derive keys using the string segments that can represent an arbitrary path in any logical hierarchical structure of the records. This is possible because HD KDFs (Key Derivation Function) take arbitrary bytes for path segments (people just typically use integers because they don't care about mapping to external content).
4. We can define multiple key derivation schemes to logically and hierarchically structure the records for the KDF to be applied against.

Results:
- Any outside participant can know exactly how to derive a public key for a given record using the recipient's public key and perform the ECIES encryption of its data.

## Key Derivation Schemes
We have identified a number of key derivation schemes that will be useful in different use cases:

1. `protocols` - allows decryption of records under a particular context ID

```javascript
const fullDerivationPath = [
  'protocols',
  protocol,
  contextId,
  ...protocolPathSegments,
  dataFormat,
  recordId
];
```

1. `protocolPaths` - allows decryption of records under a particular protocol path across all context IDs

```javascript
const fullDerivationPath = [
  'protocolPaths',
  protocol,
  ...protocolPathSegments,
  contextId,
  dataFormat,
  recordId
];
```

1. `schemas` - allows decryption of records under a particular schema

```javascript
const fullDerivationPath = [
  'schemas',
  schema,
  dataFormat,
  recordId
];
```

1. `dataFormats` - allows decryption of records under a particular data format

```javascript
const fullDerivationPath = [
  'dataFormats',
  dataFormat,
  recordId
];
```


## Data Encryption Scheme

It is probably advisable to look for an Elliptic Curve Integrated Encryption Scheme that works with secp256k1 (assuming we want to target that key type for our first MUST). ECIES typically uses the asymmetric ECC key to encrypt an AES symmetric key, the latter of which is used to encrypt the actual data. The following library may be a promising candidate for review: https://github.com/ecies/js

```javascript
import { encrypt, decrypt, PrivateKey } from 'eciesjs'
const k1 = new PrivateKey()
const data = Buffer.from('this is a test')
const result = decrypt(k1.toHex(), encrypt(k1.publicKey.toHex(), data)).toString()
console.log(result) // Returns: 'this is a test'
```

## Considerations

We will want to expand the supported key set, but for practical reasons we should probably pick one MUST implement key/alg combo and KDF scheme for DWN encryption features at the beginning.
