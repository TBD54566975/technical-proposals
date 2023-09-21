# TP 30: Delegates Interface

## Problem Statement
The private keys to a master DID are precious and should be kept safe. However, applications built on DWNs require messages to be signed frequently and on many devices. If the master DID private keys exist only on one device, it is onerous for remote DWNs to ping the master key holder to sign every single DWN message.

### Requirements
1. The master DID must be able to delegate a subordinate DID to sign messages on its behalf.
2. Other DWNs must be able to verify this delegation and check that it has not been revoked.

## Methods
1. Grant
2. Revoke
3. Check
  - Alternate names: StatusCheck, Status, Verify, Validate, Confirm, Audit
4. Register
  - Alternate names: Hook, Subscription

## Characters
For the sake of easier naming, let's define the following three DIDs who will appear throughout this TP.
1. *Alice* is a DWN owner who will be the *delegator*.
2. *Alice-ChatApp* is the app on Alice's phone that will be the *delegatee*.
3. *Bob* is the *counterparty* who wants to receive DWN messages signed by Alice-ChatApp with the understanding that Alice-ChatApp is representing Alice. Bob wants to be confident that Alice has not revoked Alice-ChatApp's grant, so he will keep his eyes peeled.

## Granting and Invocation
Alice creates a `DelegatesGrant` message with her own DID in the `delegatedBy` field and Alice-ChatApp's DID in the `delegatedTo` field.
When Alice-ChatApp sends a message to Bob invoking the grant, the `authorization` must contain both Alice-ChatApp's signature AND a reference to the `DelegatesGrant` message. The reference may take one of two forms: Either the entire `DelegatesGrant` message may be listed in the `delegatesGrant` field of the `authorization` OR the CID of the `DelegatesGrant` message may appear in the `delegatesGrantId` field of the `authorization`. The fields `delegatesGrant` and `delegatesGrantId` are mutually exclusive.

## Obtaining and Processing Grants
When Bob receives the message from Alice-ChatApp, either the message contains the entire grant in the `delegatesGrant` field OR it contains a CID of the grant `DelegatesGrant` with. In the former case, if Bob has never seen the grant before, he will process and store it in his DWN. In the latter case, he must already have the grant in his DWN.

## Revocation
### Phase 1: `DelegatesCheck`
When Bob authorizes a message with a `DelegatesGrant`, he will send a `DelegatesCheck` message to Alice's to check if the grant is still active. If the grant is active, Alice will say so. If the grant has been revoked, Alice will send the `DelegatesRevoke` message in the response along with a list of message CIDs to retain. To reduce network calls, Bob will cache the result of `DelegatesCheck`.

This first phase only requires `DelegatesGrant`, `DelegatesRevoke`, and `DelegatesCheck` to exist.

### Phase 2: `DelegatesRegister`
We build on Phase 1 to further reduce network calls. The first time Bob sees a given `DelegatesGrant`, he will send a `DelegatesRegister` to Alice, adding Bob to a list of entities who want to know when a grant is revoked. If the grant has already been revoked when Bob sends the `DelegatesRegister`, he will receive the `DelegatesRevoke` and retention list in the response. After Bob sends the `DelegatesRegister`, he will not need to ping Alice about revocation again. When Alice wants to revoke Alice-ChatApp's grant, she will get the list of DIDs who sent a `DelegatesRegister` for that grant and send the `DelegatesRevoke` to them. If Bob is not able to receive the `DelegatesRevoke`, he will likely know in advance, leading him to use the Phase 1 approach.

## Message structure
```ts
type DelegatesGrantDescriptor = {
  messageTimestamp: string;
  delegatedTo: string;
  delegatedBy: string;
}
type DelegatesGrantReply = {
  status: Status;
};

type DelegatesRevokeDescriptor = {
  messageTimestamp: string;
  delegatesGrantId: string;
  retentionListId: string; // CID of retention list sent in data payload
};
type DelegatesRevokeReply = {
  status: Status;
};

type DelegatesCheckDescriptor = {
  messageTimestamp: string;
  delegatesGrantId: string;
};
type DelegatesCheckResponse = {
  status: Status;
  revoke?: DelegatesRevokeMessage;
  retentionlist?: string[];
};

type DelegatesRegisterDescriptor = {
  messageTimestamp: string;
  delegatesGrantId: string;
}
type DelegatesRegisterResponse = {
  status: Status;
  revoke?: DelegatesRevokeMessage;
  retentionlist?: string[];
};
```

## Why isn't this part of the `Permissions` interface?
The two interfaces are conceptually distinct. `Permissions` is introverted; it controls whether an external entity can affect change on my DWN. `Delegates` is extroverted; it determines whether an external entity can affect change on other parties' DWNs in my name.

Consider if we added `PermissionsCheck`. Why would Bob care whether Alice-ChatApp can affect change on Alice's DWN? He is only interested in whether Alice-ChatApp should be allowed to affect change on his own DWN.

## Open Questions
1. Should `DelegatesCheck` always return the entire retention list? Should it take a `messageCID` in the request so Bob can check if a specific message is in the retention list, and Alice can just return a boolean? Does it matter?
2. Should `DelegatesRegister` return the revocation and retention list? Or is that unnecessary sugar? It seems useful and easy to implement to me.
