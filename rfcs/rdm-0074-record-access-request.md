# RFC: Managing access to records

## Summary

The RFC proposes an UX for managing access on an individual record. This is complementary to managing access at a collection level via  communities. Record managers will be able to grant access on a single record to users/groups, and create secret links that can be shared with guests (unauthenticated users). In addition, record managers can allow users and guests to request access to a record. Access can be granted to view, preview, edit or manage a record.


## Motivation

Following is an overview of high-level user stories and actors involved in managing access to a record.

### Actors

- Record owner - the user who uploaded the record.
- Community manager - a user with the manager or owner role in a community that the record is part of.
- Record manager - a user who has *can manage* access on a record (record owners, users granted can manage access and community managers).
- User - Any authenticated user.
- Guest - Any unauthenticated user.

### User stories

- Granting access:
    - As a record manager, I want to grant access to my record, so that I can share individual records with specific people and groups.
    - As a record manager, I want to grant a collegue edit permission on my draft record, so that they can validate the record prior to publishing it.
    - As a guest, I want to be granted acccess via a secret link, so that I don't have to create an account on the system.
    - As a community manager, I want to grant special access to a specific record in my community, so I'm not dependent on the record owner.
- Requesting access:
    - As a record manager, I want to receive access request, so that I can control dissemination of my record.
    - As a guest, I want to request access to an embargoed record, so that I don't have to send an email to the author.  
    - As a record manager, I want to communicate with the requestor, so that I can ask for more details for an access request.
- Confidential peer review:
    - As a record manager, I want to send my journal editor a secret link providing access to my draft record, so that the editor can send it to the peer reviewers.

:::danger
TUW: Giving access to the files but *not* the metadata could make the peer review double-blind instead of single-blind (unless the researchers' names are contained in the files). At least for Zenodo; for institutional repositories you're already giving the institution away (theming, URL, ...).
:::


## Design

The UX design is divided in 4 areas describing:

- Managing access - how a record manager can manage the access to a record.
- Requesting access - how users and guest can request access to a record.
- Approving access - how a record manager approves or declines request for access.
- Using access - how users and guest use access granted to them.

### Managing access

Record managers access a Share modal used manage access to their record via:

- Record landing page:
    - "Share"-button.
    - Dropdown menu next to restricted files.
- Deposit form:
    - "Manage access"-button in right column

All three buttons will load the Share modal and are only visible for record managers.

*Record landing page*

![](./0074/share-button.png)

*Deposit form*

![](./0074/deposit-page.png)

#### Share modal

The Share modal consits of three tabs:

- People and groups - used to manage access for users and groups.
- Links - used to manage access via secret links.
- Access requests - used to configure ability of users and guests to request access.

The first two tabs will display the number of users/groups/links the record has been shared with. The Share modal has been modelled over the community members UI.

:::danger
TUW: Should we include creation/edit timestamps for the access grants in the UI, e.g. as on-hover tooltip in each line? That could help people in their decision making when revoking access.
:::

#### Share modal: People and groups-tab

A record manager can manage access to users from the people and groups tab. From the tab they can:

- Remove access of a user
- Change access mode of a user
- Grant access to one or more users
- Remove or change access in bulk

The list of users is sorted alphabetical by name and can be paginated. The modal can be dismissed via a done-button.

:::info
**Record visibility**
The share modal is not impacted by a records visibility - i.e. public records, public records with restricted files, or restricted records. For instance a user given "can view" access to a public record will not gain any further permission that what they already have, until the record at a later stage may change to a restricted record.
:::

![](./0074/share-modal.png)

:::info
**Max/Lars:** The list of users/groups comes from the record's grant tokens (`record.parent.access`). The grants need to be expanded into something that's useful for the frontend.
**Max/Lars:** We only expect a small number of grants per record.
:::

Access menu has the following options:

- Can view
- Can preview
- Can edit
- Can manage

A record owner in addition has a menu item to request transfer of ownership of a record to a user already in the list. The transfer of ownership request is subject of a separate RFC to be written.

:::info
**One record owner**

Currently InvenioRDM allows multiple users to own a record. This RFC proposes to restrict record ownership to a single user. This will reduce complexity for:
- Routing of access requests to record owners.
- Business rules related to ensuring a record always has an owner.
- Accounting of resources consumed by a record owner.
:::

![](./0074/access-dropdown.png)

A record manager can search for users of groups. When a user is selected via the mouse or hitting enter, the content will change to the *add people and groups*-interface.

This interface allows:

- Search for an add one or more users and groups
- Set the access mode of the users being added.
- Decide if people should be notified and provide a message (groups cannot be notified).

The interface can be dismissed via:

- Back-arrow in the title
- Discard-button

Both will put the modal back to the list of people and groups.

Once the Add-button is clicked, the users are added to the interface returns to the list of users.

![](./0074/people-tab.png)

:::danger
TODO: Define email being sent to users.
TODO: Can we search users and groups combined via the REST APIs?
:::

:::info
**Max/Lars:** We do not want to have two different lists for *users* and *groups* because that gets overwhelming fast. If it's infeasible to support users and groups in a single list, it would be preferable to only support users.
**Max/Lars:** One of the biggest obstacles for having users and groups available for search in a single endpoint is making sure that you can't search for stuff you shouldn't be (c.f. query parsers).
:::


#### Share modal: Links-tab

The links-tab is used to manage links created by the record managers themselves or links created via access requests.

From the tab, a record manager can:
- Get a new link to share access to the record
- Copy an existing link.
- Delete an existing link.
- Change access mode of an existing link.
- Change or delete links in bulk.

The interface differentiate links created:
- by the record owner 
- via an access request

A link may have a future expiration date.

A link may have a description defined when created, which can also be edited inline by pressing the pencil-icon next to the description.

Pressing "Get a link" will change the existing row into a link so that it can easily be copied since the "Copy link" button will be straight below the mouse cursor.

![](./0074/links-tab.png)

#### Share modal: Access requests-tab

From access request-tab, a record manager can enable an option to allow users and guests to request access to the record. Once the option is enabled the remaining interface changes from disabled to normal state.


The record manager can now:
- Provide a text to describe the condition under which they approve access.
- Change default options for the links (changable later for a specific request).

The following options can be changed:

- The default access mode - defaults to can view.
- The default link expiration - defaults to never with options for 1 month, 2 months, 6 months and 1 year.
- The receiver of the requests - defaults to the record owner, but can be changed to any community that a record is part.

:::danger
- TODO: Should this be a data access agreement instead of conditions?
- KTH: - As a community manager/curator/record owner I would like to grant access to data under certain conditions and I would like the person who requests access to accept a customizable/template data access agreement before getting access to the data files.
- TODO: Needs a "I agree checkbox"
- TODO: Allow an admin to define a list of templates that can be rendered instead.
:::

:::info
**Max/Lars:** In addition to the topic of Captchas in the "request access form" (below), we should split the single **"allow anyone to..."** checkbox into two checkboxes: **"allow registered users to..."**, and **"allow unauthenticated guests to..."**
**Max/Lars:** The reason why the request receiver can only be the record owner or the community is purely technical. As soon as requests support multiple receivers, access requests should be able to set "both" as receivers.
:::

![](./0074/access-request-tab.png)

### Requesting access

The record landing page will allow:
- *users and guests* to request access to a *public record with restricted files*.
- *users* to request access to a *restricted record* (because a restricted record will by default ask for a user to login first).

The landing page will display:
- A message saying files are restricted
- A message saying if a user/guest can request access
- Optionally, a description provided by the record owner describing the conditions under which they approve access.

The form to request access is differentiated between guests and users.

#### Guest form - public record with restricted files

Guests a required to provide a:

- Full name 
- Email address
- Optionally a message
- Confirm they consent to sharing their name and email with record managers (third-party).

![](./0074/access-request-form.png)


Once a guest requests access, they will be presented with a modal dialog saying email confirmation is needed. The guests stays on the record landing page.

![](./0074/email-confirm.png)

Once the email is confirmed, the request is created and the guest will be redirected to the request detail page via a secret link, and receive the secret link by email as well. This ensures that guests do not need an account on the system to be able to respond to messages from record managers.

:::danger
TUW: So it might be that the token for link to the request detail page will need to keep track of the guest's name, and email address and set up an Identity.
TUW: Will we need a Captcha or similar, to prevent people from spamming? And is it a viable attack vector to have InvenioRDM spam somebody else by entering arbitrary peoples' email addresses?
:::

:::info
**Max/Lars:** Captchas could be used here (maybe frontend-only?), as well as rate-limiting to prevent Bob from spamming Alice's email inbox with access request emails.
**Max/Lars:** It's fine that we don't have an API endpoint to create access requests, because automation is exactly what we _do not_ want here.
**Max/Lars**: Requests should only be created once the link sent out to the specified email address is accessed. This ensures that the guest owns the email address, and enables us to easily give the link a short life time (i.e. a day or so), before we can delete it from the database (so we don't have to play around with the request's expiration date).
:::

#### Users form - public record with restricted files

Users can optionally provide a message. A request is created immediately, and the user is redirected to the request detail page.

:::danger
TODO: Should users provide consent to share name/email with third-party.
:::

![](./0074/user-access-request-form.png)


#### Embargoed records

Users and guests can also request access to embargoed records. The form fields are identical to restricted records.
![](./0074/embargo-request.png)


#### No access conditions

If the record managers did not provide access conditions text, a simplified view is rendered.

![](./0074/no-policy.png)



### Approving access requests

Approving access happens via requests which is created as a result of submitting the request acces form as described in the previous section. The request detail can be accessed

Requests are sent to either (depending on the configuration set when access request feature was enabled):
- a record owner, or 
- community (i.e. communit managers and owners) 

:::danger
TODO: If a request is sent to a record owner, then users with can manage access cannot manage the request.
TUW: Above, it sounded like record managers can define the (default?) receiver of access requests (owners or community), but they can't accept/decline the requests - could this be an issue somehow?
:::

 
Accessing the request:
- Record owners and users can access a request via their "My dashboard".
- Guests can access the request via a secret link directly to the request.
- Community managers can access a request via the community request inbox. 

Note that an access request cannot be sent to both the record owner and a community.

:::danger
TODO: Do we want to create a record request inbox? To view all requests for a record?
:::

#### Request title

The title of the request is created from:

```
<full name> requests access to <title>
```

#### Request detail: User view

A guest will see a standard request detail page, and has the ability to cancel the request as well as provide comments.

The initial comment provide a view of the request payload of the access request - specifically the following information is displayed:

- the record 
- the access mode

This model also works for potential future types of requests such as quota increase requests, file replacement, spam reports, record claiming etc.

The user making the request is explicit from the creator of the request and the first message.

The associated record can be viewed quickly via the record tab. It's important that the record is rendered according to the guests current permissions, and not the requested permissions.

![](./0074/access-request-detail.png)

:::info
**Max/Lars:** The request approval for users should result in a grant token (which doens't expire) being added to the record (`record.parent.access`) rather than a link. "Link expiration" in the image above is a typo. If the owner doesn't want to give out a "permanent" grant, they can create a shared link, post it as a comment in the request, and then decline the request.
**Max/Lars:** The settings/information (access level, link expiration) are part of the *request's payload*, and partially editable by the receiver.
:::

#### Request detail: Guest view

The guest view of a request detail is similar to the user view. THe following additonal information is displayed:

- Full name
- Email address

The request receivers must be extended to support also email creators. The same feature will be needed for community membership invitations as well.

![](./0074/request-detail-guest.png)


#### Request detail: Record manager view (e.g. record owner)

A record manager will be able to edit the the two properties of the request payload:

- Access
- Link expiration

This enables the record manager to change the request according their needs. The defaults for the two properties are configured under "managing access"

![](./0074/request-manager-view.png)

#### Request detail: Approved request

Once a requst is approved, an event will be logged on the request timline, with abutton to view the record.

![](./0074/approved-request.png)

### Access granted

Once a user or guest have access to a record, the restricted part (either full record or restricted files) will be displayed.

![](./0074/access-granted-files.png)

Any access condition is still displayed. A button "Explain access" allows the user to understand why they have access to a given record, by popping up a modal providing the details.

![](./0074/explain-access.png)


### Emails

**Email confirmation needed, sent to guest**

*Subject*: 
Access request verification

*Body*: 
You have submitted an access request for the following record:

SkyNet: A modular nuclear reaction network library
https://inveniordm.web.cern.ch/records/1170313

To complete the access request, please verify your email address by clicking the link below:

https://sandbox.zenodo.org/record/1170313/accessrequest/eyJhbGciOiJIUzUxMiIsImV4cCI6MTY3ODU2OTYzMiwiaWF0IjoxNjc4MTM3NjMyfQ.eyJkYXRhIjp7ImVtYWlsIjoibGFycy5ob2xtLm5pZWxzZW5AY2Vybi5jaCJ9LCJpZCI6NTcwLCJybmQiOiI2MmY0NTc0NSJ9.mH4mQmV8nSRbtdQ4J6thw4GQ-wFeNS28jaClCMjf_lBoeONPwW5GzLbSnlU3lK0GBfcHYWRmbG873TeIKH4usQ/confirm

Please note that above link is only valid for 5 days. If this request was not made by you please ignore this message.

**Access request submitted, sent to guest/user**

*Subject:* 
Access request: SkyNet: A modular nuclear reaction network library

*Body:* 
Your access request has been successfully sent to the record owner. The decision whether to accept or decline the access request is solely under the responsibility of the record owner. 

You can manage your request on:
https://inveniordm.web.cern.ch/requests/ce9773da-e825-476f-8088-bedb78ed4c69?token=eyJhbGciOiJIUzUxMiIsImV4cCI6MTY4MDczNTU5OSwiaWF0IjoxNjc4MTQxMjEzfQ.eyJkYXRhIjp7InJlY2lkIjoxMTcwMzEzfSwiaWQiOjM3Mywicm5kIjoiMDNmNGNlMmQifQ.PZFZFK3cmL0-bwSOSzsrplDZ73o4FYw3VG6F9f4yJVhGaJxTPVXmq0zGSyMW0szXqI0XiQDzzBoUulhF89QWVQ

**Access request submitted, sent to record owner**

*Subject:* 
Access request: SkyNet: A modular nuclear reaction network library

*Body*:
You have a new access request. To accept or decline the request, please open below link: 
https://inveniordm.web.cern.ch/me/requests/ce9773da-e825-476f-8088-bedb78ed4c69

:::danger
TODO: Email sent to people granted access.
TODO: Email on accept or reject of request.
:::

### Special considerations

#### Privacy

- Revoke consent by guest: The request is deleted.
- Revoke consent by user: The request is deleted.

:::danger
TODO: Should users confirm consent to share their email - or can we make it as part of the privacy policy?

of you make a request, the third-party will see your identity.
:::

#### Deletion

- User requesting access: The request is cancelled. The record owner can still see the request, but user details are no longer available.
- Record owner: The request is cancelled. The user can still see the request, but the record owner details are no longer available.
- Community deletion: The request is cancelled. The user can still the request, but the community detaills are no longer available.
- Record deletion: The request is cancelled.

#### Ownership transfer of record

If an access request for a record is submitted to a record owner, and the record owner transfers ownership to a new user the following behaviour is expected:

- ....

:::danger
TODO: What will happen? Error on accept? 

Options: 
- New owner will inherit request (privacy policy must allow for this)
- Request is cancelled with a message 
- Request stays open, but when old owner tries to accept it provides an error message saying they don't have permissions anymore.

What happens when request is sent to 
- Users with can manage permission
- Community owners/managers
:::

## REST API needs

:::danger
TUW: would the first endpoint be used for actually searching users/groups, or just for _listing_ all users/groups that have been granted access?
:::

- Search users/groups
    - ``GET /api/records/{id}/access/(users|groups)``
- Remove a user/group
    - ``DELETE /api/records/{id}/access/(users|groups)/{entity-id}``
- Update a user/group
    - ``PUT /api/records/{id}/access/(users|groups)/{entity-id}``
        - access
- Bulk remove/update
    - TODO: Check members API
- Add users/groups
    - ``POST /api/records/{id}/access/(users|groups)``
        - access
        - notify
        - message  
- Update access link
    - ``PATCH /api/records/{id}/access/links/{link-id}``
        - Support editing the name of the link
- Get record access settings
    - ``GET /api/records/{id}/access``
        - allow requests
        - condition
        - default access
        - link expiration
        - send to: owner, community
- Explain access
    - ``GET /api/records/{id}/access?explain=1``
- Update record access settings
    - ``PUT /api/records/{id}/access``

- Request access
    - ``POST /api/records/{id}/access/requests``
        - requestor: user or email entity
        - message
        - consent
- Confirm email (non API)
    - ``GET /records/{id}/access/requests/{request-id}/{token}``

## Data model

- Access links
    - Support different types of links (user created, and request created with more fields)
    - Support clean up of expired links after say 1 year.
- Access request type:
    - Grant permission to a request given a signed token.
    - Support for updating the payload of the request while in submitted state.
    - For emails: request is in created state until email confirmed
    - For users: requests is submitted immediately
    - Request timeline should have a link to view the record so we need a way to have event data to render it.
    - Is it possible to have multiple receivers