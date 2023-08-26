# TP26 Permissions Interface

```yaml
TP: 26
Title: TP26 - Permissions Interface
Authors: Diane Huxley (@diehuxx)
Comments URI: TODO
Status: Draft
Created: June 7, 2023
Updated: June 7, 2023
```

## Context
From the DWN Spec:
> The Permissions interface provides a mechanism for external entities to request access to various data and functionality provided by a Decentralized Web Node. Permissions employ a capabilities-based architecture that allows for DID-based authorization and delegation of authorized capabilities to others, if allowed by the owner of a Decentralized Web Node.

`Permissions` complements the `Protocols` interface. Whereas the `Protocols` interface defines a declarative syntax for application domain models and implicit authorization, `Permissions` defines a pattern for expliticly requesting, granting, and revoking access to a specific set of DWN resources for a specific DID under some conditions.


## Life Cycle of a Grant
The lifecycle of a grant will generally follow these steps:
1. *Request*: (Optional) The external entity sends a `PermissionsRequest` with their DID in the `grantedTo` field.
2. *Grant*: The DWN owner or a delegate sends a `PermissionsGrant` with their DID in the `grantedBy` field and the external entity's DID in the `grantedTo` field.
    - The `PermissionsGrant` may contain the CID of a corresponding `PermissionsRequest`
3. *Authorize*: The external entity uses their grant by sending a DWN message(s) where the `authorization` has a `permissionsGrantId` property, which contains the CID of the relevant `PermissionsGrant` message.
4. *Revoke*: The grant expires or is revoked via a `PermissionsRevoke` message, which contains the message CID of the `PermissionsGrant`.

### Storage of Grants
There are three parties involved in this lifecycle who need to store a `PermissionsGrant` in their respective DWNs:
1. The exposed DWN, named in the `grantedFor` property. This is the DWN to which the grantee is given access.
2. The grantor, named in the `grantedBy` property. The grantor must retain the grant in order to create a `PermissionsRevoke` in the future.
3. The grantee, named in the `grantedTo` property. The grantee includes the grant in messages to the exposed DWN.
In most scenarios, the DWN belongs to the grantor. However, for delegated grants, the DWN owner and the grantor are separate entities, so separate fields are necessary to differentiate them.

Why is `grantedFor` needed? If `grantedFor` were not present, it would be strange for the grantor and grantee to store the `PermissionsGrant` in their own DWN. The grantor's and grantee's DWNs would have no way to discern that "this grant is meant to give grantee access to someone else's DWN, not to give someone else access to mine." One option is for the grantor and grantee to store the grant in their respective web5 wallets. Storing in their DWN is better though, because their DWNs have built-in message storage and validation.


## Scope and Conditions
The descriptors of `PermissionsRequest` and `PermissionsGrant` both contain the `scope` and `conditions` properties. `scope` is required, and `conditions` is optional. Some `scope` and `conditions` options may be relevant only for a few DWN message types.

The `scope` property sets the boundaries on what abilities the grantee has. `scope` specifies a single DWN interface and method as well as several optional fields. The abilities granted by the `PermissionsGrant` are the intersection of properties within `scope`. In other words, whenever a grant is used, the action taken must follow all of the rules outlined in the `scope`.

The `conditions` property describes certain features about messages which use the grant.

### Encryption
The `encryption` condition in a `PermissionsGrant` can be used to authorize messages that write data, such as `RecordsWrite`. `encryption` has three possible options. If `encryption === 'required'`, then the grant-authorized message must contain an `encryption` field, as specified in [TP18](https://github.com/TBD54566975/technical-proposals/pull/6). If `encryption === 'prohibited'`, then the grant-authorized message may not contain an `encryption` field. If `encryption` is absent or equals `optional`, then then the grant-authorized message may contain or omit `encryption`.

### Protocols
There are two ways an entity may gain access to a protocol record. First, the protocol definition may specify access to the actor. Second, the entity may have a `PermissionsGrant` with a scope that contains the record. In either case, an incoming `RecordsWrite` must conform to the protocol type and path laid out in the protocol definition.

For scopes specifying `interface: Records`, the `protocol` scope restricts a `PermissionsGrant` to reading, writing, querying, or deleting records within a single protocol. The `contextId` can further restrict record access within a protocol to a single context.

If `protocol` is specified with `interface: Protocols` and `method: Configure`, this allows an entity to configure a single protocol. If an entity has a grant allowing for `ProtocolsQuery`, the `protocol` scope will force the `ProtocolsQuery` to have a filter containing the granted `protocol` URI.

### Delegation
The `delegation` condition contains enough complexity that it deserves a separate technical proposal. There are mentions of delegates and delegation in this TP, but its nuance is not fully captured. For the curious, delegation allows the following scenario: if Alice gives Bob a grant with `delegation` enabled, then Bob can issue grants of the same or narrower scope to Carol.

### Type Definitions
See the type definitions below for a list of `scope` and `conditions` options. More options may be added in the future.
```typescript
type PermissionsScope = {
  interface: DwnInterface;
  method: DwnMethod;
  // May only be present when `interface` === 'Records'.
  // The grantee may access records with matching `schema`.
  schema?: string;
  // May only be present when `interface` is 'Records' or 'Protocols'.
  // The grantee may access protocols or records with matching protocol URI.
  protocol?: string;
  // May only be present when `interface` is Records and `protocol` is present.
  // The grantee may access records within a particular protocolPath of a protocol.
  protocolPath: string;
  // May only be present when `interface` is Records and `protocol` is present.
  // The grantee may access records within a particular context of a protocol.
  contextId?: string;
  // May only be present when `interface` === 'Records'.
  // The grantee may access a specific records with matching recordId.
  recordIds?: string[];
};

type PermissionsConditions = {
  // Determines whether the permissioned message must, may, or must not be signed.
  attestation?: 'required' | 'optional' | 'prohibited';
  // Determines whether a record must, may, or must not be signed.
  encryption?: 'required' | 'optional' | 'prohibited';
  // If true, the grantee may mark a record as published. If false or omitted, they may not.
  publication?: boolean;
  // If true, the grantee may write a subsequent `PermissionsGrant`, giving the same or narrower access
  // to another entity. If false or omitted, the grantee may not write issue their own `PermissionGrant`.
  // We use an enum instead of a boolean because we may add more options for delegation in the future.
  delegation?: 'prohibited' | 'allowed';
  // If true, the grantee may access any objects or data within the granted capabilities, including objects or
  // data that it has not created. If false or omitted, the the grantee may only access objects or data that it has created.
  sharedAccess?: boolean;
};
```


## Messages
### PermissionsRequest
By themselves `PermissionsRequests` do nothing. However, it is necessary for the DWN spec and its reference implementation to define them. `PermissionsRequest` serves as an "inbox" for requests.
2. We must standardize how one entity to requests access to another entity's resources. If we do not define this, we risk each app defining their own access-request pattern with no interoperability between apps.

### PermissionsGrant
`PermissionsGrant` explicitly gives an external entity the ability to send a DWN message within some `scope`; further restrictions are defined in the `conditions` property. To use a `PermissionsGrant`, the external entity must include the CID of the relevant `PermissionsGrant` in the `permissionsGrantId` field of each message's `authorization`. The grant is active until its `dateExpires` time or until a `PermissionsRevoke` is affected.

### PermissionsRevoke
Who can revoke a grant? In the immediate term, a grant can be revoked by the DID in the `grantedBy` field or by the DWN owner. In a separate TP about delegation we will elaborate on more complex answers to this question.

A `PermissionsRevoke` contains the CID of a `PermissionsGrant` message. If an incoming message contains this `permissionsGrantId` in its `authorization`, its timestamp must be after the timestamp of the `PermissionsGrant` and before the timestamp of the `PermissionsRevoke`.

### PermissionsQuery
A `PermissionsQuery` contains a filter and returns a list of active `PermissionsGrants`. For an empty `filter`, the response will contain all grants where the requester's DID is in the `grantedTo`, `grantedFor`, or `grantedBy`. The `filter` may also contain any field in the `scope` of a `PermissionsGrant`.

The filter may also contain `grantedTo`, `grantedFor`, or `grantedBy`.The DWN owner may pass any DID to these fields. For anyone else, if these fields are present, they must equal the DID of the requester, or the `PermissionsQuery` must contain a `permissionsGrantId` allowing them to query `

## Type definitions
```typescript
type PermissionsRequestDescriptor = {
  interface: 'Permissions';
  method: 'Request';
  dateCreated: string;
  // The DID of the DWN which the grantee will be given access
  grantedFor: string;
  // The recipient of the grant. Often this will be the author of the PermissionsRequest message
  grantedTo: {
    // Enum, currently with only one value "did".
    // This structure is extensible; we can add more parties to give grants to in the future, such as groups or participants.
    grantedToType: string;
    grantee: string;
  };
  // The granter, who will be either the DWN owner or an entity who the DWN owner has delegated permission to.
  grantedBy: string;
  // Optional string that communicates what the grant would be used for
  description?: string;
  scope: PermissionsScope;
  conditions?: PermissionsConditions
}

type PermissionsGrantDescriptor = {
  interface: 'Permissions';
  method: 'Grant';
  dateCreated: string;
  // Optional CID of a PermissionsRequest message. This is optional because grants may be given without being officially requested
  permissionsRequestId?: string;
  // Optional timestamp at which this grant will no longer be active.
  dateExpires?: string;
  // The DID of the DWN which the grantee will be given access
  grantedFor: string;
  // The recipient of the grant. Often this will be the author of the PermissionsRequest message
  grantedTo: {
    // Enum, currently with only one value "did".
    // This structure is extensible; we can add more parties to give grants to in the future, such as groups or participants.
    grantedToType: string;
    grantee: string;
  };
  // The granter, who will be either the DWN owner or an entity who the DWN owner has delegated permission to.
  grantedBy: string;
  // Optional string that communicates what the grant would be used for
  description?: string;
  scope: PermissionsScope;
  conditions?: PermissionsConditions
};

type PermissionsRevokeDescriptor = {
  interface: 'Permissions';
  method: 'Grant';
  // The CID of the `PermissionsGrant` message being revoked.
  permissionsGrantId: string;
}

type PermissionsQueryDescriptor = {
  interface: 'Permissions';
  method: 'Query';
  filter: PermissionsScope & {
    // The DWN owner may pass any DID to these fields.
    // For anyone else, if these fields are present, they must equal the DID of the requester.
    grantedTo?: string;
    grantedBy?: string;
    grantedFor?: string;
  }
}
```
