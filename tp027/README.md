# TP27 RecordsCommit Interface for DWeb Nodes

```yaml
TP: 27
Title: RecordsCommit Interface for DWeb Nodes
Comments URI: test
Status: Draft
Created: July 21, 2023
Updated: July 21, 2023
```

## Problem Statement

DWeb Nodes need a mechanism for multiple writers to update the same record asyncronously.


## Proposal

Part of the original DWN specs, but needs to be implemented and updated:
https://identity.foundation/decentralized-web-node/spec/#recordscommit
https://identity.foundation/decentralized-web-node/spec/#if-the-message-is-a-recordscommit

Copying over relevent parts of the spec so that we could have discussions around them here.


1. Records will always begin with a `RecordsWrite`.
2. `RecordsWrite` MAY have the property `commitStrategy` which MUST be a string from the table of registered [Commit Strategies](https://identity.foundation/decentralized-web-node/spec/#commit-strategies)
3. Subsequent updates to the record MAY be done via `RecordsCommit` messages.
4. `RecordsCommit` messages MUST have a the property `recordId` and it MUST be the `recordId` of the logical record the entry corrosponds with.
5. `RecordsCommit` messages MUST have the property `commitStrategy` and it MUST match the `commitStrategy` of the parent `RecordsWrite` message.
6. `RecordsCommit` messages MUST have the property `parentId` and it MUST be the CID of the descriptor of the previous `RecordsWrite` or `RecordsCommit` ancestor in the record's lineage.
7. Subsequent `RecordsWrite` can be used to snapshot the data or change the `commitStrategy`


Clients will use `RecordsQuery` and the results will allow them to use a root `RecordsWrite` and a lineage of `RecordsCommit` to produce the latest state of the object.