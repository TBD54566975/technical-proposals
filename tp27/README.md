# TP 27 - Data Encryption Scheme

TP: 27
Title: TP17 Data Encryption Scheme
Authors: Daniel Buchner (@csuwildcat)
Status: Implemented
Created: July 10, 2023
Updated: July 10, 2023

## Introduction

DWeb Nodes contain a mix of data, some that is intended to be private to the owner, some that is shared with a group of permissioned parties, and other data that is entirely public. To make this possible, the following technical proposal outlines a data encryption scheme that allows for a variety of encrypted data modes to serve the feature requirements of all application and permissioned relationship types.

## Requirements

The encryption scheme:

1. Must be compatible with the common key type(s) used with the target DID methods that are currently supported by DWNs.
2. Must support permissions that allow for:
    - Apps to have full management/access for a `protocol`
    - Apps and external entities to have full management/access for a `context` type
    - Apps and external entities to have full management/access for a `context` instance
    - External entities to have full management/access for a `schema` bucket in flat space
    - External entities to have full management/access for a `dataFormat` bucket in flat space

## Proposal

The following modifications will be made to areas of various DWN messages and their process, such as ProtocolConfigurations, Records, and Permissions, to facilitate encryption in a way that accounts for all the desired features and flows expected.

### ProtocolConfiguration Modifications

The following structure represents a `ProtocolConfiguration` modified to include encryption keys and metadata:

```javascript
{
  protocol: "example.com/protocol",
  types: {
    foo: {...},
    bar: {...},
    deez: {...},
    almonds: {...}
  },
  structure: {
    foo: {
      $encryption: {
        key: {JWK}
      },
      bar: {
        $encryption: {
          key: {JWK}
        }
      }
    },
    deez: {
      $encryption: {
        key: {JWK}
      },
      almonds: {
        $encryption: {
          key: {JWK}
        }
      }
    }
  }
}
```

#### `ProtocolConfiguration` Construction

1. The controller of the DWN owner's DID Document encryption key generates a hardened derived key for the protocol in question, then, from the root level of the `structure` downward, a hardened derived key for each of the nodes in the protocol graph (using the protocol path as an input).
2. These keys are placed in the protocol's `structure` at the node points corresponding to the protocol path they were generated for.
3. The keys will be located at the `key` field of the `$encryption` property, and shall be represented as a JWK object.

### Protocol Records

The following sequence is used to generate the wrapped keys required to support the full range of access and permission features envisioned for records:

1. A record creator must first fetch (or have cached) the ProtocolConfiguration for the protocol they seek to generate a record for.
2. The record creator generates a random symmetric key for the record, and uses it to encrypt the data payload of the record.
3. 
    - If the record is a context root: the record creator generates a hardened derived secp256k1 key for the record using a hardened, derived encryption key that descends from their DID Document encryption key, and uses it to encrypt the symmetric key. This allows them to decrypt it again in the future.
    - If the record is any descendant record: the record creator uses the context root record public key to encrypt the symmetric key they generated.
5. The record creator generates a wrapped key using the structural key corresponding to the protocol path of the record they are creating.
6. If present, the record creator generates a wrapped key for the `recipient` using the keys from the recipient's `ProtocolConfiguration` corresponding to the protocol path of the record type they are creating.
7. All of these wrapped keys are attached to the record prior to the conclusion of record construction/submission.

### Flat-Space Records

1. The record creator will generate a symmetric key for encrypting the record data.
2. If the owner is generating the record, they use their DID Document encryption key to generate two hardened derived keys using the `schema`+`dataFormat` and `dataFormat` (latter only if schema is not present) as inputs to the derivation. 
3. To grant permissions to another entity to generate flat-space records, the owner uses their DID Document encryption key to generate up to two hardened derived keys: one for `schema` if present, and one for `dataFormat`, but if `schema` is present, use the tuple `schema` + `dataFormat`for the `dataFormat` derivation. The private keys of these derived keys are then wrapped with the keys of the recipient of the permission.

### Access, Decryption, and Permissions

The scheme above affords the following capabilities:

1. Record creators can decrypt messages they send.
2. Record recipients can decrypt messages sent to them.
3. Permissions can be granted in descending fashion to the whole protocol, generic contextual roots, or any node in the graph (again, with access being descending in nature).

## Further Options & Considerations

1. We could introduce the concept of having to get the public key of the last message and generating a derivation from it, providing for the cascading access feature, but this seems quite complex to deal with as a dev.
2. Probably should align the access to key knowledge to match the permission leveled access.

