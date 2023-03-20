# TP17 Encryption for DWeb Nodes

```yaml
TP: 17
Title: Encryption for DWeb Nodes
Author(s): Daniel Buchner (@csuwildcat)
Comments URI: https://github.com/TBD54566975/technical-proposals/discussions/3
Status: Draft
Created: March 15, 2023
Updated: March 16, 2023
```

## Problem Statement

DWeb Nodes need a mechanism for the user and permitted external entities to create and read encrypted Records. The following proposal lays out options for doing so that span across all Record types (Protocols and flat object space).

### Considerations

We will want to expand the supported key set, but for practical reasons we should probably pick one MUST implement key/alg combo and KDF scheme for DWN encryption features at the beginning.

## Proposal

1. Every DID would have an ECC public encryption key in its DID Document
2. That key will be used as the HD derivation master key
3. Instead of derivations using integer indices for the child key paths, entities derive keys using the URIs strings of the Protocol and/or the schema/label path of the records, creating deterministic buckets that map directly to our Record objects. This is possible because HD KDFs take arbitrary bytes for index paths (people just typically use integers because they don't care about mapping to external content).
4. Keys are derived based on each objects logical 'bucket' in the virtual hierarchy that implicitly exists based on the Protocol URI and/or the schema/label paths that already latently exists within our system.

Results:
- Any outside participant can know exactly how to derive a public key for a given record and perform the ECIES encryption of its data.
- PermissionsGrants can grant access to the following 'buckets' of records:
  - All of DWN
  - All of a Protocol
  - Some subset of a Protocol's records, based on virtual path position, wherein access is granted to decendants.
  - All records with the same schema for records that *do not* have Protocol URIs.

### KDF Algorithm

The KDF will utilize the following pseudocode explains how one would generate a key based on the implicit tree structure of a record when deriving keys from the master public key present in the DID Document of the target DID:

```javascript
let key;
if (PROTOCOL_URI_PRESENT) {
  const path = PROTOCOL_URI + '/' + PROTOCOL_LABEL_PATH;
  if (!record.parentCid) { // dealing with root record in a Protocol
    const protocolKey = getKeyForProtocol(PROTOCOL_URI);
    if (protocolKey) {
      key = KDF(protocolKey, path);
    }
    else {
      const didDocumentKey = getKeyForDID(TARGET_DID);
      key = KDF(didDocumentKey, path);
    }
  }
  else {
    const parentKey = getKeyForParentRecord(record.parentCid);
    if (parentKey.path === path) {
      key = parentKey;
    }
    else {
      key = KDF(parentKey, path)
    }
  }
}
else {
  const path = record.schema;
  let schemaKey = getKeyForSchemaBucket(record.schema);
  if (schemaKey) {
    key = schemaKey;
  }
  else {
    const didDocumentKey = getKeyForDID(TARGET_DID);
    key = KDF(didDocumentKey, path);
  }
}
```

## Encryption Scheme

With the above KDF fleshed out, it is probably advisable to look for an Elliptic Curve Integrated Encryption Scheme that works with secp256k1 (assuming we want to target that key type for our first MUST). ECIES typically uses the asymmetric ECC key as the encryptor of an AES symmetric key, the latter of which is used to encrypt the actual data. The following library may be a promising candidate for review: https://github.com/ecies/js

```javascript
import { encrypt, decrypt, PrivateKey } from 'eciesjs'
const k1 = new PrivateKey()
const data = Buffer.from('this is a test')
const result = decrypt(k1.toHex(), encrypt(k1.publicKey.toHex(), data)).toString()
console.log(result) // Returns: 'this is a test'
```

## Open Questions

1. Do we want a tree-based scheme?

If we don't do tree-based, we would have to give out many more keys, because access would no longer flow downward from a logical record association perspective.

- DID Doc Key
  - Schema (if object has no Protocol URI)
  - Protocol URI (you can make buckets out of this)
    - Top-level Schema
      - Context ID
        - Schema-based sub-path
