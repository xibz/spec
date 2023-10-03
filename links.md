# Connecting Events - Links Proposal

## Abstract

This proposal will outline how to connect individual CDEvents to one other.
Currently there's no way of associating events without needing to backtrack
across certain context attributes, e.g.
[id](https://github.com/CDEvents/spec/blob/main/spec.md#id-context).  While
this does give us the ability to construct some graph, we do not know when a
particular chain starts or finishes.

This proposal will outline a new approach that will allow for connecting
CDEvents and give a clear distinction of when an activity chain starts and
finishes.

## Semantics

This section will define various terms to ensure there are no assumptions being
made when we talk about linking events

* **CI**        - [Continuous integration](https://bestpractices.cd.foundation/learn/ci/)
* **CD**        - [Continuous delivery](https://bestpractices.cd.foundation/learn/cd/)
* **Chain**      - A chain is an end to end representation of all activities performed
  in the CI/CD life cycle of some entity
* **Link**      - A link is an object containing relations between two CDEvents.

## Goals

The biggest challenge in this whole process is ensuring that connected events
can be retrieved quickly and efficiently, while also providing the necessary
metadata and structure to give a detailed picture of how things occurred.

1) Provide a way of quickly retrieving all related links
2) Keep link data structured small and simple
3) Scalable

## Use Cases

This section will go over a few use cases and explain how this approach can be
used to solve for each particular case.

### 1. Fan Out Fan In

The fan out fan in use case is an example where a system may make parallel
requests (fan out), and merge back to some other or the very same system (fan in)

Let us assume we have 3 system in our CI/CD environment. A continuous
integration environment, which we will call CI system, that runs tests and
builds artifacts, an artifact store that receives artifacts from the CI system,
and lastly the CD system which consume these artifacts shown in the diagram
below.

```
                    +-------------------+   +-----------------+
                +-->|   Build for Mac   |-->|   Test on Mac   |---+
+-----------+   |   +-------------------+   +-----------------+   |
|  Static   |   |   +-------------------+   +-----------------+   |   +---------------------------+
|  Code     |---+-->|   Build for SLES  |-->|   Test on SLES  |---+-->| Promote & Deliver Release |
|  Analysis |   |   +-------------------+   +-----------------+   |   +---------------------------+
+-----------+   |   +-------------------+   +-----------------+   |
                +-->| Build for Windows |-->| Test on Windows |---+
                    +-------------------+   +-----------------+
```

The diagram above shows the CI event creating 3 separate artifacts that will
make it's way to the artifact store. Some CD system would then consume those
artifact, but would need to group all the artifact when the CI system finishes
its pipeline. This is not meant to be a full diagram of the whole CDEvents
flow, but a simple representation of the artifact(s) life cycle.

### 2. Generic UI

With the goal of interoperability, this allows for compatible standalone
generic UIs for any CI/CD system.

## Design

This section will propose two designs. The first being how individual events
can be linked from a CDEvents perspective and define how links will look. The
second portion will address the goal of scalability.

## Connecting Events

An individual event usually has some relation to some trigger. This can be a
new commit added to a pull request, a test suite being called in the CI
process, or publishing a new artifact for some system to consume. While these
events mean something themselves, they do not give the proper context of what
caused what. This section will introduce a new field within the CDEvents
context that will allow for giving some relation between CDEvents.

```json
{
  "context": {
    "version": "0.3.0-draft",
    "id": "505b31c2-8bc8-47b3-a1a0-269d7a8530ac",
    "source": "dev/jenkins",
    "type": "dev.cdevents.testsuite.finished.0.1.1",
    "chain_id": "00000000-0000-0000-0000-000000000001", # new chain id field
    "timestamp": "2023-03-20T14:27:05.315384Z"
  },
  "subject": {
    "id": "MyTestSuite",
    "source": "/tests/com/org/package",
    "type": "testSuite",
    "content": {}
  }
}
```

The `chain_id` is an ID that will be generated when a new CDEvent chain is
wanted or if no CDEvent chain is present. This ID will follow the
[UUID](https://datatracker.ietf.org/doc/html/rfc4122) format. Chain IDs will
serve as a bucket for all CDEvents with some sort of relation to each other.

## Propagation

Chain propagation will be handled differently depending on the protocol that is
used. This proposal will not cover the technical details but will cover various
use cases that can occur in large systems.

### Partial CDEvent Support

The first case is where only a subset of the systems support CDEvents.
For this example, we will use four services. Service A, B, and C.

Services A, B, and D support CDEvents, while C does not. An example CDEvent
chain may look like:

```
  +-------------------------------------------------+
  |                     Event Bus                   |
  +-------------------------------------------------+
       ^           |                           ^
       |           v                           |
  +---------+ +---------+   +---------+   +---------+
  |    A    | |    B    |-->|    C    |-->|    D    |
  +---------+ +---------+   +---------+   +---------+
```

From the example above we can see that persisting a CDEvent event across A and
B can be easily done as they both support CDEvents. However, that propagation
is broken when B talks to C.

### Different Protocols

As mentioned before, propagation is up to the protocol, e.g. HTTP(S), MQTT,
AMQP, etc. A full list can be found [here](https://github.com/cloudevents/spec#cloudevents-documents).
Protocols that handle propagation differently should rely on the SDKs to
consume and continue the propagation.

### Separate Event Buses

Another situation which occurs when there exists multiple event buses and
sharing occurs due to consumer(s).

```
# Case 1: Different Event Busses

+---------------+   +---------------+   +---------------+
|  Event Bus A  |   |  Event Bus B  |   |  Event Bus C  |
+---------------+   +---------------+   +---------------+
             ^ |     ^ |         ^ |     ^ | 
             | v     | v         | v     | v 
            +-----------+       +-----------+
            | Service A |       | Service B |
            +-----------+       +-----------+

# Case 2: Possible Duplicate Consumed Events

+---------------+   +---------------+
|  Event Bus A  |<->|  Event Bus B  |
+---------------+   +---------------+
             ^ |     ^ | 
             | v     | v 
            +-----------+
            | Service A |
            +-----------+
```

This proposal will not tackle these problems, but identify them as issues that
the technical design document will need to solve.

For case 1, introducing a links service becomes much more difficult. If a user
is to query for a full chain since the link data may be persisted in different
databases.

Case 2 is more about consuming duplicate events and sending duplicate events
which can cause a potential issue around how links represents those
duplications.

## Links Spec

So far we have only talked about what a service may receive when consuming a
CDEvent with links. However, when we store a link, there's a lot more metadata
that can and should be added.

The idea is we would expect users to start links, groups, and end links
accordingly through APIs we would provide. This is very similar to how tracing
works in that the individual services dictate when a tracing span starts and
finishes.  Granularity in tracing is completely up to the engineer which this
proposal also intends users to do.

This will introduce new APIs to the CDEvents SDKs to handle automatic injection
of links or allow users to provide their own definition of what a CDEvent chain
is.

The link spec definition will look like this

```json
{
  "chain_id": "05CEDE1F-8C13-4699-80C3-B4AC6468414E",
  "link_type": "RELATION",
  "timestamp": "2023-03-20T14:27:05.315384Z",
  "from": {
      "id": "F27E36A4-5C78-43C0-840A-52524DFEED03",
      "event_kind": "ARTIFACT"
  },
  "to": {
      "id": "F004290E-5E45-45F4-B97A-FA82499F534C"
  },
  "tags": [
    "ci.environment": "prod"
  ]
}
```

| Name            | Description                                                        |
|-----------------|--------------------------------------------------------------------|
| chain_id   | This represents the full lifecycles of a series of events in CDEvents   |
| link_type  | An enum that represents the type of link, e.g. 'TO', 'FROM', 'GROUP'    |
| event_kind | An enum that represents a category of what the event is associated with |
| from       | Where the link is coming from                                           |
| to         | Where the link is going to                                              |
| tags       | Custom metadata that an individual link can have                        |

Below defines a list of `link_type` enums a kind can have

| Name     | Description                                                                               |
|----------|-------------------------------------------------------------------------------------------|
| START    | The starting CDEvent in a chain                                                           |
| END      | The ending CDEvent in a chain                                                             |
| RELATION | When a link has a relation, it can describe that relation with the `to` and `from` fields |
| GROUP    | When a link is to be grouped with other events, it will use GROUP to establish a grouping |

Below defines a list of `event_kind` enums a link can have

| Name     | Description         |
|----------|---------------------|
| ARTIFACT | An artifact CDEvent |
| BUILD    | An artifact CDEvent |
| CD       | A CD event          |
| CI       | A CI event          |
| TEST     | A test CDEvent      |

These links can be, but are not limited to being, sent when a CDEvent has
completed: to some collector, the link service, or the event bus . Further
the link service will allow for tagging of various metadata allowing for users
to specify certain labels when looking at links.

Some users may prefer to not run a separate links service especially if they
know their overall flow may only contain a few links. If that is the case,
simply turning on link payload aggregation, will send all links in the
payload. Mind you, this can make the payload very large, but may be good for
debugging.

The chain ID header will continue to propagate, unless the user explicitly
starts a new CDEvent chain. If there is no chain ID, the client will generate
one and that will be used for the lifetime of the whole events chain

Further, links are always constructed from the consumer point of view, with the
exception being the start and stop links. Allowing for start and stop to be
sent immediately, as opposed from the consumers point of view, allows for
single node events and/or allowing for UI systems to display as events are
occurring. This means when an event is consumed, the consumer will send a link
connecting itself to the parent event, as the consumer will know its context id
which a parent event would not be aware of. Wneh I care.

```
+-----+      +-----+      +-----+                                                                +--------------+         +-----------+
| Git |      | CI  |      | CD  |                                                                | Link Service |         | Event Bus |
+--+--+      +--+--+      +--+--+                                                                +-------+------+         +-----+-----+
   |            |            |           #1 (send change merged event)                                   |                        |
   +----------------------------------------------------------------------------------------------------------------------------->|
   |            |            |           #2 (source change link to start chain)                          |                        |
   +----------------------------------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |   #3 (proxy link #1)   |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #4 (receive change merged event)                                |                        |
   |            |<----------------------------------------------------------------------------------------------------------------+
   |            |            |           #5 (send pipeline queued event)                                 |                        | 
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |           #6 (source change links connecting #1 -> #5)                    |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |   #7 (proxy link #6)   |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #8 (send pipeline started event)                                |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |           #9 (pipeline queued links connecting #5 -> #8)                  |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #10 (proxy link #9)   |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #11 (send build queued event)                                   |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |           #12 (pipeline started links connecting #6 -> #9)                |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #13 (proxy link #12)  |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #14 (send build started event)                                  |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |           #15 (build queued event links connecting #11 -> #14)             |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #17 (proxy link #16)  |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #18 (send build completed event)                                |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |           #19 (build started event links connecting #14 -> #18)           |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #20 (proxy link #15)  |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #21 (send pipeline finished event)                              |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |           #22 (build completed event links connecting #18 -> #21)         |                        |
   |            +---------------------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #23 (proxy link #22)  |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #24 (receive pipeline finished event)                           |                        |
   |            |            |<---------------------------------------------------------------------------------------------------+ 
   |            |            |           #25 (send pipeline queued event)                                |                        |
   |            |            +--------------------------------------------------------------------------------------------------->|
   |            |            |           #26 (pipeline finished event links connecting #21 -> #25)       |                        |
   |            |            +--------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #27 (proxy link #26)  |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #28 (send pipeline started event)                               |                        |
   |            |            +--------------------------------------------------------------------------------------------------->|
   |            |            |           #29 (pipeline queued links connecting #25 -> #28)               |                        |
   |            |            +--------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #30 (proxy link #29)  |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #31 (send environment created event)                            |                        |
   |            |            +--------------------------------------------------------------------------------------------------->|
   |            |            |           #32 (pipeline started links connecting #28 -> #31)              |                        |
   |            |            +--------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #33 (proxy link #32)  |
   |            |            |                                                                           |<-----------------------|
   |            |            |           #34 (environment created end link for #31)                      |                        |
   |            |            +--------------------------------------------------------------------------------------------------->|
   |            |            |                                                                           |  #32 (proxy link #31)  |
   |            |            |                                                                           |<-----------------------|
```

This show's an example of how these different types would be used in a CI/CD
setting, but is not the only architecture.

### Payloads

This section will describe the first few sequences in the sequence diagram to
help explain the overall flow using payloads from CDEvents.

1. This is our very first event to the start of our CI/CD chain. This event
   would have been sent from some source management tool like Github, Gitlabs,
   etc.

```json
{
  "context": {
    "version": "0.4.0-draft",
    "chain_id": "D0BE0005-CCA7-4175-8FE3-F64D2F27BC01",
    "id": "38A09112-A1AB-4C26-94C4-EDFC234EF631",
    "source": "/event/source/123",
    "type": "dev.cdevents.change.merged.0.1.2",
    "timestamp": "2023-03-20T14:27:05.315384Z"
  },
  "subject": {
    "id": "mySubject123",
    "source": "/event/source/123",
    "type": "change",
    "content": {
      "repository": {
        "id": "cdevents/service",
        "source": "https://github.com/cdevents/service/pull/1234"
      }
    }
  }
}
```

Something to call out here is that the `chain_id` may have been `null`, for
whatever reason, prior to this event. This means any parents to this event did
not generate a `chain_id`. When an event is sent, it is important that the
sender generates this id.

2. We send the start link to let the links service know that we are creating a
   new chain.

```json
{
  "chain_id": "D0BE0005-CCA7-4175-8FE3-F64D2F27BC01",
  "link_type": "START",
  "timestamp": "2023-03-20T14:27:05.315384Z",
  "id": "38A09112-A1AB-4C26-94C4-EDFC234EF631" # context.id of #1
}
```

3. Event bus proxies the link payload from `#2` to the links service.

4. Shows some consumer consuming `#1` to do some action.

5. The CI system will queue a pipeline execution, and will generates a
`context.id` to be sent

```json
{
  "context": {
    "version": "0.4.0-draft",
    "chain_id": "D0BE0005-CCA7-4175-8FE3-F64D2F27BC01",
    "id": "AA6945F8-B0F1-48DD-B658-25ACF95BD2F5",
    "source": "/event/source/123",
    "type": "dev.cdevents.pipelinerun.queued.0.1.1",
    "timestamp": "2023-03-20T14:27:05.315384Z"
  },
  "subject": {
    "id": "mySubject123",
    "source": "/event/source/123",
    "type": "pipelineRun",
    "content": {
      "pipelineName": "myPipeline",
      "url": "https://www.example.com/mySubject123"
    }
  }
}
```

6. As the send change merge event is sent, the system will follow up with
sending a link associated with the prior event which connects `#1` to `#5`

```json
{
  "chain_id": "D0BE0005-CCA7-4175-8FE3-F64D2F27BC01",
  "link_type": "RELATION",
  "timestamp": "2023-03-20T14:27:05.315384Z",
  "from": {
      "id": "38A09112-A1AB-4C26-94C4-EDFC234EF631" # context.id of #1
  },
  "to": {
      "id": "AA6945F8-B0F1-48DD-B658-25ACF95BD2F5" # context.id of #5
  },
  "tags": [
    "ci.environment": "prod"
  ]
}
```

7. The event bus will then forward link `#6` to the links service

34. Paylods from `8-33` are very similar to all prior payloads shown here, but
    the last sequence is the ending link.

```json
{
  "chain_id": "D0BE0005-CCA7-4175-8FE3-F64D2F27BC01",
  "link_type": "END",
  "timestamp": "2023-03-20T14:27:05.315384Z",
  "id": "7D5E011F-5073-44A7-B4F0-86DD7D4C2C7F" # context.id of #31
}
```

### Separate Links vs Within CDEvents

This will go over the design choice of having links as a separate event
rather than being contained within some CDEvent. The most important design
consideration is that links could expand the payload of a CDEvent especially
for a large complex systems. A system may have a deployment of hundreds of
micro-services as artifacts to kubernetes. If we consider having the links live
within the CDEvent, then this specific deployment would contain links to all
consumed artifact events as parents, which greatly expands the total size of
the CDEvent. By having them as separate events allows users to more easily
consume CDEvents. Further, a counter argument may be that those hundred
artifacts could be represented as a single artifact. While that is true, we are
forcing users to know that they need to compress all those artifacts into a
single artifact, because we decided to include links within the CDEvent itself,
and this also restricts how users use CDEvents.

Further links act as metadata about how events are connected, and do not
pertain to anything for the event, itself. Many consumers will not rely on
links at all, so having things separate seems appropriate.

### CDEvents Chain Definition

While most use cases can rely on the CDEvents SDK to dictate what a chain may
look like. However, with the links portion of the SDK, users may specify when a
chain begins and ends giving full control to the user.

### Scalability

Scalability is one of the bigger goals in this proposal and we wanted to ensure
fast lookups. This section is going to describe how the proposed links format
will be scalable and also provide tactics on how DB read/writes can be done.

The purpose of the chain ID is to ensure very fast lookups no matter the
database. Without a chain ID the database or its client would need to
recursively follow event references, upstream or downstream depending on the
use case. A graph DB would easily provide that, and it is also possible to
implement on top of SQL like DBs and document DBs, but it will never be as fast
as querying for a chain ID.

Instead a link service that processes and stores the links to some DB is much
prefer as it gives companies and developers options to choose from.  When using
an SQL database, the chain ID could be the secondary key to easily retrieve
indexed entities. Links could be easily sorted by timestamps which should
roughly coordinate to their linked neighbors, parent and child.

CDEvents that are to be ingested by some service would also have to worry about
the number of events returned. This problem is mitigated in that only the
immediate parent(s) links are returned, and any higher ancestry are excluded.
If some service needs to get access to a higher (a parent's parent) they would
need to use the links API to retrieve them.
