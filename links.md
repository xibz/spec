# Connecting Events - Links Proposal

## Abstract

This proposal will outline how to connect individual CDEvents to each other.
Right now there's no way of associating events to one another without needing
to backtrack across certain context attributes, eg
[id](https://github.com/CDEvents/spec/blob/main/spec.md#id-context).  While
that does give us the ability to construct some graph, we do not know when a
particular span starts or finishes.

This proposal will outline a new approach that will allow for connecting
CDEvents to one another and give a clear distinction of when an activity span start and
finishes.

## Semantics

This section will line out various definitions to ensure there are no
assumptions being made when we talk about linking events

* **CI** - [Continuous integration](https://bestpractices.cd.foundation/learn/ci/)
* **CD** - [Continuous delivery](https://bestpractices.cd.foundation/learn/cd/)
* **Span** - A span is an end to end representation of all activities performed
  in the CI/CD life cycle of some entity
* **Link** - A link is an object containing relations between two CDEvents.
* **Global ID** - A global ID is some overarching ID for a given span.

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
can be linked to one another from a CDEvents perspective and define how links
will look. The second portion will address the goal of scalability.

## Connecting Events

An individual event usually has some relation to some trigger. This can be a
new commit added to a pull request, a test suite being called in the CI
process, or publishing a new artifact for some system to consume. While these
events mean something themselves, they do not give the proper context of what
caused what. This section will introduce two new fields to be used in the
CDEvent spec that will allow for giving some relation between CDEvents. The two
new fields will be added to the context section of CDEvents to solve this, a
global ID and parent IDs.

```json
{
  "context": {
    "version": "0.3.0-draft",
    "id": "505b31c2-8bc8-47b3-a1a0-269d7a8530ac",
    "source": "dev/jenkins",
    "type": "dev.cdevents.testsuite.finished.0.1.1",
    "global_id": "00000000-0000-0000-0000-000000000001", # new global id field
    "parent_ids": [ # new parent id field
        {
            "link_id": "271069a8-fc18-44f1-b38f-9d70a1695819",
        }
    ],
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

The `global_id` is an ID that will be generated when a new CDEvent span is
wanted or if no CDEvent span is present. This ID will follow the
[UUID](https://datatracker.ietf.org/doc/html/rfc4122) format. Global IDs will
serve as a bucket for all CDEvents with some sort of relation to each other.

The `parent_ids` is an array of `context.id`s rather than `subject.id` since
the `subject.id` do not have a specified format, and may not be unique across
spans. For example, a CI service can have the same `subject.id` as some CD
service. Further, this field is an array which allows for events to fan out and
in like when a test suite calls a bunch of test cases in parallel.

Below represents an example of a few CDEvents and how they link to one another.


**Change Merged CDEvent**
```json
{
  "context": {
    "version": "0.4.0-draft",
    "id": "11111111-0000-0000-0000-000000000000",
    "global_id": "00000000-0000-0000-0000-000000000000",
    "parent_ids": [],
    "source": "/event/source/123",
    "type": "dev.cdevents.change.merged.0.2.0",
    "timestamp": "2023-03-20T14:27:05.315384Z"
  },
  "subject": {
    "id": "mySubject123",
    "source": "/event/source/123",
    "type": "change",
    "content": {
      "repository": {
        "id": "TestRepo/TestOrg",
        "source": "https://example.org"
      }
    }
  }
}
```

**Pipeline Queued CDEvent**
```json
{
  "context": {
    "version": "0.4.0-draft",
    "id": "22222222-0000-0000-0000-000000000000",
    "global_id": "00000000-0000-0000-0000-000000000000",
    "parent_ids": ["11111111-0000-0000-0000-000000000000"],
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

**Pipeline Started CDEvent**
```json
{
  "context": {
    "version": "0.4.0-draft",
    "id": "33333333-0000-0000-0000-000000000000",
    "global_id": "00000000-0000-0000-0000-000000000000",
    "parent_ids": ["22222222-0000-0000-0000-000000000000"],
    "source": "/event/source/123",
    "type": "dev.cdevents.pipelinerun.started.0.1.1",
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

With these three separate CDEvents we can see that these all belong to a single
span of `00000000-0000-0000-0000-000000000000`, and that the relation is as follows

```
+---------------+   parent   +-----------------+   parent   +------------------+
| change merged |<-----------| pipeline queued |<-----------| pipeline started |
+---------------+            +-----------------+            +------------------+
             |                   ^          |                   ^
             +-------------------+          +-------------------+
              inferred connection            inferred connection
```

By establishing who has what parent, we can infer how they connect.


## Propagation

Span propagation will be handled differently depending on the protocol that is
used. This proposal will not cover the technical details but will cover various
use cases that can occur in large systems. A separate technical design document
can be found [here](TODO/link/to/tech/doc) for those who want some guidance on
how propagation could be handled for a given protocol

### Partial CDEvent Support

The first case is where only a subset of the systems support CDEvents.
For this example, we will use four services. Service A, B, and C.

Services A, B, and D support CDEvents, while C does not. An example CDEvent
span may look like:

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
is to query for a full span since the link data may be persisted in different
databases.

Case 2 is more about consuming duplicate events and sending duplicate events
which can cause a potential issue around how links represents those
duplications.

## Links Spec

So far we've only talked about what a service may receive when receiving the
new CDEvent containing links. However, when we store a link, there's a lot more
metadata that can and should be added.

The idea is we'd expect users to start link, group, and end links accordingly
through APIs we'd provide. This is very similar to how tracing works.
Granularity in tracing is completely up to the engineer which this proposal
also intends users to do.

This will introduce new APIs to the CDEvents SDKs to handle automatic injection
of links or allowing users to provide their own definition of what a CDEvent
span is.

The link spec definition will look like this

```json
{
  "global_id": "05CEDE1F-8C13-4699-80C3-B4AC6468414E",
  "link_type": "TO",
  "event_transport": "TRIGGER",
  "from": "F27E36A4-5C78-43C0-840A-52524DFEED03",
  "to": "F004290E-5E45-45F4-B97A-FA82499F534C",
  "tags": [
    "ci.environment": "prod"
  ]
}
```

| Name            | Description                                                          |
|-----------------|----------------------------------------------------------------------|
| global_id       | CDEvent span ID                                                      |
| link_type       | An enum that represents the type of link, e.g. 'TO', 'FROM', 'GROUP' |
| event_transport | An enum that represents how the event message was transported |
| from            | Where the link is coming from                                        |
| to              | Where the link is going to                                           |
| tags            | Custom metadata that an individual link can have                     |

Below defines a list of `link_type` enums a kind can have

| Name  | Description                                                                               |
|-------|-------------------------------------------------------------------------------------------|
| TO    | When a link is creating an event, it will use TO to signal to what target ID              |
| FROM  | When a link is receiving an event, it will use FROM to signal what service called it      |
| GROUP | When a link is to be grouped with other events, it will use GROUP to establish a grouping |

Below defines a list of `event_transport` enums a link can have

| Name    | Description                                                        |
|---------|--------------------------------------------------------------------|
| TRIGGER | When an CDEvent was triggered, e.g. cron |
| P2P     | When a CDEvent sent using p2p (peer to peer) instead of using the event bus |
| BUS     | When a CDEvent was propagated via event bus                         |

These links can be, but are not limited to, sent when a CDEvent has completed
to some collector or to the link service itself. Further the link service will
allow for tagging of various metadata.

Some users may prefer to not run a separate links service especially if they
know their overall flow may only contain a few links. If that is the case,
simply turning on linking payload aggregation, will send all links in the
payload. Mind you, this can make the payload very large, but may be good for
debugging.

The global ID header will continue to propagate, unless the user explicitly
starts a new CDEvent span. If there is no global ID header, the client will
generate one and that will be used for the lifetime of the whole events span

```
+-----+      +-----+      +-----+                                                                +--------------+         +-----------+
| Git |      | CI  |      | CD  |                                                                | Link Service |         | Event Bus |
+--+--+      +--+--+      +--+--+                                                                +-------+------+         +-----+-----+
   |            |            |           #1 (send change merged event)                                   |                      |
   +--------------------------------------------------------------------------------------------------------------------------->|
   |            |            |           #2 (source change links)                                        |                      |
   +---------------------------------------------------------------------------------------------------->|                      |
   |            |            |                                                                           |                      |
   |            |            |           #3 (receive change merged event)                                |                      |
   |            |<--------------------------------------------------------------------------------------------------------------+
   |            |            |           #4 (send pipeline queued event)                                 |                      | 
   |            +-------------------------------------------------------------------------------------------------------------->|
   |            |            |           #5 (pipeline queued links w/ parent: #1)                        |                      |
   |            +--------------------------------------------------------------------------------------->|                      |
   |            |            |           #6 (send pipeline started event)                                |                      |
   |            +-------------------------------------------------------------------------------------------------------------->|
   |            |            |           #7 (pipeline started links w/ parent: #5)                       |                      |
   |            +--------------------------------------------------------------------------------------->|                      |
   |            |            |           #8 (send build queued event)                                    |                      |
   |            +-------------------------------------------------------------------------------------------------------------->|
   |            |            |           #9 (build queued event links w/ parent: #7)                     |                      |
   |            +--------------------------------------------------------------------------------------->|                      |
   |            |            |           #10 (send build started event)                                  |                      |
   |            +-------------------------------------------------------------------------------------------------------------->|
   |            |            |           #11 (build started event links w/ parent: #9)                   |                      |
   |            +--------------------------------------------------------------------------------------->|                      |
   |            |            |           #12 (send build completed event)                                |                      |
   |            +-------------------------------------------------------------------------------------------------------------->|
   |            |            |           #13 (build completed event links w/ parent: #11)                |                      |
   |            +--------------------------------------------------------------------------------------->|                      |
   |            |            |           #14 (send pipeline finished event)                              |                      |
   |            +-------------------------------------------------------------------------------------------------------------->|
   |            |            |           #15 (pipeline finished event links w/ parent: #13)              |                      |
   |            +--------------------------------------------------------------------------------------->|                      |
   |            |            |                                                                           |                      |
   |            |            |           #16 (receive pipeline finished event)                           |                      |
   |            |            |<-------------------------------------------------------------------------------------------------+
   |            |            |           #17 (send pipeline queued event)                                |                      |
   |            |            +------------------------------------------------------------------------------------------------->|
   |            |            |           #18 (pipeline queued links w/ parent: #15)                      |                      |
   |            |            +-------------------------------------------------------------------------->|                      |
   |            |            |           #19 (send pipeline started event)                               |                      |
   |            |            +------------------------------------------------------------------------------------------------->|
   |            |            |           #20 (pipeline started links w/ parent: #18)                     |                      |
   |            |            +-------------------------------------------------------------------------->|                      |
   |            |            |           #21 (send environment created event)                            |                      |
   |            |            +------------------------------------------------------------------------------------------------->|
   |            |            |           #22 (environment created links w/ parent: #20)                  |                      |
   |            |            +-------------------------------------------------------------------------->|                      |
```

This show's an example of how these different types would be used in a CI/CD setting.

### CDEvents Span Definition

While most use cases can rely on the CDEvents SDK to dictate what a span may
look like. However, with the links portion of the SDK, users may specify when a
span begins and ends giving full control to the user.

### Scalability

Scalability is one of the bigger goals in this proposal and we wanted to ensure
fast lookups. This section is going to describe how the proposed links format
will be scalable and also provide tactics on how DB read/writes can be done.

The purpose of the global ID is to ensure very fast lookups no matter the
database. Without a global ID the database or its client would need to
recursively follow event references, upstream or downstream depending on the
use case. A graph DB would easily provide that, and it is also possible to
implement on top of SQL like DBs and document DBs, but it will never be as fast
as querying for a global ID.

Instead a link service that processes and stores the links to some DB is much
prefer as it gives companies and developers options to choose from.  When using
an SQL database, the global ID could be the secondary key to easily retrieve
indexed entities. Links could be easily sorted by timestamps which should
roughly coordinate to their linked neighbors, parent and child.

CDEvents that are to be ingested by some service would also have to worry about
the number of events returned. This problem is mitigated in that only the
immediate parent(s) links are returned, and any higher ancestry are excluded.
If some service needs to get access to a higher (a parent's parent) they would
need to use the links API to retrieve them.
