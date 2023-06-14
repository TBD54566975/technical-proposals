# TP 16 Creation vs Writing

```yaml
TP: 16
Title: Creation vs Writing
Author(s): Daniel Buchner (@csuwildcat)
Comments URI: https://github.com/TBD54566975/technical-proposals/discussions/1
Status: Draft
Created: March 13, 2023
Updated: March 13, 2023
```

## Problem Statement

It can be cumbersome/strange in dev UX terms to have to replay immutable properties of a record for subsequent Writes. The optimal setup would be to have immutable properties cast once, upon creation of a new record, and only be required to provide changes to mutable values you wish to modify in subsequent Writes.

## Considerations

For this propsal, create the following distinction in your mind regarding the properties that make up a record entry: 

Immutable props:

- Interface
- Method
- Protocol
- Schema
- dataFormat
- dateCreated
- parentCid

Mutable props:

- dataCid
- datePublished
- published
- encryption
- commitStrategy


## Proposal

The following are options for achieving the desired result based on the problem statement above:

### Option 1

This option introduces no new Record methods, and the only outward change for developers is that props/values will be split into two separate objects.

- Distinguish between a `RecordsWrite` that creates a record and one that overwrites a record by separating im/mutable properties into two areas: `descriptor` for immutable properties, and `parameters` for mutable properties.
- For a creating write, `descriptor` must be present with all the required immutable properties. For subsequent Writes, `descriptor` is not required, just `recordId`.
- The `parameters` object will be present whenever the developer intends to mutate something within that set of properties.
- The `recordId`, the `descriptor`, and the CID of `parameters` (if present) will always be signed over.
- Upon query, the implementation will retrieve the creation record to get the `descriptor` and current `parameters` bag and append them to the results object, along with the authorization JWS of the latest Write.

**Example**

```javascript
// Initial Write:

{
    recordId: '345ve657h6egwv5v5b765ht6',
    descriptor: {
        interface: 'Records',
        method: 'Write',
        schema: 'https://schema.org/Playlist',
        dataFormat: 'application/json'
    },
    parameters: {
        dataCid: 'f365vb54fe6v57b8n7t67bevw45'
        published: true
    }
}

// Subsequent Writes:

{
    recordId: '345ve657h6egwv5v5b765ht6',
    descriptor: {
        interface: 'Records',
        method: 'Write'
    },
    parameters: {
        dataCid: 'f365vb54fe6v57b8n7t67bevw45'
        published: false
    }
}

// Query result after subsequent Write above:

{
    recordId: '345ve657h6egwv5v5b765ht6',
    descriptor: {
        interface: 'Records',
        method: 'Write',
        schema: 'https://schema.org/Playlist',
        dataFormat: 'application/json'
    },
    parameters: {
        dataCid: 'f365vb54fe6v57b8n7t67bevw45'
        published: false
    }
}
```

### Option 2

This option introduces a new Record method: Create, which requires developers/implementers to do more to separate code paths and handling of their respective structures.

- `RecordsCreate` creates a record, while `RecordsWrite` overwrites any previous entry of a record post-Create. 
- Im/mutable properties are separated into two areas: `descriptor` for immutable properties, and `parameters` for mutable properties.
- For a `RecordsCreate`, `descriptor` must be present with all the required immutable properties. For subsequent Writes, `descriptor` only has the interface and method inside it, plus any Write-specific props (if there ever are any).
- The `parameters` object will be present whenever the developer intends to mutate something within that set of properties.
- The `recordId`, the `descriptor`, and the CID of `parameters` (if present) will always be signed over.
- Upon query, the implementation will retrieve the `RecordsCreate` to get the `descriptor` and current `parameters` bag and append them to the results object, along with the authorization JWS of the latest Write.

**Example**

```javascript
// Create:

{
    recordId: '345ve657h6egwv5v5b765ht6',
    descriptor: {
        interface: 'Records',
        method: 'Create',
        schema: 'https://schema.org/Playlist',
        dataFormat: 'application/json'
    },
    parameters: {
        dataCid: 'f365vb54fe6v57b8n7t67bevw45'
        published: true
    }
}

// Write:

{
    recordId: '345ve657h6egwv5v5b765ht6',
    descriptor: {
        interface: 'Records',
        method: 'Write'
    },
    parameters: {
        dataCid: 'f365vb54fe6v57b8n7t67bevw45'
        published: false
    }
}

// Query result after Write above:

{
    recordId: '345ve657h6egwv5v5b765ht6',
    descriptor: {
        interface: 'Records',
        method: 'Write',
        schema: 'https://schema.org/Playlist',
        dataFormat: 'application/json'
    },
    parameters: {
        dataCid: 'f365vb54fe6v57b8n7t67bevw45'
        published: false
    }
}
```

### Option 3

Just leave it as-is, and replay immutable props.