# RFC: Moderation of users

## Motivation

Key goals:

- Equivalent of Zenodo safelisting

- Goals:
    - Create moderation request when user submit record/create community for the first time?
    - List, approve and block request
        - Inject information into record/community (i.e. reindex)
    - Sort search record/community results according to safelisting
    - (Un)Block a user, (Un)Suspend a user, (Un)Delete a user

## User stories

As a **Administrator** I want to...

- have an administration view to moderate users
- select an user and moderate them (e.g. block, approve or suspend)
- search an user that was previously moderated and moderate them (e.g. blocked, approved)
- see the user information (e.g. e-mail, external accounts, etc) when I am moderating an user to have more context on the user.
- see past moderations for an user to understand the user's history (if any).
- know who moderated a user previously (e.g. who blocked the user and why)
- select many users at once and block or approve them all
- see my past moderations and be able to undo them (e.g. in case I miss clicked and want to moderate differently)
- open the user's panel to see detailed information on the user
- open the user's record(s) and community(ies) and be able to quickly navigate to them (as in Zenodo)
- be able to sort the users by e.g. domain
- (OPT) add a note to explain why the user was blocked, approved or suspended.

As a **User** I want to...

- (Optional) report another user for e.g. spam or copyrights
- (Optional) protest my previous moderation (e.g. in case I was blocked by mistake)

## Scope

A moderation request exists to moderate users in terms of acess to the platform. These requests can be created by any entity and are handled by specific users of the system (e.g. moderators).

### Current scope

*User moderation: block, approve, suspend*

Implement the ability to block, approve or suspend an user.

*User moderation history* (if multiple requests per user are accepted)

Retrieve all the previous user moderations.

*Moderation requests search*

Moderation requests will be indexed in the search engine so they can be searchable, sortable, etc.

*Bulk actions*

By default, the actions will be implemented in bulk. E.g. block a group of users.

### Out of scope

*Trusted domains*

> These are CRUD operations for "Trusted Domains". Even though this can come later, we can have the service already prepared to e.g. automatically accept users.

*Spam report*

> This allows users to report other users (e.g. for spam or copyrights infringement). There are many questions here, e.g. can anyone report other users? Or just "trusted" users? "trusted" as in partners (e.g. under a specific e-mail domain). For the backend, the core question is "who can create a moderation request?".

*Status check*

> This incorporates a check from an external source. It means that there can be some asynchronous process running (e.g. spam model) that checks whether an user is legit or not. We need to think how to integrate this with the backend (`update_status_check`?) and the UI (reloading the page or something more fancy). For now, we try to "leave the door open" so this can be implemented further ahead.

*Moderation history*

> This is a UI problem mostly, if users can have multiple moderation requests. How to display this in the list view or even the detailed view? Is it part of the "activity" or something else?

*Moderation notes*
> This can be implemented from the beginning, it's an open text field. However it needs more thought on the UI level.


## Design

The moderation request outcome has an impact on the user state and its related resources (e.g. records). 

Moderation requests translate to actions performed on top of the user and are, therefore, implemented at the user service level.


**Questions**

- What is the difference between user moderation and user management? We can have block / approve / suspend baked in the user model and don't need the UserModeration at all.
    - However if we want to have extra things like who blocked the user then we probably need something else
- Is UserModeration mainly about verified True / False? If so, we can have a view to moderate e.g. unverified users which is just a facet of user management
- What is the default behavior when someone is not verified? E.g. on Zenodo it means search score is modified. It could mean other things as well (e.g. user can't upload records)
- Do we want user report? If so, it might be:
    - A complete new feature for reporting, having maybe comments etc OR
    - We have two distinct entities where one (e.g. UserModeration) 
- Do we want to have content moderation at some point? E.g. moderate one record only or one comment.
- Usage of invenio-requests came up as the "logging" system for moderation requests. Might not be adequate after some thought and would fit more into an audit-type entity.
- can we assume the actions (e.g. block actions) are  synchronous? For asynchronous actions we need to consider some mechanism of either retrying or flagging that the blocking failed.
- can we make it optional? E.g. an instance does not want User Moderation
    - Feature flag?
    - What happens with the data model? If it's optional then can we really bake in new fields into the user model? e.g. 'verified' field or 'blocked_at'

### Assumptions

Blocked users can't request re-moderation since they do not have access to the platform. If they do, the state diagram needs to be adapted.

Block / Unblock actions are atomic, e.g. either all actions are executed or none. 

### User moderation

![image](https://github.com/inveniosoftware/rfcs/assets/1814661/bf23a40c-bcf0-45e2-b3eb-823ab3c89c2e)

![image](https://github.com/inveniosoftware/rfcs/assets/1814661/04361edf-eb10-44af-a5cf-c37a33519dae)

#### Covered use cases

1. User creates an account and is blocked by support
    a. `PENDING -> BLOCKING -> BLOCKED`
2. User creates an account and is approved by support
    a. `PENDING -> APPROVED`
3. User creates an account and is temporarily suspended by support
    a. `PENDING -> PENDING`
4. Moderator changes the status of the moderation
    a. From `BLOCKED -> UNBLOCKING -> APPROVED` (e.g. user was blocked by mistake)
    b. From `APPROVED -> BLOCKING -> BLOCKED` (e.g. user engaged in inappropriate behavior after being approved, e.g. spammer or copyrights infringements)
5. User was reported by another user
    a. From `APPROVED -> PENDING` 
7. (UN)Blocking can take some time and the moderator knows about it (BLOCKING and UNBLOCKING states)

Use case (6) may not be needed if an user can't request re-moderation. In that case, from BLOCKED the only transition would be `BLOCKED -> UNBLOCKING -> APPROVE` when a moderator unblocks an user. This transition (blocked to pending) actually makes the service more complex:

- going back from "PENDING" to "BLOCKED" requires the service to actually retrieve the state from the user itself in order not to execute any blocking side-effects (e.g. deleting records and communities)
- alternatively, we can make "BLOCKED" actions idempotent. The result should be the same as before, since the user was already blocked but it should not be an expensive operation given that there is nothing to do.

#### States machine

States:

- Pending (initial state)
- Blocked
- Approved

Intermediate states:
- Blocking
- Unblocking

#### Actions

* Approve
* Block
* Unblock
* Request moderation
* Suspend

APPROVE

    Set verified = True
    Set state APPROVED

SUSPEND

    Set active = False
    Set suspended_at = now()
    Set state PENDING

BLOCK

    Set state BLOCKING
    Execute block actions (e.g. delete records)
    Set active = False
    Set blocked_at = now()
    Set verified = False
    Set state BLOCKED

UNBLOCK

    Set state UNBLOCKING
    Execute unblock actions (e.g. restore records)
    Set active = True
    Set blocked_at = None
    call APPROVE()

REQUEST_MODERATION

    Set state PENDING
    

### Workflow

All workflows are permission based (e.g. only admins can moderate users)


System components involved:

- Users service
    - UsersModeration sub-service
    - Service components (e.g. block/unblock actions)
    - UOW
- Users resource (REST API)
- Database
- Search engine
- (TBD) records and communities services

### Data model

*TEMPORARY*

**User Moderation Request**
- status: Pending, Approved, Blocking, Blocked
- created
- updated
- moderated_by
- requested_by
- notes
- status check: (this is for external info e.g. spam model)
    -  new, processing, succeeded, failed
    -  timestamp
- user: (shoudl a user only have one moderation request? what about that we remoderate certain users? say spam classifier catches an issue of a verified user)
    - full name
    - affiliation
    - email
    - email domain: domain + flag
    - %inactive domains
    - identities
    - avatar
- activity:
    - record(s)
    - community(ies)


**UserEmailDomains**
- domain
- status: false, undefined, true
- created
- updated

**User:**
- active: T/F (can they log in)
- suspended_at
- blocked_at
- deleted_at
- verified_at
- quotas
    - per_record_storage 

### Service

```python
users_service.search_moderation()
users_service.moderate(identity, users=[], action="")
users_service.request_moderation(identity, user=)

users_service.suspend(identity, users=[], dry_run)
users_service.unsuspend(identity, users=[], dry_run)
users_service.block(identity, users=[], dry_run)
users_service.unblock(identity, users=[], dry_run)
users_service.delete(identity, users=[], dry_run)
users_service.undelete(identity, users=[], dry_run)
users_service.purge(identity, users=[], dry_run)

users_service.set_quota(identity, users=[], record_storage=10000334, ...)

users_service.userdomains.create(identity, ...)
users_service.userdomains.search(identity, ...)
users_service.userdomains.read(identity, ...)
users_service.userdomains.update(identity, ...)
users_service.userdomains.delete(identity, ...)
```

BLOCK_ACTIONS:
    - block: delete records (mark spam), delete communities (mark spam), delete user, ..
    - unblock, restore, ...

### REST APIs

```http
GET /api/users/moderation - List moderation requests
    ?state=pending|approved|blocked 
    ?sort=oldest|newest
    ?statuscheck=
    ?created=
    ?domain=
    ?domain=
    ?identities=github|orcid|missing
```

```http
POST /api/users/moderation - Bulk approve/block
    
{
    "users": ['1234123', '12323'] # strings are important
    "action": "block|approve" # perhaps need 4 actions for pending -> approve, pending -> block, block -> approve, approve -> block
}
```


```htpp
GET /api/users/domains - List user email domains
POST /api/users/domains - Create a new domain
PUT /api/users/domains/:id - Update a domain
GET /api/users/domains/:id - Get a domain
DELETE /api/users/domains/:id - Delete a domain
```
Note: caching

### Open questions

### Brainstorm/meeting 

**17/07**

- It is key to fix mistakes (e.b. block to approved and vice-versa)
- Approach based on requests. E.g. create equests of type "moderation"
    - Start by creating the request type and take it from there
    - At some point we need to hook in the request type to actions (e.g. on record creation)
- Multiple requests (N) (cancel, accept, reject)
    - Request actions are not changed
- Audit as a new mechanism for auditing all the user activity
    - Will come later
- 'Verified' property on the user
    - User deleted_at, blocked_at, suspended_at and verified_at
- user status is computed
- UserEmailDomains in DB (invenio-accounts)
    - support basic CRUD operations
- notes, async block actions and spam report out of scope for now
    - spam report uses the same 'requests' mechanism

