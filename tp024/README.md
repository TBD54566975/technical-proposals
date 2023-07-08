# TP24 Participant Authorization in Protocols

```yaml
TP: 24
Title: TP24 - Participant Authorization in Protocols
Authors: Henry Tsai (@thehenrytsai)
Comments URI: https://github.com/TBD54566975/technical-proposals/discussions/8
Status: Draft
Created: May 23, 2023
Updated: May 23, 2023
```

## Problem Statement
We need a protocol mechanism for enabling multi-party communication such as group chat.

## Possible Approaches
Assuming we need to explicitly declare participants, there are a few ways to model it using `RecordsWrite`, using a chat protocol as an example:

1. A `participants` "container" record which is referenced by `participant` child records.

```mermaid
graph TD;
    Participants-->Thread1;
    Participant1-->Participants;
    Participant2-->Participants;
    Message1-->Thread1;
    Message2-->Thread1;
```

2. Direct `participant` child records without the "container". Pro: protocol rules can be reused for managing participants.

```mermaid
graph TD;
    Participant1-->Thread1;
    Participant2-->Thread1;
    Message1-->Thread1;
    Message2-->Thread1;
```

3. One `participants` record that contain the list of participants.

```mermaid
graph TD;
    Participants-->Thread1;
    Message1-->Thread1;
    Message2-->Thread1;
```


## Proposal
1. Use approach #2
1. When a participant is written, the `recipient` is populated with the participant's DID.
1. When performing protocol authorization, filter `participant` records to :

```javascript
    const filter = {
      interface    : DwnInterfaceName.Records,
      protocol     : 'http://chat-protocol.xyz',
      protocolPath : 'thread/participant',
      contextId    : incomingMessage.contextId,
      schema       : 'http://participant.tbd',
      recipient    : 'did:example:alice',
      dateCreated  : {
        lt  : incomingMessage.dateCreated,
      };
    };
```

Example protocol definition:

```json
{
  "protocol": "http://chat-protocol.xyz",
  "types": {
    "thread": {
      "schema": "http://thread.xyz",
      "dataFormats": [
        "application/json"
      ]
    },
    "message": {
      "schema": "http://message.xyz",
      "dataFormats": [
        "application/json"
      ]
    },
    "$comment": "Below is implied and need not be specified, but shown for demonstrating the intent",
    "$participant": {
      "schema": "http://participant.tbd",
      "dataFormats": [
        "application/json"
      ]
    }
  },
  "thread": {
    "$participant": {
      "$actions": [
        {
          "who": "participant",
          "of": "tread",
          "can": "write"
        },
        {
          "who": "participant",
          "of": "tread",
          "can": "query"
        },
        {
          "who": "participant",
          "of": "tread",
          "can": "delete"
        }
      ]
    },
    "message": {
      "$actions": [
        {
          "who": "participant",
          "of": "tread",
          "can": "write"
        },
        {
          "who": "participant",
          "of": "tread",
          "can": "query"
        }
      ]
    }
  }
}
```

## Open Questions
1. Should the `$participant` data be embedded directly within the message like the placement of protocol definition? If so, this would differ from the pattern used by a pure record currently.

1. How can we prevent a participant from modifying past records (especially `participant` records) after they are removed?

   Probably through the usage of the "snapshot" feature to be spec-ed in a separate Technical Proposal, which would be useful in general, as well as when revoking a permission.

