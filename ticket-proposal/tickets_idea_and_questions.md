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

## Ticket States

* Created/Open
* InProgress (from the development point of view)
* Reviewed
* Approval statuses
    - approval.declined
    - approval.inprogress
    - approval.approved
* Testing
* Resolved

## Updated Ticket Event

```
{
    "ticket_id": "864d6c49-5a61-45a6-b330-b1ba83315002",
    # Link could be used here
    "updater": "xibz",
    "prev_state": "REVIEWED",
    "status": "DEPLOYED",
    "timestamp": "Thu Oct 12 14:26:11 CDT 2023"
}
```

deploy to dev --> wait for successful tests --> deploy to beta --> wait for successful tests --> wait for approval --> deploy to prod

```
+---------------+       +----------------+       +----------------+       +----------------+       +-------------------+       +----------------+
| Deploy to dev +------>| Wait for Tests +------>| Deploy to beta +------>| Wait for Tests +------>| Wait for Approval +------>| Deploy to prod |
+---------------+       +----------------+       +----------------+       +----------------+       +-------------------+       +----------------+
```

1. Dev service sends the event deployed to dev
2. Test service sends test events
3. Beta service sends the event deployed to beta
4. Test service sends test events
5. Ticketing system sends ticket updated event
6. Prod sends the event deployed to prod

```
{
    "ticket_id": "864d6c49-5a61-45a6-b330-b1ba83315002",
    # Link could be used here
    "updater": "xibz",
    "prev_state": "TESTING",
    "status": "APPROVED"
}
```

## CDEvent Ticket Event

Based on talking about consistency within Spinnaker, the updated ticket event
would be the only event with state, rather the state living in the predicate

1. `dev.cdevents.ticket.opened`
```json
{
    "ticket_uri": "https://jira.com/1234", 
    "ticket_type": "DOCUMENTATION",
    "author": "xibz",
}
```
2. `dev.cdevents.ticket.in_progress`
```json
{
    "ticket_uri": "https://jira.com/1234", 
    "ticket_type": "DOCUMENTATION", # do we need this?
    "committer": "xibz"
}
```
3. `dev.cdevents.ticket.in_review`
```json
{
    "ticket_uri": "https://jira.com/1234", 
    "ticket_type": "DOCUMENTATION", # do we need this?
    "committer": "xibz"
}
```
4. `dev.cdevents.ticket.reviewed`
```json
{
    "ticket_uri": "https://jira.com/1234", 
    "ticket_type": "DOCUMENTATION", # do we need this?
    "committer": "xibz"
}
```
5. `dev.cdevents.ticket.testing`
6. `dev.cdevents.ticket.resolved`

**Approvals would be**

6. `dev.cdevents.ticket.approval.approved`
7. `dev.cdevents.ticket.approval.declined`
8. `dev.cdevents.ticket.approval.in_progress`

or

6. `dev.cdevents.approval.approved`
7. `dev.cdevents.approval.declined`
8. `dev.cdevents.approval.in_progress`

**Questions About Approval events**

Do we need a list of approvers?

