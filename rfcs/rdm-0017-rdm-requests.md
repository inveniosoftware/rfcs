- Start Date: 2020-03-16
- RFC PR: [#17](https://github.com/inveniosoftware/rfcs/pull/17)
- Authors: Lars Holm Nielsen
- State: DRAFT

# Invenio modules for creation and handling of standardized user requests

## Summary

> One paragraph explanation of the feature.

## Motivation

Operating a repository means operating an online service and providing users
with support. Many of these requests for support will be related to
standard requests. Here's some examples on Zenodo for standard requests:

- File replacement
- Deletion request
- Report spam
- Quota increase
- Take-down request
- Ownership transfer

The workload on support teams can be reduced significantly if as much of these
requests are automated, and the only required involvement from the support is
a review and either accept/reject the request.

This RFC proposes a generic framework for allowing developers to quickly write
plugins for new types of requests.

A request is essentially something that can be submitted by a user via a form
with some associated data. The request is routed to an entity (e.g. user or
community) in the system, who can accept or reject the request.

Each request can lead to a conversation between the creator and one or more
users, before the final acceptance/rejection of the request
can be

## Detailed design

### Data model

A **request** has both a set of common fields and set of fields custom to the
type.

- id (uuid)
- type (e.g. quota-increase)
- created_by (a user)
- created (a timestamp)
- modified (a timestamp)
- expires_at (a timestamp if needed)
(- cancelled_at (a timestamp if needed))
(- sla (a timestamp if neeeded))
- routing_key
(- assignees (one or many?))
- state

Custom fields are stored as a JSON object:

- data (json data)

A request can have **request comments** associated with it:

- id (uuid)
- created_by (a user)
- created  (a timestamp)
- modified  (a timestamp)
- message (markdown?)

### Plugins

A request plugin needs to define the following parts:

- A form for allowing the user to make a request
- A plugin implementation plugin for the type including validation and schema
  of data.
- Templates for rendering the requests in generic interfaces.
- Permissions for who can create, edit, accept/reject the request.
- Feature flags for enabling/disabling features (like allowing comments or auto
  expiring).

### Example plugin

Following is pseudo code for a request:

```python
class DeletionRequest(Request):
    schema = DeletionSchema()
    template_detail = "deletion_detail.html"
    template_list = "deletion_list.html"
    permissions = deletion_policy

    can_expire = True
    can_comment = True

    def on_create(req):
        # Create the request and other related resources (e.g. bucket for file
        # replacements).

    def on_validate(req):
        # Custom validation of the request

    def on_accept(req):
        # ...

    def on_cancel(req):
        # ...

    def on_reject(req):
        # ...

    def on_expire(req):
        # ...

deletion_policy = class RequestPermissionPolicy():
    can_list
    can_create = ...
    can_read = ...
    can_update = ...

```

### REST API

- GET /requests?q=... - Search for requests
- GET /requests/<type>?q=... - Search for requests
- POST /requests - Create a request
- GET /requests/:id - Get a request
- DELETE /requests/:id - Delete a request
- PUT /requests/:id - Update a request (e.g. change quota or edit text)
- POST /requests/:id/accept or PATCH /requests/:id ?
- GET /requests/:id/comments - List comments
- POST /requests/:id/comments - Create a comment
- PUT /requests/:id/comments/:id - Update a comment


### Implementation

Creating a request:

- Load data from request
- Check permission against data
- Create a PID (via self.minter)
- Create the record
- Commit
- Index
- Send response (with links)

==

POST /requests
{
    'type': 'quota-increase',
    'data': {
        'record': 'recid:1xa343-adf31',
        'bucket': '5fa8ed0f-efaf-4a5d-bdf5-2b3cdf58edee',
        'new-quota': 100000000,
    }
}

- Map type to loader.
    - Record id, Bucket, New quota


### User interface

- Forms for creating new requests (both related to record and not)
- List of current requests (my requests) - i.e see my current requests
- List of current requests - i.e. see which requests I have to deal with.
- Possibly have specific UIs for specific types of requests.

### Implementation

The module can use Invenio-Records-REST to provide a REST API on top of this.
It's to be seen how the comments should be implemented.

## Example

#### Quota increase request

- Data: Record, Bucket, New quota
- Routing: admins (or possibly curators?)
- On accept:
    - Set quota of associated bucket to the requested quota size
- On reject:
    - Do nothing

#### Report spam

- Data: Record, Reason
- Routing: admins
- On accept:
    - Delete record and mark as spam
- On reject:
    - Send reply to user who reported the problem.

#### Transfer ownership

- Data: Record, new owner, reason
- Routing: New owner of record
- On accept:
    - Transfer ownership
- On reject:
    - Message is sent back to old owner
- Request expires after 5 days

#### Request deletion

- Data: Record, reason
- Routing: group:admins
- On accept:
    - Delete record and pids?
- On reject:
    - Message is sent back to old owner
- Request expires after 5 days

#### Replace files

- Data: Record, reason, bucket
- On create:
    - Create a bucket snapshot.
- Routing: group:admins
- On accept:
    - Take changes from snapshot and overwrite new.
    - Remove temporary snapshot
    - Create new SIP.
- On reject:
    - Remove snapshot and files.

#### User account removal

- ???

#### GDPR related request - get me all information on me.

- ???

#### Inclusion request

- ???

#### Claim request

## How we teach this

## Drawbacks

## Alternatives

## Unresolved questions

- Do we need to add e.g. labels or some sort of close code.
- How are requests flushed from the system? After a year? Never?
- Do we need to add something like reviewers, and approvals?
- Can this be used for inclusion requests / claim requests?
- How are notifications handled? Email? How do you unsubscribe from
  notifications? What about large automated uploads of 300k new things?
- Scalability of the solution?
- Can we avoid PIDs? Probably not
- Some sort of notification system on top of it.
- Email rendering
