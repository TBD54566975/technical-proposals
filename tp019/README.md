# TP19 Subscription Functionality

```yaml
TP: 19
Title: TP19 - Subscription Functionality
Authors: Andor Kesselman (@andorsk)
Comments URI: https://github.com/TBD54566975/technical-proposals/discussions/6
Status: Proposed
Created: August 16, 2023
Updated: August 16, 2023
```

## Problem Statement

In the context of a Distributed Web Node (DWeb Node or DWN), the goal is to
provide a seamless and responsive user experience for both local and remote
subscriptions. This involves enabling users to subscribe to DWN instances,
whether owned by themselves or by others.

**Example:**

Consider a chat application utilizing DWeb Nodes. When a user sends a message to
a DWeb Node, all participants should receive notifications about the new message
and be able to retrieve a unique recordId associated with the newly created
record. This should have minimal latency, and be handled by a "push" rather than
"pull" mechanic.

This problem aims to ensure that subscribing to both local and remote DWeb Nodes
is user-friendly and functional, facilitating real-time interactions and data
retrieval.

## Proposal
