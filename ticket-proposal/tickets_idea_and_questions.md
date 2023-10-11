# Tickets Ideas and Questions

## Design Considerations

### Ticket Events
Do we need an actual ticket created event?
```
{
  "context": {
    "version": "0.4.0-draft",
    "id": "271069a8-fc18-44f1-b38f-9d70a1695819",
    "source": "/event/source/123",
    "type": "dev.cdevents.ticket.created.0.1.1",
    "timestamp": "2023-03-20T14:27:05.315384Z"
  },
  "subject": {
    // ticket related metadata
  }
}
```

How would an event know what ticket was associated with any CDEvent?

So, maybe this approach would not work, since system would need to be aware of
tickets which I do not know of any system that does this today. However, some
standard need to be introduced to persist that ticketing metadata.

Currently, how a lot of teams associate some ticket to a feature, is to
mention it within a commit or the pull request.

A git commit might look something like:
```
<ticket ID or link>: my feature that solves everything
```

Ticket system -> creates create_ticket event
PR -> associates via commit that ticket link and/or create_ticket event -> source_change_created event -> passes this ticket metadata to all other events that are created

```
{
  "context": {
    "version": "0.4.0-draft",
    "id": "271069a8-fc18-44f1-b38f-9d70a1695819",
    "source": "/event/source/123",
    "type": "dev.cdevents.ticket.linked.0.1.1",
    "timestamp": "2023-03-20T14:27:05.315384Z"
  },
  "subject": {
    // specifies the chain_id that it is apart of
  }
}
```
Benefits:

- Flexibility. Allows users to implement (or not) this event type as they see fit.
- Allows actions to be taken at the time of ticket creation.

Drawbacks:
- There could be tickets that are not related to CD.
### Metadata on Existing CDEvents

```
{
  "context": {
    "version": "0.4.0-draft",
    "id": "271069a8-fc18-44f1-b38f-9d70a1695819",
    "source": "/event/source/123",
    "type": "dev.cdevents.artifact.packaged.0.1.1",
    "timestamp": "2023-03-20T14:27:05.315384Z",
		"tickets": [
			{
				// ticket metadata here
			}
    ]
  },
  "subject": {
    "id": "pkg:golang/mygit.com/myorg/myapp@234fd47e07d1004f0aed9c",
    "source": "/event/source/123",
    "type": "artifact",
    "content": {
      "change": {
        "id": "myChange123",
        "source": "my-git.example/an-org/a-repo"
      }
    }
  }
}
```

How would an event know what ticket was associated with any CDEvent?

So, maybe this approach would not work, since system would need to be aware of
tickets which I do not know of any system that does this today. However, some
standard need to be introduced to persist that ticketing metadata.
