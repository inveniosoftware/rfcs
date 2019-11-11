- Start Date: 2019-10-01
- RFC PR: [#7](https://github.com/inveniosoftware/rfcs/pull/7)
- Authors: Pablo Panero

# Access Control

## Summary

Overall design for management of access control and permissions over resources in Invenio RDM. Due to the need of more fine-grained permission policies, the chosen model is ABAC (Attribute Based Access Control).

### Why is this needed

The current implementation (November 2019) requires the re-implementation of full permission factories in order to customize access control. Moreover, it is not clear across implementations if the factories should return a set of needs or simply over-write the `can` function. In the latter and most common does not leverage the whole power of *Flask-Principal*. Therefore, a change of paradigm is needed in order to make access control customizations simple and consistent.

## Motivation

### Administrator:
- As an administrator, I want to have both public and restricted records, so that not all the records are publicly visible.
- As an administrator, I want to have a flexible and easy way of defining access control policies, so that I do not need to code them.
- As an administrator, I want to have IP-based access-control, so that I can comply with publisher rules.
- As an administrator, I want to easily define roles like *academic staff* based on my IdP (Identity Provider), so that I can implement business rules.
- As an administrator, I want to restrict access to certain records based on user features (e.g. registered user, member of a specific community, member of academia), so that I can implement my institution's policies.
- As an administrator, I want to give access my repositories for other institutions and their IDMS (?? Institutional Data Management System).

### User:
- As a user, I want to restrict access to files in a public record, so that I can safely store and cite my dataset without giving access to files.
- As a user, I want to restrict files and/or metadata during an embargo period, so that I can comply with publisher rules.
- As a user, I want to be able to obtain a shareable link, so that my colleagues can access the files of my **restricted record**.
- As a user, I want to share a preview of a **deposit** record prior to publishing it, so that my colleagues can validate it.
- As a user, I want to grant access to my restricted records by listing specific users and/or groups, so that I can easily revoke the access if needed.
- As a user, I want to access my data even after I have left my university.

## Terminology

- **Resources**: The term resource in the system represents anything you want to protect. Resources can both be at the object-level (e.g. access to a record) or of more general nature (e.g access to the administration interface).
- **Actions**: Resources can be protected by requiring the ability to perform one or more actions.
- **Subjects**: Access to resources can be granted to *subjects*, which include users, roles and system roles. System roles are roles that are programmatically assigned to users. For example:
    - Authenticated user: any logged in user.
    - Any user: both authenticated and anonymous users.
    - Campus user: Assigned to users based on their IP address being in a specific range.


## Detailed design

Invenio implements a Role-Based Access Control system (RBAC), that can be used to protect and grant access to resources.

### Records as resources

The most complex access rights use cases in Invenio RDM are related to records. Note that the following access control can be different between records and deposit records. In addition, this access control model is not tied to Invenio RDM, so it can be applied to any data model in Invenio.

**Record actions**

All records have the following basic CRUD actions:
- Create: Ability to create new records.
- Read: Ability to read a record (including during searches).
- Update: Ability to update a record metadata.
- Delete: Ability to delete a record.
- List: Ability to perform searches (note, for a record to appear in the search result it is subject to the above mentioned read action).

Records linked with files (possible for all records) have the additional following actions.
- Create files: Ability to create files associated with a record. 
- Read files: Ability to read files associated with a record.
- Update files: Ability to update files associated with a record.
- Delete files: Ability to delete files associated with a record.
- Administrate files: Ability to perform all possible actions on files associated with a record, including reading previous versions of files.

Deposit records have the additional following actions:
- Publish: Ability to publish a new record.
- Edit: Ability to edit an unpublished record. This differs from the record update action.
- New version: Ability to create a new version of an existing record.

It is possible to extend this model with more actions for each data model as needed.

**Requiring permissions**

A record must store permissions inside the metadata. This can be as simple as a list of owners, or very complex based on custom rules (e.g. books can only be curated by books curators). This rules are translated into `generators`.

**Generators**

A *generator* implements a rule to allow or deny the access to perform an action.  

The generator would return a set of *needs* and *excludes* for each action. The *needs* are the constraints the user has to meet in order to be allowed to perform an action (e.g. be the owner of the record). The *excludes* specify a set of blacklisting constraints, which have precedence over the *needs*. 

For example, the *record owners* generator would look like:

``` python
Document:
    _owners: [1, 2, 3]

RecordOwner:
    needs:
        return Document._owners
    excludes:
        pass
```

Since the record owner does not set any specific blacklist (*excludes*), the generator would result in an access granted for users with id 1, 2 or 3.

In addition, generators return a set of *query filters*, which are used only when searching. The set of rules returned there are in the form of an Elasticsearch Query object. The `RecordOwner` generator would therefore look like:

``` python
RecordOwner:
    needs:
        return Document._owners
    excludes:
        pass
    query_filter:
        return Query(owner=user.id)
```

As a result, it can be seen that the *query filter* inverts the paradigm with respect to the *needs*/*excludes*. Instead of receiving a record and returning its owners, it takes the id of the user performing the action and returns a query object. This is due to the nature of the *list* (search) operation. The check must not be *Can user A access record R, for each one of the records in the system*. This would not be scalable. Therefore, the check must be *which documents can be accessed by the user A*.

Invenio RDM comes with a set of predefined generators:
- Any user: Any user can perform the action.
- Any user if public: Any user can perform the action if the record is public.
- Record owners: Owners of the record can access perform the action.
- Curators: Curators can perform an action.
- Super Curators: ??


**Permission Policies**

A *permission policy* defines the actions and who can or cannot perform them as generators. A record policy would be set as:

``` python
RecordPermissionPolicy:
    can_create: []
    can_read: []
    can_update: []
    can_delete: []
    can_list: []
```

The policy would return a set of *needs* and *excludes* for each action. The *needs* and *excludes* of a policy is the union of all those of the specified generators.

For the following user story:

```
As an admin I want to grant read access to the owners of a document, except if the owner is part of team A.
```

The following policy applies the `RecordOwner` and `TeamMembership` generators to the record read action. The `RecordOwner` generator, allows the owners of the document (defined right above it, with ids 1, 2 and 3) to access the document. On the other hand, `TeamMembership` blacklists users belonging to Team A.

``` python
Document:
    _owners: [1, 2, 3]

RecordPermissionPolicy:
    can_read: [RecordOwner, TeamMembership]

RecordOwner:
    needs:
        return Document._owners

TeamMembership:
    needs:
        pass
    excludes:
        return [Team A]
```

Assuming the following two users:

``` python
UserOne:
    id: 1
    team: B

UserTwo:
    id: 2
    team: A
```

*UserOne* would be able to read the document since it has id `1`. However, *UserTwo* would not. Eventhough, it has id `2`, it belongs to `Team A` which has been blacklisted.

## How we teach this

This is a hard to teach concept because it involves a breaking change on the paradigm as from what it is today. However, it makes it much more user friendly and involves less work to implement custom access control rules. The best way to teach this would be to edit the corresponding section in the Invenio Tutorials (assuming the model is adopted by the whole Invenio Framework). In addition, create tutorial in the general RDM documentation.

## Drawbacks

It might increase the learning curve for current power users. However, for new adopters it will make the access control handling much easier.

## Alternatives

Keep using the current model where every instance has to implement their own using flask-principal.

## Unresolved questions

- Update and Delete actions over files, are iherited from the associated record. Or are per file?
- Deposit Edit vs Record update?
- Is *query filters* a good name?
- Mention Elasticsearch Query object. Which is a implementation detail.
- Is the example with Team A being blacklisted clear? I was thinking for example of ATLAS vs. CMS not allowing each other to access their research (for mission purposes.)
- Curator vs Supercurator? Where is the line? Community vs Global curator?
- Are the commented examples good for here? or should they go to the module/RDM docs. (See below)
