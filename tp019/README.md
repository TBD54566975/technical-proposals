# TP19 Pub/Sub 

```yaml
TP: 19
Title: TP19 - Subscription Functionality
Authors: Andor Kesselman (@andorsk)
Comments URI: https://github.com/TBD54566975/technical-proposals/discussions/6
Status: Drafting
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


```mermaid 
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
graph TD
  subgraph Thread
      subgraph messages 
        subgraph existingMessages
         A1(A1)
         A2(A2)
         A3(A3)
        end
         A4(A4)
      end
      subgraph Participants
          Alice(Alice)
          Bob(Bob)
          Charlie(Charlie)
          Mallory(Mallory)
      end
  end
  
  Charlie -->|sends message| A4
  A4 .->|notifies| Alice
  A4 .->|notifies| Bob
  A4 .->|notifies| Mallory
  ```

### Modeling the Problem

We define $N$ as the nodes in the network. $C$ is an attached context for which each record $r_n$ in the set of records $\{r_0, r_1, ..., r_n\}\in{R}$ for which $N$ is subscribed to.

We subset the spaces into two disjoint sets. $R_0$, which contains all synced events to a DWN context and $R_1$, which contains the set of new incoming events not yet synced. We define the $latency$ or $l_n$ as the time between the first write event to $C$ for $r_n$ and the time it takes to propogate the write to the rest of the thread contexts. 

Unlike normal sync, which is scoped to improve latency for $R$, we confine the problem statement to only reducing latency for $R_1$.

The goal of the subscriptions is to minimize $l$ for the set $R\prime$ such that the $lim_{l\to{0}}R$, contrained for the user $C$. In aggregate across a network, a histogram of the latency values can be constructed to understand the performance over $N$.

### Prior Art

This section describes some prior art on decentralized pub/sub systems. 

* [SCRIBE: A large-scale and decentralized
publish-subscribe infrastructure](https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=eea6ef05a309cccbfe76806ab9f4f2b212f3264c) : Scribe creates a multi-cast tree similar to  to reverse path forwarding and traverses it via Pastry APIs. This is very useful for looking at some of the potential APIs required, as well as some architectural considerations. 
* [Decentralized Message Ordering for
Publish/Subscribe Systems](https://link.springer.com/chapter/10.1007/11925071_9) Message ordering at scale. Tasks are distributed across sequencing atoms. 
* [
Decentralized Publish-Subscribe System to Prevent Coordinated Attacks via Alert Correlation
](https://link.springer.com/chapter/10.1007/978-3-540-30191-2_18) This was less considered, but worth looking into down the road. 
* [UnifiedPush: a decentralized, open-source push notification protocol](https://f-droid.org/2022/12/18/unifiedpush.html) A reference architecutre of hybrid decentralized. Coordination servers are required, but open. 
* [Analysis for flow termination in the decentralized and centralized pre-congestion notification architecture](https://www.sciencedirect.com/science/article/pii/S1389128623003213) Also not heavily considered in this, but possibly a useful reference for later.
* [Decentralized Edge-to-Cloud Load Balancing:
Service Placement for the Internet of Things](https://eprints.whiterose.ac.uk/162348/1/09418552.pdf) : Multi-agent load balancing. Useful for later. 


## Higher Level Interaction

As an example of a proposed higher level 1:1 flow between Alice and Bob for subscriptions.

**1:1 Subscription Event**

``` mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
sequenceDiagram
  participant Alice
  participant Bob
  
  Bob -->> Alice : Bob requests to subscribe to Alice's context. Implicitly requires read access.
  note right of Alice: Alice grants Bob subscription access ( implicit read )
  Bob -->> Alice : Bob writes to Alice's DWN context.
  note right of Alice : Alice listens to event
  note right of Alice: Alice handles event

  Alice -->> Bob : event pushed to Bob's DWN subscription handler
  note left of Bob: Bob handles subscription event
```

**Multi-Party Subscription Event**

In this case, all new messages must be broadcasted across multiple parties. In this interaction diagram Alice multiplexes new messages on the thread registered subscriptions. 


``` mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
sequenceDiagram
  participant Alice
  participant Bob
  participant Charlie

   Bob -->> Alice : Bob requests to subscribe to Alice's context. Implicitly requires read access.
   note right of Alice: Alice grants Bob subscription access ( implicit read )

   Charlie -->> Alice : Charlie requests to subscribe to Alice's context. Implicit read access.
    note right of Alice: Alice grants Charlie subscription access ( implicit read 
    
  Bob -->> Alice : Bob writes to Alice's DWN context.
  note right of Alice : Alice listens to event
  note right of Alice: Alice handles event
 
  Alice -->> Bob : event pushed to Bob's DWN subscription handler
  note left of Bob: Bob handles subscription event
  Alice -->> Charlie : event pushed to Bob's DWN subscription handler
  note left of Charlie: Charlie handles subscription event
```

As you can see, in this proposed higher level flow, we have two main operations:

1. **Registration of a Subscription with a particular context**: This is a $O(1)$, for each subscription participant, $O(N)$ for the set of subscribers. 
2. **Notification on updates to a subscription**: The owner of $C$ must run notification operations at a $O(N)$, linearly scaling with the # of subscription nodes in in a context. Over the duration of a context, the number of notification operations are $O(N * len(R_1))$, or the number of nodes * the length of $R_1$ over time.

Finally, to close the loop, not diagramed but worth noting for the complete life cycle, would be the death of a subscription, either by revocation or by deletion.

## Technical Considerations and Questions

* Without a central coordinator, how do we propogate update events efficently to each DWN? 
* How to handle correctly unavailable DWNs?
* How can we re-use as filters across repositories to avoid redundancy and allow
  for better user experience?
* What is the right way to describe the subscription api?
* How do we insert code at the point of behind validation?
* Default behavior if no did is specified? 
* How do we ensure users can only subscribe to the right options?
* How do users control the subscriptions? What happens if there is a node with millions of subscriptions?
* How do we handle unsubscriptions?

## Proposal

I will break up the problem into three parts:

1.  **Listening**: How does DWN user listen to activity in DWN bound by a contextId? 
2. **Notification**: How does DWN user notify another DWN of $e$?  
3. **Propogation**: How can $e$ be efficiently propogated through a network of subscribers : $S$ 

To reduce complexity, we will assume that the `author` of $C$ is also responsible for delegating the strategy for notifications. 

Technically, the author will install a subscription $s_i$ in $C$ which is responsible for managing subscriptions. Subscription objects will be managed in separate special partition on a DWN. They will be mapped to a lookup, `contextID: s_i`, which that the key is based on the contextID. On an event $e_i$ to a context $C$, a lookup into the subscription partition is made. If it exists, subscription object is activated. To note: this will only activate with *forward* events. I.e it is not built to back propogate to old events. 

To Discuss: DWN Hooks 

### Models

```mermaid 
classDiagram
  class subscription {
      callback
  }
  
  class subscriptionLookup {
    <<Dictionary>>
    - dict: Map~string, Map string: Object~
  }
  
  class hooks {  
    onCreate()
    onDelete()
    onUpdate()
    onRead()
  }
  
  class dwn {
    hooks
    +subscribe(contextId)
    +unsubscribe(contextId)
  }
  
  class context
    
  subscription .. dwn : installed on
  subscription "n".."1" context  : subscriptions are bound by context
  
```

## Propogation Strategies

In this section, we define various propogation strategies for notification. As described above,the we define the `author` of $C$ synonymous with the `publisher` of $C$. The `author` of $C$ is responsible for managing the propogation strategies of subscriptions. 

### Simple Case : Multiplexing Strategy

In this case, the author of $C$ is designated as the "publisher" of C, and thus is soley responsible for distributing updates from their DWN over a multiplexing strategy.

```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
graph TD
  publisher
  publisher --> A
  publisher --> B
  publisher --> C
  publisher --> D
  publisher --> E
  publisher --> F
  publisher --> G
```
**Advantages**

* Simple to implement
* Permissions are easier because relationship is already assumed between publisher and subscriber. 
* Coordination is easier.  

**Disadvantages**

* Doesn't scale as well, as one DWN is responsible for $O(N * R)$ interactions.

### Delegation Propogation

In the below propogation strategy, the publisher may delegate notifiations downstream to the network. 

Neighbors may be chosen in a number of ways: 

1. Predetermined neighbors chosen by publisher
2. Dynamic Discovery : Requires knowing who the participants of the subscription are. 

```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
graph TD
  publisher
  publisher --> A
  publisher --> B
  A --> C
  A --> D
  B --> E
  B --> F
  F --> G
```

**Advantages**

* Will scale better
* More resiliant
* Can be optimized around latency 

**Disadvantages**

* Requires more coordination to avoid duplication of notifications.
* Requires interaction bteween two unrelated DWN's, which in cases may not be desirable. 
* Strategy required when node in the middle is down.  

## Notification Strategies

The following section describes how notifications can be delivered to the DWN endpoint via publishing. 

### Web Sockets

In this case, the publisher and the suscriber hold an active socket connection between the two parties. The publisher can now push via web sockets on a topic to the subscriber. 

```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
graph TD
    publisher -->|socket connection| subscriber
```

```mermaid 
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
sequenceDiagram
  participant publisher
  participant subscriber
  
  subscriber ->> publisher : subscribes to topic C. listens for message
  note right of publisher: update to C happens
  publisher ->> subscriber : sends push to subscriber topic. 
  note left of subscriber: implements callback
```

Example message: 

```
{
  recordId: <>.
  contextId: <>,
  action: "create"
}
```


**Advantages**

* Minimal operations. As the network scales, could be problematic. 
* Fast ( nearly real time ) 


**Disadvantages**

* Requires active connections
* Would need additional operations to handle backlogging.


### DWN Notifications

An alternative strategy involves writing to DWN's into a cached "queue", which allows for $n_i$ to pick up unresolved messages off the queue and interpret them. It would be a short term cache, and messages would eventually delete if not pulled off.  

```mermaid 
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
graph TD
  subgraph Alice 
      dwn
      subgraph Subscription Cache
          subgraph Context
              s_1
              s_2
          end
      end
  end
 
  subgraph Bob 
      dwn2[dwn]
      workers[subscription workers]
      subgraph New Messages
          queueAPI --> m1
          subgraph New Message Queue
              m1
              m2
          end
      end
      m1 -->|pull new messages off context and work| workers
  end
  
  dwn2 -->|subscribes| dwn 
  dwn --> |registered| s_2
  dwn --> queueAPI
```

**Advantages**
* Doesn't require actively maintaining connections

**Disadvantages**
* Needs to do more ops per event. Write, pull, etc. 
* Cache can fill over time. 

### Local Notifications

We now need to think about the case where one DWN gets updated, but is synced to multiple DWNs. 

```mermaid 
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
graph TD
  DWN1(dwn 1)
  DWN2(dwn 2)
  DWN1 --- |sycned| DWN2 --- |synced| DWN3(dwn 3)
  
  SubscriptionNotice(Subscription Notice) -->|dwn2 gets subscription notice| DWN2 
 
```
In this case, DWN2 MUST resolve propogation against registered syncs for local DWN. This can overlap with existing sync (every 2 min), OR send a push request to the other DWN's with a tied contextID to force the DWN to specifically sync against a particular context.



## Interfaces

### web5-js API updates

Expose the following API with the following options to web5-js.

```typescript
type SubscribeOptions = {
  filter: RecordsQueryFilter // take in a records query filter
  did?: string // target did to subscribe to
}

web5.dwn.subscribe(opts: SubscribeOptions, async (message) => {
  // TODO: add stuff here for callbakc handling
})
```

## Subscription Hooks

In the context of the web hook paradigm, during an event update, only the
'recordId' is shared with Bob. The primary processing responsibility lies with
Alice. To ensure scalability, Alice will likely need to implement a queuing
mechanism for effectively routing signals to various DWN's.

Here's a clearer version of the steps you provided:

1. Bob initiates a request to establish a subscription on Alice's DWN for a specific context.
2. Alice has the option to either approve or reject the request.
3. If the request is approved, Alice proceeds to install the subscription onto
   her DWN. She then notifies Bob about the approval.
4. Subsequently, Bob sets up the subscription on his DWN. This subscription is
   designed to monitor a specific contextual event and trigger a callback
   function whenever a new write event occurs within that context.
5. The subscription is closely associated with a particular context within the
DWN. Any event, along with its associated sub-events, is managed through a
callback. If not customized, the default behavior of the callback is to write
the event to Bob's DWN.

``` mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
sequenceDiagram
  participant Bob
  participant Alice
  
  Bob -->> Alice : installs subscription to Alice's DWN and Context
  note right of Bob: Bob gives Alice write access to Subscription context
  note right of Bob: Bob installs subscription listener to context with callback function.
  note left of Alice : Alice registers subscription on DWN
  note left of Alice : DWN event occurs
  Alice -->> Bob : Alice sends event note to Bob with record id and context
  Bob -->> Alice: Bob handles callback to new event.
```

The following shows the high level architecture for subscription

```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }
}%%
classDiagram

  class Web5JS {
    + subscribe(target, contextId, options, callback) // callback installed locally
    + unsubscribe(target, contextId, options) // callback installed locally
  }
  
  class dwn {
    - publish(contextId, options)
  }
  
  class InstallSubscriptionResponse {
    string subscriptionId
  }

  class SubscriptionAPI {
    - handlers map<string, subscriptionHandler> 
    - install(request InstallSubscriptionRequest)
    - uninstall(id string)
  }
```

### DWN Server Updates

Insert code in
[handleDWNProcessMessage](https://github.com/TBD54566975/dwn-server/blob/main/src/json-rpc-handlers/dwn/process-message.ts#L9).
Message processing events should be sent to a subscription queue of some sort. 

```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#5097E6',
      'background': '#fff',
      'primaryTextColor': 'black',
      'primaryBorderColor': 'black',
      'lineColor': 'black',
      'secondaryColor': 'white',
      'tertiaryColor': '#fff'
    }
  }%%
graph TD
    subgraph Server
    
    end
    subgraph Client
    
    end
```

### DWN-SDK-JS Updates


Insert code somewhere around the [processMessage](https://github.com/TBD54566975/dwn-sdk-js/blob/main/src/dwn.ts#L93C19-L93C19)

Similar behavior to DWN Server
